---
id: "8.3"
title: "dart:async"
difficulty: "advanced"
section: "08-standard-library"
prerequisites:
  - "Event loop i Futures (async/await)"
  - "Streams"
---

# 8.3 dart:async

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Event loop i Futures (async/await)](../07-concurrency/01-async-await.md), [Streams](../07-concurrency/02-streams.md)
- **Cele nauki:**
  1. Wykorzystywać `Completer` do ręcznego, imperatywnego sterowania cyklem życia `Future` oraz wiedzieć, kiedy jest to konieczne
  2. Stosować `StreamController`, `StreamSink` i `StreamTransformer` w zaawansowanych scenariuszach mostkowania kodu callbackowego ze strumieniami
  3. Programować z użyciem `Timer` — zarówno jednorazowych, jak i cyklicznych — wraz z poprawnym anulowaniem
  4. Rozumieć koncepcję `Zone` i wykorzystywać `runZoned`/`runZonedGuarded` do przechwytywania błędów, przechowywania wartości lokalnych strefy i nadpisywania zachowań (`print`, `scheduleMicrotask`, `createTimer`)
  5. Świadomie wybierać między niskopoziomowymi narzędziami `dart:async` a wysokopoziomową składnią `async`/`await` i konstruktorami strumieni

---

## Wprowadzenie: co daje dart:async poza Future i Stream?

Biblioteka `dart:async` zawiera fundamenty programowania asynchronicznego w Dart. Znasz już `Future` i `Stream` z modułów o współbieżności. Ten moduł skupia się na **niskopoziomowych narzędziach**, po które sięga się, gdy składnia `async`/`await` i gotowe konstruktory strumieni nie wystarczają:

- **`Completer`** — pozwala utworzyć `Future` i ukończyć je ręcznie w dowolnym momencie, np. z wnętrza callbacku.
- **`StreamController` / `StreamSink`** — pozwalają zbudować strumień sterowany imperatywnie i wstrzykiwać do niego zdarzenia z zewnątrz.
- **`StreamTransformer`** — reużywalny obiekt przekształcający jeden strumień w inny.
- **`Timer`** — planuje wykonanie kodu w przyszłości (jednorazowo lub cyklicznie).
- **`Zone`** — kontekst wykonania, który przechwytuje błędy asynchroniczne i pozwala nadpisać podstawowe operacje (`print`, planowanie mikrozadań, tworzenie timerów).

Wszystkie przykłady wymagają importu biblioteki:

```dart
import 'dart:async';
```

---

## Completer — ręczne sterowanie Future

`Completer<T>` to "pilot zdalnego sterowania" dla `Future`. Tworzy obiekt `Future` dostępny przez getter `future`, który pozostaje nieukończony, dopóki nie wywołasz `complete(wartość)` (sukces) lub `completeError(błąd)` (błąd). Jest to jedyny sposób, aby ukończyć `Future` z zewnątrz — np. z callbacku, który nie jest funkcją `async`.

Poniższy przykład pokazuje podstawowe użycie: `Completer` jest ukończony po opóźnieniu, a kod czekający na `future` wznawia się dopiero po wywołaniu `complete`.

```dart
import 'dart:async';

void main() async {
  final completer = Completer<String>();

  // Planujemy ukończenie Future za 100 ms (np. reakcja na zdarzenie zewnętrzne)
  Timer(Duration(milliseconds: 100), () {
    completer.complete('gotowe'); // ręczne ukończenie Future sukcesem
  });

  print('Czekam na wynik...');
  final wynik = await completer.future; // wznawia się dopiero po complete
  print('Wynik: $wynik');
}
// Oczekiwane wyjście:
// Czekam na wynik...
// Wynik: gotowe
```

`Completer` obsługuje też ukończenie błędem. Getter `isCompleted` pozwala sprawdzić, czy `Future` zostało już ukończone — wywołanie `complete`/`completeError` po raz drugi rzuca wyjątek, więc warto się zabezpieczyć.

```dart
import 'dart:async';

void main() async {
  final completer = Completer<int>();

  Timer(Duration(milliseconds: 50), () {
    // completeError kończy Future błędem zamiast wartością
    completer.completeError(StateError('operacja nieudana'));
  });

  try {
    await completer.future;
  } on StateError catch (e) {
    print('Przechwycono błąd: ${e.message}');
  }

  // isCompleted chroni przed podwójnym ukończeniem (rzuciłoby wyjątek)
  print('Czy ukończony? ${completer.isCompleted}');
}
// Oczekiwane wyjście:
// Przechwycono błąd: operacja nieudana
// Czy ukończony? true
```

### Kiedy używać Completer (a kiedy async/await wystarcza)

W **zdecydowanej większości przypadków** `async`/`await` jest właściwym wyborem — jest czytelniejszy i mniej podatny na błędy. `Completer` sięgasz tylko wtedy, gdy `async`/`await` **nie jest w stanie** wyrazić danego wzorca.

**Scenariusz, w którym async/await jest niewystarczający:** mostkowanie API opartego na callbackach (callback-based) do świata `Future`. Wyobraź sobie starą bibliotekę, która przyjmuje callback `onSukces`/`onBlad` zamiast zwracać `Future`. Nie możesz jej `await`-ować, bo nie zwraca `Future`. `Completer` pozwala opakować taki interfejs.

```dart
import 'dart:async';

// Symulacja starego API opartego na callbackach — NIE zwraca Future
void staraBibliotekaPobierz(
  String zasob, {
  required void Function(String dane) onSukces,
  required void Function(Object blad) onBlad,
}) {
  Timer(Duration(milliseconds: 80), () {
    if (zasob.isEmpty) {
      onBlad(ArgumentError('pusty zasób'));
    } else {
      onSukces('dane dla "$zasob"');
    }
  });
}

// Adapter: opakowuje callbackowe API w Future za pomocą Completer
Future<String> pobierz(String zasob) {
  final completer = Completer<String>();

  staraBibliotekaPobierz(
    zasob,
    onSukces: (dane) => completer.complete(dane),      // callback -> complete
    onBlad: (blad) => completer.completeError(blad),   // callback -> completeError
  );

  return completer.future; // zwracamy Future, które możemy await-ować
}

void main() async {
  final wynik = await pobierz('uzytkownik/1'); // teraz działa await!
  print(wynik);
}
// Oczekiwane wyjście:
// dane dla "uzytkownik/1"
```

Bez `Completer` nie dałoby się przekształcić tego callbackowego API w coś, co da się `await`-ować — funkcja `async` musi bazować na istniejących `Future`, a tutaj żadnego nie ma. To właśnie scenariusz, w którym alternatywa (`async`/`await`) jest niewystarczająca.

---

## StreamController i StreamSink

`StreamController<T>` widziałeś w module o strumieniach — to główne narzędzie do tworzenia strumieni sterowanych imperatywnie. Tutaj pogłębiamy dwa aspekty: rolę `StreamSink` oraz zaawansowaną obsługę backpressure i callbacków cyklu życia.

`StreamSink<T>` to interfejs "wejścia" strumienia — to przez niego producent dodaje zdarzenia (`add`, `addError`, `addStream`, `close`). Getter `controller.sink` zwraca właśnie `StreamSink`. Rozdzielenie `sink` (wejście) od `stream` (wyjście) pozwala bezpiecznie przekazać producentowi tylko możliwość dodawania zdarzeń, a konsumentowi tylko możliwość ich odbioru.

Poniższy przykład pokazuje przekazanie `StreamSink` do osobnej funkcji-producenta, która nie ma dostępu do samego strumienia. Metoda `addStream` pozwala "przelać" cały inny strumień do sinka.

```dart
import 'dart:async';

// Producent dostaje tylko sink (wejście) — nie widzi strumienia wyjściowego
Future<void> produkuj(StreamSink<int> sink) async {
  sink.add(1);
  sink.add(2);
  // addStream przelewa cały strumień źródłowy do sinka i czeka na jego koniec
  await sink.addStream(Stream.fromIterable([3, 4, 5]));
  await sink.close(); // zamykamy sink -> strumień emituje zdarzenie done
}

void main() async {
  final controller = StreamController<int>();

  // Konsument dostaje tylko stream (wyjście)
  final subskrypcja = controller.stream.listen((v) => print('Odebrano: $v'));

  await produkuj(controller.sink); // przekazujemy wyłącznie wejście
  await subskrypcja.cancel();
}
// Oczekiwane wyjście:
// Odebrano: 1
// Odebrano: 2
// Odebrano: 3
// Odebrano: 4
// Odebrano: 5
```

`StreamController` udostępnia callbacki cyklu życia: `onListen` (pierwszy słuchacz), `onPause`, `onResume` i `onCancel`. Pozwalają one leniwie uruchamiać i zatrzymywać produkcję danych oraz zwalniać zasoby. Poniższy przykład używa `onListen`/`onCancel`, aby uruchomić i zatrzymać cykliczny `Timer` tylko wtedy, gdy ktoś faktycznie słucha.

```dart
import 'dart:async';

Stream<int> tykajacyStrumien() {
  late StreamController<int> controller;
  Timer? timer;
  var licznik = 0;

  controller = StreamController<int>(
    // onListen: startujemy produkcję dopiero, gdy pojawi się słuchacz
    onListen: () {
      timer = Timer.periodic(Duration(milliseconds: 50), (_) {
        controller.add(licznik++);
      });
    },
    // onCancel: sprzątamy zasoby, gdy słuchacz anuluje subskrypcję
    onCancel: () {
      timer?.cancel();
      print('Zasoby zwolnione');
    },
  );

  return controller.stream;
}

void main() async {
  final sub = tykajacyStrumien().listen((v) => print('Tik: $v'));

  await Future.delayed(Duration(milliseconds: 175));
  await sub.cancel(); // wyzwala onCancel -> zatrzymuje Timer
}
// Oczekiwane wyjście:
// Tik: 0
// Tik: 1
// Tik: 2
// Zasoby zwolnione
```

### Kiedy używać StreamController (a kiedy async* wystarcza)

Do tworzenia strumieni najczęściej wystarcza generator `async*` — jest zwięzły i automatycznie zarządza cyklem życia. `StreamController` sięgasz, gdy potrzebujesz **imperatywnego, sterowanego z zewnątrz** źródła zdarzeń.

**Scenariusz, w którym async* jest niewystarczający:** most między systemem zdarzeń (event-driven) a strumieniem, gdzie zdarzenia przychodzą z **wielu niezależnych, zewnętrznych źródeł callbackowych** w nieprzewidywalnym momencie. Generator `async*` musi sam "wyprodukować" każde `yield` w swoim ciele — nie potrafi czekać biernie na zdarzenia pchane z zewnątrz przez wiele różnych callbacków. `StreamController` pozwala dowolnemu fragmentowi kodu wywołać `controller.add(...)`.

```dart
import 'dart:async';

// Agregator zdarzeń z wielu niezależnych źródeł callbackowych.
// Generator async* nie potrafiłby tego wyrazić — nie ma jednej pętli produkcji.
class AgregatorZdarzen {
  final _controller = StreamController<String>.broadcast();

  Stream<String> get zdarzenia => _controller.stream;

  // Każde z tych źródeł może być wywołane niezależnie, z zewnątrz, w dowolnej chwili
  void zgloszenieZUI(String akcja) => _controller.add('UI: $akcja');
  void zgloszenieZSieci(String pakiet) => _controller.add('SIEĆ: $pakiet');
  void zgloszenieZCzujnika(int wartosc) => _controller.add('CZUJNIK: $wartosc');

  Future<void> zamknij() => _controller.close();
}

void main() async {
  final agregator = AgregatorZdarzen();
  final sub = agregator.zdarzenia.listen(print);

  // Zdarzenia pchane z różnych, niezależnych miejsc kodu
  agregator.zgloszenieZUI('klik');
  agregator.zgloszenieZCzujnika(42);
  agregator.zgloszenieZSieci('ping');

  await Future.delayed(Duration.zero); // pozwalamy dostarczyć zdarzenia
  await agregator.zamknij();
  await sub.cancel();
}
// Oczekiwane wyjście:
// UI: klik
// CZUJNIK: 42
// SIEĆ: ping
```

---

## StreamTransformer — reużywalne przekształcenia

`StreamTransformer<S, T>` to reużywalny obiekt przekształcający strumień typu `S` w strumień typu `T`, stosowany metodą `stream.transform(...)`. W module o strumieniach poznałeś `StreamTransformer.fromHandlers`. Tutaj pokażemy `StreamTransformer.fromBind` — bardziej elastyczny wariant, w którym sam definiujesz, jak strumień źródłowy jest przekształcany w wyjściowy. Pozwala on w pełni wykorzystać istniejące transformacje strumieni.

Poniższy przykład tworzy transformer, który dołącza indeks do każdego elementu (odpowiednik `enumerate`), wykorzystując wewnętrznie generator `async*`.

```dart
import 'dart:async';

// Transformer indeksujący: (indeks, wartość) dla każdego elementu.
// fromBind przyjmuje funkcję (Stream<S>) -> Stream<T>
StreamTransformer<T, (int, T)> zIndeksem<T>() {
  return StreamTransformer<T, (int, T)>.fromBind((zrodlo) async* {
    var indeks = 0;
    await for (final element in zrodlo) {
      yield (indeks++, element); // rekord: para (indeks, wartość)
    }
  });
}

void main() async {
  final litery = Stream<String>.fromIterable(['a', 'b', 'c']);

  final zindeksowane = litery.transform(zIndeksem<String>());

  await for (final (i, v) in zindeksowane) {
    print('$i -> $v');
  }
}
// Oczekiwane wyjście:
// 0 -> a
// 1 -> b
// 2 -> c
```

---

## Timer — planowanie kodu w czasie

`Timer` planuje jednorazowe lub cykliczne wykonanie callbacku. To niskopoziomowe narzędzie leżące u podstaw `Future.delayed`, `Stream.periodic` i wielu innych mechanizmów czasowych.

### Timer jednorazowy (one-shot)

Konstruktor `Timer(czas, callback)` planuje **jednokrotne** wykonanie callbacku po upływie `czas`. Metoda `cancel()` anuluje timer, jeśli jeszcze nie wypalił — po anulowaniu callback nie zostanie wywołany. Getter `isActive` mówi, czy timer wciąż czeka na wykonanie.

Poniższy przykład tworzy dwa jednorazowe timery: jeden zostaje anulowany przed wypaleniem, drugi wykonuje się normalnie.

```dart
import 'dart:async';

void main() async {
  // Timer jednorazowy — callback wykona się raz po 100 ms
  final timerA = Timer(Duration(milliseconds: 100), () {
    print('Timer A wypalił');
  });

  // Timer jednorazowy, który anulujemy zanim zdąży wypalić
  final timerB = Timer(Duration(milliseconds: 100), () {
    print('Timer B wypalił'); // NIE zostanie wywołany
  });

  print('B aktywny przed anulowaniem? ${timerB.isActive}');
  timerB.cancel(); // anulowanie — callback B nie zostanie wywołany
  print('B aktywny po anulowaniu? ${timerB.isActive}');

  await Future.delayed(Duration(milliseconds: 150));
  print('A aktywny po wypaleniu? ${timerA.isActive}');
}
// Oczekiwane wyjście:
// B aktywny przed anulowaniem? true
// B aktywny po anulowaniu? false
// Timer A wypalił
// A aktywny po wypaleniu? false
```

### Timer cykliczny (periodic)

Konstruktor `Timer.periodic(czas, callback)` wykonuje callback **wielokrotnie**, w regularnych odstępach `czas`, aż do wywołania `cancel()`. Callback otrzymuje sam obiekt `Timer` jako argument, dzięki czemu można go anulować z jego wnętrza (np. po osiągnięciu warunku stopu).

Poniższy przykład tworzy timer cykliczny odliczający tiki i anulujący się samodzielnie po piątym tiku.

```dart
import 'dart:async';

void main() async {
  var tiki = 0;

  // Timer.periodic wykonuje callback co 50 ms; argument to sam Timer
  final timer = Timer.periodic(Duration(milliseconds: 50), (t) {
    tiki++;
    print('Tik $tiki');

    if (tiki == 5) {
      t.cancel(); // anulowanie z wnętrza callbacku po 5 tikach
      print('Timer anulowany');
    }
  });

  // Czekamy dłużej niż potrzeba — timer i tak zatrzyma się po 5 tikach
  await Future.delayed(Duration(milliseconds: 500));
  print('Aktywny? ${timer.isActive}');
}
// Oczekiwane wyjście:
// Tik 1
// Tik 2
// Tik 3
// Tik 4
// Tik 5
// Timer anulowany
// Aktywny? false
```

---

## Zone — konteksty wykonania asynchronicznego

`Zone` to jeden z najbardziej zaawansowanych mechanizmów `dart:async`. Strefa (zone) reprezentuje **kontekst wykonania** kodu asynchronicznego. Każdy fragment kodu działa w jakiejś strefie (domyślnie w `Zone.root`), a strefy propagują się przez granice asynchroniczne — callbacki `Future`, `Timer` i mikrozadania "pamiętają", w której strefie zostały utworzone.

Strefy dają trzy główne możliwości:

1. **Przechwytywanie nieobsłużonych błędów asynchronicznych** (error handling zones),
2. **Przechowywanie wartości lokalnych strefy** (zone-local values),
3. **Nadpisywanie podstawowych operacji** (`print`, `scheduleMicrotask`, `createTimer` i inne).

### Strefy obsługi błędów: runZonedGuarded

Zwykły `try/catch` nie przechwyci błędu, który powstanie asynchronicznie **po** zakończeniu bloku `try` (np. w callbacku `Timer` albo w niezaawaitowanym `Future`). `runZonedGuarded` uruchamia kod w strefie z globalnym handlerem błędów, który łapie **wszystkie** nieobsłużone błędy asynchroniczne powstałe w tej strefie.

Poniższy przykład pokazuje błąd rzucony wewnątrz callbacku `Timer` — normalnie zakończyłby program, ale `runZonedGuarded` go przechwytuje.

```dart
import 'dart:async';

void main() {
  // runZonedGuarded: drugi argument to handler nieobsłużonych błędów strefy
  runZonedGuarded(() {
    print('Start strefy');

    // Błąd powstanie asynchronicznie — zwykły try/catch by go nie złapał
    Timer(Duration(milliseconds: 50), () {
      throw StateError('błąd w callbacku Timera');
    });
  }, (blad, stos) {
    // Ten handler przechwytuje błąd asynchroniczny z całej strefy
    print('Przechwycono w strefie: $blad');
  });
}
// Oczekiwane wyjście:
// Start strefy
// Przechwycono w strefie: Bad state: błąd w callbacku Timera
```

### Wartości lokalne strefy: zone-local values

Strefa może przechowywać wartości powiązane z kontekstem, dostępne przez `Zone.current[klucz]`. Wartości te przekazuje się w mapie `zoneValues` do `runZoned`. Są one **niejawnie propagowane** przez cały kod asynchroniczny działający w strefie — to wygodny mechanizm np. do przekazywania identyfikatora żądania (request ID) czy kontekstu użytkownika bez jawnego przekazywania parametrów.

Poniższy przykład zapisuje identyfikator żądania jako wartość lokalną strefy i odczytuje go w zagnieżdżonej funkcji asynchronicznej, bez przekazywania go jako argument.

```dart
import 'dart:async';

// Odczytuje wartość lokalną strefy — nie potrzebuje jej jako parametru
Future<void> zalogujKrok(String opis) async {
  await Future.delayed(Duration(milliseconds: 10));
  final idZadania = Zone.current[#idZadania]; // odczyt wartości lokalnej strefy
  print('[żądanie $idZadania] $opis');
}

void main() async {
  // zoneValues wstrzykuje wartości lokalne dostępne w całej strefie
  await runZoned(() async {
    await zalogujKrok('walidacja');
    await zalogujKrok('zapis do bazy');
  }, zoneValues: {#idZadania: 'REQ-777'});
}
// Oczekiwane wyjście:
// [żądanie REQ-777] walidacja
// [żądanie REQ-777] zapis do bazy
```

### Nadpisywanie specyfikacji strefy: print, scheduleMicrotask, createTimer

`ZoneSpecification` pozwala nadpisać podstawowe operacje wykonywane w strefie. Trzy szczególnie użyteczne to:

- **`print`** — przechwytuje wszystkie wywołania `print` w strefie (np. do dodania prefiksu, przekierowania do logów),
- **`scheduleMicrotask`** — przechwytuje planowanie mikrozadań,
- **`createTimer`** — przechwytuje tworzenie timerów (np. do testów z przyspieszonym czasem).

Poniższy przykład nadpisuje wszystkie trzy operacje: `print` dostaje prefiks z datownikiem-zastępczym, a planowanie mikrozadań i tworzenie timerów jest liczone.

```dart
import 'dart:async';

void main() async {
  var liczbaMikrozadan = 0;
  var liczbaTimerow = 0;

  final specyfikacja = ZoneSpecification(
    // Nadpisanie print: dodaje prefiks i deleguje do rodzica (parent.print)
    print: (self, parent, zone, wiadomosc) {
      parent.print(zone, '[LOG] $wiadomosc');
    },
    // Nadpisanie scheduleMicrotask: zlicza i deleguje wykonanie do rodzica
    scheduleMicrotask: (self, parent, zone, zadanie) {
      liczbaMikrozadan++;
      parent.scheduleMicrotask(zone, zadanie);
    },
    // Nadpisanie createTimer: zlicza i tworzy timer przez rodzica
    createTimer: (self, parent, zone, czas, callback) {
      liczbaTimerow++;
      return parent.createTimer(zone, czas, callback);
    },
  );

  await runZoned(() async {
    print('cześć ze strefy');                 // trafia w nadpisany print
    scheduleMicrotask(() => print('mikrozadanie')); // trafia w nadpisany scheduleMicrotask
    Timer(Duration(milliseconds: 20), () => print('timer')); // nadpisany createTimer
    await Future.delayed(Duration(milliseconds: 40));
  }, zoneSpecification: specyfikacja);

  print('Mikrozadań: $liczbaMikrozadan, timerów: $liczbaTimerow');
}
// Oczekiwane wyjście:
// [LOG] cześć ze strefy
// [LOG] mikrozadanie
// [LOG] timer
// Mikrozadań: 1, timerów: 2
```

Liczba timerów wynosi 2, a nie 1 — oprócz jawnego `Timer(...)` również `Future.delayed` wewnętrznie tworzy timer, więc `createTimer` zostaje wywołany dwukrotnie.

Zwróć uwagę na wzorzec `parent.print(zone, ...)` / `parent.scheduleMicrotask(zone, ...)` — nadpisany handler zwykle **deleguje** faktyczne działanie do strefy rodzica (`parent`), dodając własne zachowanie (prefiks, zliczanie). Pominięcie delegacji spowodowałoby, że np. `print` nic by nie wypisał.

---

## Ćwiczenie 1: Rejestrator błędów asynchronicznych ze strefą

### Opis problemu

Zaimplementuj funkcję `uruchomZRejestracjaBledow(void Function() operacja)`, która uruchamia przekazaną operację w **niestandardowej strefie** przechwytującej wszystkie nieobsłużone błędy asynchroniczne. Funkcja ma zwrócić `Future<List<String>>` zawierającą komunikaty (`toString`) wszystkich błędów przechwyconych w trakcie działania operacji.

Wykorzystaj `runZonedGuarded`. Operacja może rzucać błędy synchronicznie oraz asynchronicznie (np. z callbacków `Timer`). Poczekaj chwilę, aby zebrać błędy asynchroniczne, a następnie zwróć zebraną listę.

**Poziom trudności:** advanced

### Przykłady wejścia/wyjścia

| Wejście (operacja) | Oczekiwane wyjście |
|---------|-------------------|
| operacja rzucająca `Timer(..., () => throw 'A')` i `Timer(..., () => throw 'B')` | `[A, B]` |
| operacja bez żadnych błędów | `[]` |

### Wskazówki

1. Utwórz lokalną listę `bledy = <String>[]` i uzupełniaj ją w handlerze `runZonedGuarded`.
2. Handler błędu ma sygnaturę `(Object blad, StackTrace stos)` — dodawaj `blad.toString()` do listy.
3. Użyj `Completer` lub `Future.delayed`, aby odczekać na dostarczenie błędów asynchronicznych zanim zwrócisz wynik.
4. Pamiętaj, że `runZonedGuarded` nie blokuje — błędy asynchroniczne przyjdą po zakończeniu ciała operacji.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:async';

/// Uruchamia [operacja] w strefie przechwytującej błędy asynchroniczne
/// i zwraca listę komunikatów przechwyconych błędów.
Future<List<String>> uruchomZRejestracjaBledow(void Function() operacja) async {
  final bledy = <String>[];

  runZonedGuarded(operacja, (blad, stos) {
    bledy.add(blad.toString()); // rejestrujemy każdy nieobsłużony błąd strefy
  });

  // Dajemy czas na dostarczenie błędów asynchronicznych (callbacki Timerów itp.)
  await Future.delayed(Duration(milliseconds: 100));
  return bledy;
}

void main() async {
  // Operacja rzucająca dwa błędy asynchroniczne
  final wynik1 = await uruchomZRejestracjaBledow(() {
    Timer(Duration(milliseconds: 10), () => throw 'A');
    Timer(Duration(milliseconds: 20), () => throw 'B');
  });
  print(wynik1);

  // Operacja bez błędów
  final wynik2 = await uruchomZRejestracjaBledow(() {
    Timer(Duration(milliseconds: 10), () => print('bez błędu'));
  });
  print(wynik2);
}
// Oczekiwane wyjście:
// [A, B]
// bez błędu
// []
```

</details>

---

## Ćwiczenie 2: Przechwytywanie wywołań print przez ZoneSpecification

### Opis problemu

Zaimplementuj funkcję `przechwycWydruki(void Function() operacja)`, która uruchamia operację w strefie przechwytującej **wszystkie wywołania `print`** i zwraca `List<String>` z przechwyconymi komunikatami — zamiast wypisywać je na konsolę. Wykorzystaj `ZoneSpecification` z nadpisanym handlerem `print` oraz `runZoned`.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście (operacja) | Oczekiwane wyjście |
|---------|-------------------|
| `() { print('a'); print('b'); }` | `[a, b]` |
| `() { print('tylko jeden'); }` | `[tylko jeden]` |

### Wskazówki

1. Utwórz `ZoneSpecification(print: (self, parent, zone, wiadomosc) { ... })`.
2. W handlerze **nie** wołaj `parent.print(...)` — zamiast tego dodaj `wiadomosc` do lokalnej listy, aby wydruk nie trafił na konsolę.
3. Przekaż specyfikację do `runZoned(operacja, zoneSpecification: spec)`.
4. Zwróć zebraną listę po wykonaniu operacji.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:async';

/// Uruchamia [operacja] w strefie przechwytującej wywołania print
/// i zwraca zebrane komunikaty zamiast wypisywać je na konsolę.
List<String> przechwycWydruki(void Function() operacja) {
  final wydruki = <String>[];

  final spec = ZoneSpecification(
    // Nadpisany print zapisuje komunikat do listy i NIE deleguje do rodzica
    print: (self, parent, zone, wiadomosc) {
      wydruki.add(wiadomosc);
    },
  );

  runZoned(operacja, zoneSpecification: spec);
  return wydruki;
}

void main() {
  final zebrane1 = przechwycWydruki(() {
    print('a');
    print('b');
  });
  // Uwaga: ten print jest POZA strefą, więc wypisze się normalnie
  print(zebrane1);

  final zebrane2 = przechwycWydruki(() {
    print('tylko jeden');
  });
  print(zebrane2);
}
// Oczekiwane wyjście:
// [a, b]
// [tylko jeden]
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`Completer` daje ręczną kontrolę nad `Future`** — tworzysz `Future` i kończysz je jawnie przez `complete`/`completeError`. Sięgaj po niego głównie do mostkowania API opartego na callbackach, gdy `async`/`await` nie ma na czym się oprzeć.
2. **`StreamController` i `StreamSink` budują strumienie sterowane imperatywnie** — `sink` to wejście (dodawanie zdarzeń), `stream` to wyjście (subskrypcja). Wybieraj je zamiast `async*`, gdy zdarzenia są pchane z wielu niezależnych, zewnętrznych źródeł.
3. **`StreamTransformer` (w tym `fromBind`) tworzy reużywalne przekształcenia** strumieni, które można w pełni zaimplementować przy użyciu generatorów `async*`.
4. **`Timer` planuje kod w czasie** — jednorazowo (`Timer`) lub cyklicznie (`Timer.periodic`); zawsze pamiętaj o `cancel()`, aby zwolnić zasoby i uniknąć wycieków.
5. **`Zone` to kontekst wykonania asynchronicznego** propagowany przez granice `Future`, `Timer` i mikrozadań.
6. **`runZonedGuarded` przechwytuje nieobsłużone błędy asynchroniczne**, których nie złapie zwykły `try/catch`, a `zoneValues` pozwala przechowywać wartości lokalne strefy (np. request ID).
7. **`ZoneSpecification` nadpisuje podstawowe operacje** (`print`, `scheduleMicrotask`, `createTimer`) — nadpisany handler zwykle deleguje działanie do strefy rodzica (`parent`), dodając własne zachowanie.

---

**Poprzedni moduł:** [dart:collection](../08-standard-library/02-dart-collection.md)
**Następny moduł:** [dart:io](../08-standard-library/04-dart-io.md)
