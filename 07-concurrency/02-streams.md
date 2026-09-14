---
id: "7.2"
title: "Streams"
difficulty: "advanced"
section: "07-concurrency"
prerequisites:
  - "Event loop i Futures (async/await)"
  - "Generatory (sync* i async*)"
---

# 7.2 Streams

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Event loop i Futures (async/await)](../07-concurrency/01-async-await.md), [Generatory (sync* i async*)](../04-functions/04-generators.md)
- **Cele nauki:**
  1. Rozumieć naturę `Stream` jako asynchronicznej sekwencji zdarzeń oraz różnicę między strumieniami single-subscription a broadcast
  2. Tworzyć własne strumienie za pomocą `StreamController`, generatorów `async*` oraz konstruktorów fabrycznych (`Stream.fromIterable`, `Stream.periodic`, `Stream.eventTransformed`)
  3. Stosować transformacje strumieni (`map`, `where`, `expand`, `take`, `skip`, `distinct`, `asyncMap`, `asyncExpand`) oraz obsługiwać błędy w strumieniach
  4. Implementować własne transformery strumieni za pomocą `StreamTransformer`, w tym transformery stanowe (stateful)
  5. Zarządzać cyklem życia subskrypcji (`listen`, `pause`, `resume`, `cancel`) i świadomie wybierać między `Stream` a `Future`

---

## Czym jest Stream?

`Stream<T>` reprezentuje **asynchroniczną sekwencję danych** — serię zdarzeń dostarczanych w czasie. Jeśli `Future<T>` to obietnica dostarczenia **jednej** wartości w przyszłości, to `Stream<T>` to obietnica dostarczenia **zera, jednej lub wielu** wartości (oraz opcjonalnie błędów) w przyszłości.

Strumień emituje trzy rodzaje zdarzeń:

- **zdarzenia danych** (data events) — kolejne wartości typu `T`,
- **zdarzenia błędu** (error events) — błędy, które nie przerywają automatycznie strumienia,
- **zdarzenie zakończenia** (done event) — sygnał, że strumień się zakończył i nie będzie więcej zdarzeń.

Poniższy przykład pokazuje najprostszy sposób konsumpcji strumienia za pomocą pętli `await for`, która pobiera kolejne zdarzenia danych aż do zakończenia strumienia.

```dart
void main() async {
  // Stream.fromIterable tworzy strumień emitujący kolejne elementy kolekcji
  final strumien = Stream<int>.fromIterable([10, 20, 30]);

  // await for konsumuje strumień element po elemencie
  await for (final wartosc in strumien) {
    print('Otrzymano: $wartosc');
  }
  print('Strumień zakończony');
}
// Oczekiwane wyjście:
// Otrzymano: 10
// Otrzymano: 20
// Otrzymano: 30
// Strumień zakończony
```

Alternatywnie strumień można konsumować za pomocą metody `listen`, która rejestruje callbacki dla poszczególnych rodzajów zdarzeń.

```dart
void main() {
  final strumien = Stream<String>.fromIterable(['a', 'b', 'c']);

  // listen rejestruje callbacki dla danych, błędów i zakończenia
  strumien.listen(
    (dane) => print('Dane: $dane'),      // wywoływane dla każdego zdarzenia danych
    onError: (e) => print('Błąd: $e'),   // wywoływane dla zdarzeń błędu
    onDone: () => print('Zakończono'),   // wywoływane przy zdarzeniu done
  );
}
// Oczekiwane wyjście:
// Dane: a
// Dane: b
// Dane: c
// Zakończono
```

---

## Single-subscription vs broadcast

Dart rozróżnia dwa rodzaje strumieni, które fundamentalnie różnią się semantyką subskrypcji.

**Strumień single-subscription** (jednokrotnej subskrypcji) to domyślny rodzaj strumienia. Można go zasubskrybować **tylko raz**. Zdarzenia są buforowane do momentu pojawienia się słuchacza, a każde zdarzenie trafia do dokładnie jednego odbiorcy. Takie strumienie reprezentują zwykle pojedynczą sekwencję danych, np. zawartość pliku odczytywaną kawałek po kawałku.

**Strumień broadcast** (rozgłoszeniowy) może mieć **wielu jednoczesnych słuchaczy**, a każde zdarzenie jest dostarczane do wszystkich aktywnych subskrybentów. Strumień broadcast nie buforuje zdarzeń dla przyszłych słuchaczy — nowy słuchacz otrzymuje tylko zdarzenia wyemitowane po jego zasubskrybowaniu. Takie strumienie reprezentują zwykle zdarzenia niezależne od odbiorcy, np. kliknięcia myszy.

Poniższy przykład demonstruje ograniczenie strumienia single-subscription: próba drugiej subskrypcji kończy się wyjątkiem.

```dart
import 'dart:async';

void main() {
  // Uwaga: Stream.fromIterable jest zaimplementowany przez Stream.multi
  // i DOPUSZCZA wielu słuchaczy (każdy dostaje własne, niezależne odtworzenie
  // elementów) — nie nadaje się więc do demonstracji klasycznego rzucania
  // wyjątku. StreamController.stream jest prawdziwym strumieniem
  // single-subscription.
  final controller = StreamController<int>();
  final strumien = controller.stream;

  strumien.listen((v) => print('Słuchacz 1: $v')); // pierwsza subskrypcja OK

  try {
    // Druga subskrypcja strumienia single-subscription rzuca wyjątek
    strumien.listen((v) => print('Słuchacz 2: $v'));
  } catch (e) {
    print('Błąd: ${e.runtimeType}');
  }

  controller.add(1);
  controller.add(2);
  controller.add(3);
  controller.close();
}
// Oczekiwane wyjście:
// Błąd: StateError
// Słuchacz 1: 1
// Słuchacz 2 nigdy nie zostaje zarejestrowany — subskrypcja rzuciła wyjątek
// Słuchacz 1: 2
// Słuchacz 1: 3
```

Następny przykład pokazuje strumień broadcast z dwoma niezależnymi słuchaczami, którzy otrzymują te same zdarzenia. Strumień single-subscription można zamienić na broadcast metodą `asBroadcastStream()`.

```dart
void main() {
  // asBroadcastStream tworzy strumień rozgłoszeniowy z wieloma słuchaczami
  final broadcast = Stream<int>.fromIterable([1, 2, 3]).asBroadcastStream();

  broadcast.listen((v) => print('A otrzymał: $v')); // pierwszy słuchacz
  broadcast.listen((v) => print('B otrzymał: $v')); // drugi słuchacz — OK dla broadcast
}
// Oczekiwane wyjście:
// A otrzymał: 1
// B otrzymał: 1
// A otrzymał: 2
// B otrzymał: 2
// A otrzymał: 3
// B otrzymał: 3
```

---

## StreamController — ręczne tworzenie strumieni

`StreamController<T>` to najbardziej elastyczny sposób tworzenia strumieni. Kontroler udostępnia obiekt `stream` (do subskrypcji przez konsumentów) oraz obiekt `sink` (do dodawania zdarzeń przez producenta). Zdarzenia dodaje się metodami `add` (dane), `addError` (błąd) i `close` (zakończenie).

Poniższy przykład demonstruje ręczne sterowanie strumieniem: producent emituje dane, błąd i sygnał zakończenia, a konsument reaguje na każde ze zdarzeń.

```dart
import 'dart:async';

void main() async {
  // StreamController tworzy strumień single-subscription, którym sterujemy ręcznie
  final controller = StreamController<int>();

  controller.stream.listen(
    (dane) => print('Dane: $dane'),
    onError: (e) => print('Błąd: $e'),
    onDone: () => print('Zakończono'),
  );

  controller.add(1);                  // emituje zdarzenie danych
  controller.add(2);
  controller.addError('coś poszło nie tak'); // emituje zdarzenie błędu
  controller.add(3);
  await controller.close();           // emituje zdarzenie done i zamyka strumień
}
// Oczekiwane wyjście:
// Dane: 1
// Dane: 2
// Błąd: coś poszło nie tak
// Dane: 3
// Zakończono
```

Aby utworzyć strumień broadcast, używamy konstruktora `StreamController.broadcast()`. Kontroler broadcast pozwala rejestrować wielu słuchaczy i udostępnia callbacki `onListen`/`onCancel` do reagowania na pojawianie się i znikanie subskrybentów.

```dart
import 'dart:async';

void main() {
  // Kontroler broadcast pozwala na wielu jednoczesnych słuchaczy
  final controller = StreamController<String>.broadcast();

  controller.stream.listen((v) => print('Słuchacz 1: $v'));
  controller.stream.listen((v) => print('Słuchacz 2: $v'));

  controller.add('cześć');
  controller.close();
}
// Oczekiwane wyjście:
// Słuchacz 1: cześć
// Słuchacz 2: cześć
```

---

## Konstruktory fabryczne strumieni

Oprócz `StreamController` i generatorów `async*`, klasa `Stream` udostępnia kilka gotowych konstruktorów fabrycznych.

`Stream.fromIterable` tworzy strumień emitujący kolejne elementy istniejącej kolekcji. `Stream.value` emituje pojedynczą wartość, a `Stream.error` pojedynczy błąd.

```dart
void main() async {
  // fromIterable emituje kolejne elementy kolekcji jako zdarzenia danych
  final ze_zbioru = Stream<int>.fromIterable([1, 2, 3]);
  print(await ze_zbioru.toList());

  // Stream.value emituje dokładnie jedną wartość i kończy się
  final jedna = Stream<String>.value('jedyna');
  print(await jedna.toList());
}
// Oczekiwane wyjście:
// [1, 2, 3]
// [jedyna]
```

`Stream.periodic` tworzy strumień broadcast-podobny emitujący wartości w regularnych odstępach czasu. Funkcja generująca otrzymuje kolejny numer zdarzenia (licząc od 0).

```dart
void main() async {
  // periodic emituje wartość co zadany interwał; funkcja mapuje numer zdarzenia
  final zegar = Stream<int>.periodic(
    Duration(milliseconds: 100),
    (licznik) => licznik * licznik, // kwadrat numeru zdarzenia
  );

  // take(4) ogranicza nieskończony strumień do 4 elementów
  await for (final v in zegar.take(4)) {
    print(v);
  }
}
// Oczekiwane wyjście:
// 0
// 1
// 4
// 9
```

`Stream.eventTransformed` pozwala opakować istniejący strumień własnym `EventSink`, przekształcając zdarzenia źródłowe w inne zdarzenia. Poniższy przykład podwaja każde zdarzenie danych i przepuszcza pozostałe zdarzenia bez zmian.

```dart
import 'dart:async';

// Sink przekształcający zdarzenia: każde dane emituje dwukrotnie
class PodwajajacySink implements EventSink<int> {
  final EventSink<int> _wyjscie;
  PodwajajacySink(this._wyjscie);

  @override
  void add(int event) {
    _wyjscie.add(event);      // przekazuje oryginalne zdarzenie
    _wyjscie.add(event);      // i jego duplikat
  }

  @override
  void addError(Object e, [StackTrace? s]) => _wyjscie.addError(e, s);

  @override
  void close() => _wyjscie.close();
}

void main() async {
  final zrodlo = Stream<int>.fromIterable([1, 2, 3]);

  // eventTransformed opakowuje strumień źródłowy naszym sinkiem
  final przeksztalcony = Stream<int>.eventTransformed(
    zrodlo,
    (sink) => PodwajajacySink(sink),
  );

  print(await przeksztalcony.toList());
}
// Oczekiwane wyjście:
// [1, 1, 2, 2, 3, 3]
```

---

## Transformacje strumieni

Strumienie udostępniają bogaty zestaw metod transformujących, które zwracają nowy strumień. Transformacje są **leniwe** — działają dopiero gdy strumień wynikowy zostanie zasubskrybowany.

### `map` — przekształcanie elementów

`map` przekształca każde zdarzenie danych za pomocą synchronicznej funkcji.

```dart
void main() async {
  final liczby = Stream<int>.fromIterable([1, 2, 3, 4]);

  // map przekształca każdy element synchroniczną funkcją
  final kwadraty = liczby.map((n) => n * n);

  print(await kwadraty.toList());
}
// Oczekiwane wyjście:
// [1, 4, 9, 16]
```

### `where` — filtrowanie elementów

`where` przepuszcza tylko te zdarzenia danych, które spełniają predykat.

```dart
void main() async {
  final liczby = Stream<int>.fromIterable([1, 2, 3, 4, 5, 6]);

  // where przepuszcza tylko elementy spełniające warunek
  final parzyste = liczby.where((n) => n.isEven);

  print(await parzyste.toList());
}
// Oczekiwane wyjście:
// [2, 4, 6]
```

### `expand` — rozwijanie elementu w wiele

`expand` mapuje każdy element na `Iterable` i emituje wszystkie jego elementy — jest to odpowiednik `flatMap` dla kolekcji synchronicznych.

```dart
void main() async {
  final liczby = Stream<int>.fromIterable([1, 2, 3]);

  // expand mapuje każdy element na Iterable i spłaszcza wynik
  final rozwiniete = liczby.expand((n) => [n, n * 10]);

  print(await rozwiniete.toList());
}
// Oczekiwane wyjście:
// [1, 10, 2, 20, 3, 30]
```

### `take` — pobranie pierwszych N elementów

`take` przepuszcza tylko pierwsze `n` zdarzeń danych, po czym kończy strumień.

```dart
void main() async {
  final liczby = Stream<int>.fromIterable([1, 2, 3, 4, 5]);

  // take pobiera pierwsze n elementów i kończy strumień
  final pierwsze = liczby.take(3);

  print(await pierwsze.toList());
}
// Oczekiwane wyjście:
// [1, 2, 3]
```

### `skip` — pominięcie pierwszych N elementów

`skip` pomija pierwsze `n` zdarzeń danych i przepuszcza pozostałe.

```dart
void main() async {
  final liczby = Stream<int>.fromIterable([1, 2, 3, 4, 5]);

  // skip pomija pierwsze n elementów
  final reszta = liczby.skip(2);

  print(await reszta.toList());
}
// Oczekiwane wyjście:
// [3, 4, 5]
```

### `distinct` — usuwanie kolejnych duplikatów

`distinct` pomija zdarzenia danych, które są równe **bezpośrednio poprzedzającemu** zdarzeniu (nie usuwa wszystkich duplikatów globalnie).

```dart
void main() async {
  final dane = Stream<int>.fromIterable([1, 1, 2, 2, 2, 3, 1]);

  // distinct pomija elementy równe poprzedniemu (kolejne duplikaty)
  final unikalne = dane.distinct();

  print(await unikalne.toList());
}
// Oczekiwane wyjście:
// [1, 2, 3, 1]
```

### `asyncMap` — asynchroniczne przekształcanie

`asyncMap` działa jak `map`, ale funkcja przekształcająca zwraca `Future`. Strumień czeka na zakończenie każdego `Future` przed przetworzeniem kolejnego elementu, zachowując kolejność.

```dart
// Symulacja asynchronicznego wzbogacania danych
Future<String> pobierzNazwe(int id) async {
  await Future.delayed(Duration(milliseconds: 50));
  return 'Użytkownik#$id';
}

void main() async {
  final identyfikatory = Stream<int>.fromIterable([1, 2, 3]);

  // asyncMap czeka na Future dla każdego elementu, zachowując kolejność
  final nazwy = identyfikatory.asyncMap(pobierzNazwe);

  print(await nazwy.toList());
}
// Oczekiwane wyjście:
// [Użytkownik#1, Użytkownik#2, Użytkownik#3]
```

### `asyncExpand` — asynchroniczne rozwijanie w strumień

`asyncExpand` mapuje każdy element na **strumień** i emituje wszystkie jego zdarzenia sekwencyjnie — asynchroniczny odpowiednik `expand`.

```dart
// Generator zwracający strumień dla pojedynczego elementu
Stream<String> rozbij(int n) async* {
  yield 'start-$n';
  await Future.delayed(Duration(milliseconds: 20));
  yield 'koniec-$n';
}

void main() async {
  final liczby = Stream<int>.fromIterable([1, 2]);

  // asyncExpand mapuje każdy element na strumień i spłaszcza wyniki sekwencyjnie
  final wynik = liczby.asyncExpand(rozbij);

  print(await wynik.toList());
}
// Oczekiwane wyjście:
// [start-1, koniec-1, start-2, koniec-2]
```

---

## Obsługa błędów w strumieniach

Strumienie mogą przenosić zdarzenia błędu obok zdarzeń danych. Domyślnie błąd nie kończy strumienia single-subscription — po błędzie mogą pojawić się kolejne dane. Błędy obsługujemy przez callback `onError` w `listen`, przez `try/catch` wokół `await for`, lub przez metodę `handleError`.

Poniższy przykład pokazuje obsługę błędu za pomocą `try/catch` wokół `await for`. Uwaga: błąd w `await for` **przerywa** pętlę.

```dart
import 'dart:async';

void main() async {
  final controller = StreamController<int>();

  // Producent emituje dane i błąd
  controller.add(1);
  controller.addError(Exception('awaria'));
  controller.add(2);
  controller.close();

  try {
    await for (final v in controller.stream) {
      print('Dane: $v');
    }
  } catch (e) {
    print('Przechwycono w await for: $e'); // await for przerywa się na błędzie
  }
}
// Oczekiwane wyjście:
// Dane: 1
// Przechwycono w await for: Exception: awaria
```

Metoda `handleError` przechwytuje błędy w łańcuchu transformacji, pozwalając strumieniowi kontynuować. Poniższy przykład używa `handleError`, aby zalogować błąd i kontynuować odbieranie kolejnych danych.

```dart
import 'dart:async';

void main() async {
  final controller = StreamController<int>();

  controller.add(1);
  controller.addError(Exception('problem'));
  controller.add(2);
  controller.close();

  // handleError przechwytuje błędy, ale nie przerywa strumienia
  final bezpieczny = controller.stream.handleError((e) {
    print('Zignorowano błąd: $e');
  });

  await for (final v in bezpieczny) {
    print('Dane: $v'); // dane po błędzie nadal docierają
  }
}
// Oczekiwane wyjście:
// Dane: 1
// Zignorowano błąd: Exception: problem
// Dane: 2
```

---

## StreamTransformer i transformery stanowe

`StreamTransformer<S, T>` to reużywalny obiekt przekształcający strumień typu `S` w strumień typu `T`. Transformery stosuje się metodą `transform`. Najprostszy sposób tworzenia transformera to `StreamTransformer.fromHandlers`, gdzie definiujemy handlery dla danych, błędów i zakończenia.

Poniższy przykład tworzy **transformer stanowy** — utrzymuje bieżącą sumę i emituje sumę skumulowaną (running total). Stan (`suma`) jest przechowywany w domknięciu, dzięki czemu każde zdarzenie może zależeć od poprzednich.

```dart
import 'dart:async';

// Transformer stanowy: utrzymuje sumę skumulowaną między zdarzeniami
StreamTransformer<int, int> sumaSkumulowana() {
  var suma = 0; // stan przechowywany w domknięciu

  return StreamTransformer<int, int>.fromHandlers(
    handleData: (wartosc, sink) {
      suma += wartosc;      // aktualizuje stan
      sink.add(suma);       // emituje bieżącą sumę
    },
    handleError: (e, s, sink) => sink.addError(e, s), // przekazuje błędy dalej
    handleDone: (sink) => sink.close(),               // zamyka strumień wyjściowy
  );
}

void main() async {
  final liczby = Stream<int>.fromIterable([1, 2, 3, 4]);

  // transform stosuje transformer do strumienia
  final skumulowane = liczby.transform(sumaSkumulowana());

  print(await skumulowane.toList());
}
// Oczekiwane wyjście:
// [1, 3, 6, 10]
```

Kolejny przykład to bardziej rozbudowany transformer stanowy, który grupuje przychodzące elementy w partie (batche) o zadanym rozmiarze — użyteczny wzorzec przy przetwarzaniu wsadowym.

```dart
import 'dart:async';

// Transformer grupujący elementy w partie o rozmiarze [rozmiar]
StreamTransformer<T, List<T>> wPartiach<T>(int rozmiar) {
  var bufor = <T>[]; // stan: bieżąca, niepełna partia

  return StreamTransformer<T, List<T>>.fromHandlers(
    handleData: (element, sink) {
      bufor.add(element);
      if (bufor.length == rozmiar) {
        sink.add(List<T>.from(bufor)); // emituje pełną partię (kopia)
        bufor = <T>[];                 // resetuje bufor
      }
    },
    handleDone: (sink) {
      if (bufor.isNotEmpty) sink.add(bufor); // emituje ostatnią, niepełną partię
      sink.close();
    },
  );
}

void main() async {
  final dane = Stream<int>.fromIterable([1, 2, 3, 4, 5]);

  final partie = dane.transform(wPartiach<int>(2));

  print(await partie.toList());
}
// Oczekiwane wyjście:
// [[1, 2], [3, 4], [5]]
```

---

## Cykl życia subskrypcji: listen, pause, resume, cancel

Metoda `listen` zwraca obiekt `StreamSubscription`, który reprezentuje aktywną subskrypcję i pozwala nią sterować:

- `pause()` — wstrzymuje dostarczanie zdarzeń (strumień buforuje je do wznowienia),
- `resume()` — wznawia dostarczanie zdarzeń,
- `cancel()` — trwale anuluje subskrypcję (nie da się jej wznowić).

Poniższy przykład demonstruje pełny cykl życia subskrypcji na strumieniu periodycznym, z adnotacjami które zdarzenia docierają na każdym etapie.

```dart
import 'dart:async';

void main() async {
  // Strumień emitujący kolejne liczby co 100 ms
  final zrodlo = Stream<int>.periodic(
    Duration(milliseconds: 100),
    (i) => i,
  );

  late StreamSubscription<int> sub;

  // Etap 1: listen — rozpoczyna odbieranie zdarzeń
  sub = zrodlo.listen((v) => print('Odebrano: $v'));

  // Odbierane: 0, 1 (przez pierwsze ~250 ms)
  await Future.delayed(Duration(milliseconds: 250));

  // Etap 2: pause — wstrzymuje dostarczanie; zdarzenia 2, 3, 4... nie docierają
  print('--- pause ---');
  sub.pause();
  await Future.delayed(Duration(milliseconds: 300));

  // Etap 3: resume — wznawia dostarczanie od bieżącego momentu
  print('--- resume ---');
  sub.resume();

  // Odbierane kolejne wartości po wznowieniu
  await Future.delayed(Duration(milliseconds: 250));

  // Etap 4: cancel — trwale kończy subskrypcję; więcej zdarzeń nie będzie
  print('--- cancel ---');
  await sub.cancel();

  await Future.delayed(Duration(milliseconds: 200));
  print('Koniec programu');
}
// Oczekiwane wyjście (wartości zależne od czasu, kolejność zdarzeń stała):
// Odebrano: 0
// Odebrano: 1
// --- pause ---
// --- resume ---
// Odebrano: 2
// Odebrano: 3
// --- cancel ---
// Koniec programu
```

`Stream.periodic` reaguje na `pause`/`resume` swojego subskrybenta: wewnętrzny `Timer` jest **zatrzymywany** na czas pauzy i wznawiany z uwzględnieniem czasu, jaki upłynął od ostatniego zdarzenia przed pauzą. Dlatego żadne "tyknięcia" nie giną ani się nie kumulują — numeracja zdarzeń kontynuuje się bez przeskoków (`0, 1`, pauza, `2, 3, ...`). Kluczowa obserwacja: między `pause` a `resume` **żadne zdarzenie nie dociera do callbacku** (generowanie zdarzeń jest wstrzymane u źródła), a po `cancel` subskrypcja jest martwa — dalsze zdarzenia są ignorowane.

---

## Porównanie: Stream vs Future

`Future` i `Stream` to dwa filary programowania asynchronicznego w Dart, ale służą do różnych celów.

| Wymiar | `Future<T>` | `Stream<T>` |
|--------|-------------|-------------|
| Liczba wartości | Dokładnie jedna wartość (lub jeden błąd) | Zero, jedna lub wiele wartości w czasie |
| Ewaluacja | Zwykle eager — obliczenie startuje od razu przy utworzeniu | Leniwa dla single-subscription — kod producenta rusza dopiero przy `listen` |
| Konsumpcja | `await` lub `.then()` | `await for`, `.listen()` lub transformacje |
| Anulowanie | Brak wbudowanego anulowania | `StreamSubscription.cancel()` przerywa odbieranie |
| Pamięć | Przechowuje pojedynczy wynik | Może przetwarzać dane strumieniowo, bez trzymania całości w pamięci |
| Typowe zastosowanie | Pojedyncza operacja I/O (zapytanie HTTP, odczyt pliku) | Sekwencja zdarzeń (kliknięcia, wiersze pliku, wiadomości WebSocket) |

Kluczowa różnica dotycząca ewaluacji: strumień single-subscription jest **leniwy** — kod producenta (np. ciało generatora `async*`) uruchamia się dopiero po wywołaniu `listen` lub `await for`.

```dart
// Generator async* — jego ciało nie wykona się, dopóki nikt nie zasubskrybuje
Stream<int> leniwyStrumien() async* {
  print('Producent wystartował'); // wypisze się dopiero przy subskrypcji
  yield 1;
  yield 2;
}

void main() async {
  final s = leniwyStrumien(); // utworzenie strumienia NIE uruchamia producenta
  print('Strumień utworzony, producent jeszcze śpi');

  await for (final v in s) {   // dopiero teraz producent rusza
    print('Wartość: $v');
  }
}
// Oczekiwane wyjście:
// Strumień utworzony, producent jeszcze śpi
// Producent wystartował
// Wartość: 1
// Wartość: 2
```

---

## Ćwiczenie 1: Potok transformacji strumienia

### Opis problemu

Zaimplementuj funkcję `przetworzTransakcje(Stream<int> kwoty)`, która buduje **potok co najmniej 3 transformacji** na strumieniu kwot transakcji (w groszach). Potok ma:

1. odrzucić transakcje o wartości mniejszej lub równej 0 (`where`),
2. przeliczyć grosze na złotówki jako `double` (`map`),
3. pominąć pierwszą transakcję (uznajemy ją za testową) (`skip`),
4. zwrócić strumień wartości w złotówkach.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `[1050, -20, 300, 0, 4999]` | `[3.0, 49.99]` |
| `[500, 500, 500]` | `[5.0, 5.0]` |

### Wskazówki

1. Łańcuchuj transformacje: `strumien.where(...).map(...).skip(...)`
2. Kolejność ma znaczenie — `skip` powinno działać na już przefiltrowanych i zmapowanych wartościach
3. Grosze na złotówki: podziel przez 100 (`kwota / 100`)
4. Aby zebrać wynik do listy w celu weryfikacji, użyj `await strumien.toList()`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
/// Buduje potok 3 transformacji: filtrowanie → mapowanie → pominięcie.
Stream<double> przetworzTransakcje(Stream<int> kwoty) {
  return kwoty
      .where((grosze) => grosze > 0)   // odrzuca niepoprawne kwoty
      .map((grosze) => grosze / 100)   // grosze -> złotówki
      .skip(1);                        // pomija pierwszą (testową) transakcję
}

void main() async {
  final wynik1 =
      await przetworzTransakcje(Stream.fromIterable([1050, -20, 300, 0, 4999]))
          .toList();
  print(wynik1);

  final wynik2 =
      await przetworzTransakcje(Stream.fromIterable([500, 500, 500])).toList();
  print(wynik2);
}
// Oczekiwane wyjście:
// [3.0, 49.99]
// [5.0, 5.0]
```

</details>

---

## Ćwiczenie 2: Debounce z użyciem Timer i StreamController

### Opis problemu

Zaimplementuj funkcję `debounce<T>(Stream<T> zrodlo, Duration czas)`, która zwraca nowy strumień emitujący **tylko ostatnią wartość** z serii szybko następujących po sobie zdarzeń — wartość jest emitowana dopiero gdy przez `czas` nie pojawi się nowe zdarzenie. To klasyczny mechanizm używany np. przy polach wyszukiwania, aby nie reagować na każde naciśnięcie klawisza.

Użyj `StreamController` do produkcji strumienia wyjściowego oraz `Timer` do odmierzania okresu ciszy.

**Poziom trudności:** advanced

### Przykłady wejścia/wyjścia

| Wejście (wartości co 50 ms, debounce 120 ms) | Oczekiwane wyjście |
|---------|-------------------|
| `a, b, c` (szybko), pauza, `d` | `c, d` |
| `x` (pojedyncza wartość) | `x` |

### Wskazówki

1. Przy każdym nowym zdarzeniu anuluj poprzedni `Timer` (`timer?.cancel()`) i uruchom nowy na `czas`
2. Dopiero gdy `Timer` wypali (minął okres ciszy), dodaj ostatnią wartość do `StreamController`
3. Gdy strumień źródłowy się zakończy (`onDone`), wyemituj oczekującą wartość (jeśli jest) i zamknij kontroler
4. Pamiętaj o obsłudze `onError`, aby przekazać błędy do strumienia wyjściowego

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:async';

/// Zwraca strumień emitujący tylko ostatnią wartość po okresie ciszy [czas].
Stream<T> debounce<T>(Stream<T> zrodlo, Duration czas) {
  final controller = StreamController<T>();
  Timer? timer;
  T? ostatnia;
  var maOczekujaca = false;

  controller.onListen = () {
    zrodlo.listen(
      (wartosc) {
        ostatnia = wartosc;
        maOczekujaca = true;
        timer?.cancel();                    // anuluje poprzedni odmierzany czas
        timer = Timer(czas, () {
          controller.add(ostatnia as T);    // emituje po okresie ciszy
          maOczekujaca = false;
        });
      },
      onError: controller.addError,         // przekazuje błędy dalej
      onDone: () {
        timer?.cancel();
        if (maOczekujaca) controller.add(ostatnia as T); // domyka oczekującą wartość
        controller.close();
      },
    );
  };

  return controller.stream;
}

void main() async {
  // Producent: a,b,c szybko (co 40ms), potem pauza, potem d
  Stream<String> zrodlo() async* {
    yield 'a';
    await Future.delayed(Duration(milliseconds: 40));
    yield 'b';
    await Future.delayed(Duration(milliseconds: 40));
    yield 'c';
    await Future.delayed(Duration(milliseconds: 200)); // pauza > debounce
    yield 'd';
  }

  final wynik = await debounce(zrodlo(), Duration(milliseconds: 120)).toList();
  print(wynik);
}
// Oczekiwane wyjście:
// [c, d]
```

</details>

---

## Ćwiczenie 3: Odporna obsługa błędów strumienia

### Opis problemu

Masz strumień, który przy przetwarzaniu niektórych elementów rzuca wyjątki. Zaimplementuj funkcję `bezpiecznePrzetwarzanie(Stream<String> wejscie)`, która próbuje sparsować każdy element jako liczbę całkowitą i:

1. dla poprawnych wartości emituje `int`,
2. dla niepoprawnych wartości **nie przerywa** strumienia, lecz loguje błąd przez `onError`/`handleError` i kontynuuje,
3. zwraca strumień poprawnie sparsowanych liczb.

Zademonstruj obsługę zarówno przez `handleError` (w łańcuchu), jak i przez callback `onError` w `listen`.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście (dane) |
|---------|-------------------|
| `['10', 'abc', '20']` | `[10, 20]` |
| `['x', '5', 'y']` | `[5]` |

### Wskazówki

1. Użyj `int.parse`, które rzuca `FormatException` dla niepoprawnych danych — nie używaj `int.tryParse`, bo celem jest ćwiczenie obsługi błędów
2. `map` wewnątrz łańcucha może rzucić — błąd stanie się zdarzeniem błędu strumienia
3. `handleError` pozwala przechwycić i zalogować błąd, a następnie kontynuować
4. Alternatywnie w `listen` obsłuż błędy przez parametr `onError`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:async';

/// Parsuje elementy strumienia na int; błędne elementy są logowane i pomijane.
Stream<int> bezpiecznePrzetwarzanie(Stream<String> wejscie) {
  return wejscie
      .map((s) => int.parse(s))   // rzuca FormatException dla błędnych danych
      .handleError((e) {
    // handleError przechwytuje błąd i pozwala kontynuować strumień
    // (source zamiast pełnego $e, bo toString FormatException zawiera
    // wieloliniowy fragment z pozycją błędu)
    final blad = e as FormatException;
    print('Pominięto błędny element: ${blad.runtimeType}: ${blad.source}');
  }, test: (e) => e is FormatException);
}

void main() async {
  final wynik = <int>[];

  // Wariant z callbackiem onError w listen
  final gotowe = Completer<void>();
  bezpiecznePrzetwarzanie(Stream.fromIterable(['10', 'abc', '20'])).listen(
    (v) => wynik.add(v),
    onError: (e) => print('onError: $e'), // nie zostanie wywołany — handleError już obsłużył
    onDone: () => gotowe.complete(),
  );
  await gotowe.future;

  print('Sparsowane: $wynik');
}
// Oczekiwane wyjście:
// Pominięto błędny element: FormatException: abc
// Sparsowane: [10, 20]
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`Stream<T>` to asynchroniczna sekwencja zdarzeń** — dostarcza zero, jedną lub wiele wartości w czasie, w odróżnieniu od `Future<T>`, które dostarcza dokładnie jedną wartość.
2. **Single-subscription vs broadcast** — domyślne strumienie przyjmują jednego słuchacza i buforują zdarzenia; strumienie broadcast obsługują wielu słuchaczy, ale nie buforują zdarzeń dla przyszłych subskrybentów.
3. **`StreamController`, generatory `async*` i konstruktory fabryczne** (`fromIterable`, `periodic`, `eventTransformed`) to główne sposoby tworzenia własnych strumieni.
4. **Transformacje są leniwe i łańcuchowalne** — `map`, `where`, `expand`, `take`, `skip`, `distinct` działają synchronicznie, a `asyncMap` i `asyncExpand` obsługują operacje asynchroniczne z zachowaniem kolejności.
5. **Błędy nie muszą przerywać strumienia** — `handleError` i callback `onError` pozwalają obsłużyć błąd i kontynuować, podczas gdy `await for` przerywa się na pierwszym błędzie.
6. **`StreamTransformer` umożliwia reużywalne, stanowe transformacje** — stan przechowywany w domknięciu pozwala budować sumy skumulowane, grupowanie w partie czy debounce.
7. **`StreamSubscription` steruje cyklem życia** — `pause`, `resume` i `cancel` dają kontrolę nad odbieraniem zdarzeń, co jest kluczowe dla zarządzania zasobami.

---

**Poprzedni moduł:** [Event loop i Futures (async/await)](../07-concurrency/01-async-await.md)
**Następny moduł:** [Isolates](../07-concurrency/03-isolates.md)
