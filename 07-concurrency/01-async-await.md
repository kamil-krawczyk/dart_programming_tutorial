---
id: "7.1"
title: "Event loop, Futures i async/await"
difficulty: "advanced"
section: "07-concurrency"
prerequisites:
  - "Deklaracje funkcji i parametry"
  - "Lambdy i typedef"
---

# 7.1 Event loop, Futures i async/await

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Deklaracje funkcji i parametry](../04-functions/01-declarations-params.md), [Lambdy i typedef](../04-functions/02-lambdas-typedef.md)
- **Cele nauki:**
  1. Zrozumieć model pętli zdarzeń (event loop) w Dart oraz różnicę w kolejności przetwarzania kolejki mikrozadań (microtask queue) i kolejki zdarzeń (event queue)
  2. Tworzyć i komponować obiekty `Future` przy pomocy `async`/`await`, `then`/`catchError`/`whenComplete` oraz kombinatorów `Future.wait`, `Future.any`, `Future.forEach`
  3. Ręcznie zarządzać zakończeniem operacji asynchronicznej za pomocą klasy `Completer`
  4. Rozpoznawać i unikać typowych pułapek: nieobsłużonych wyjątków, brakującego `await` oraz sekwencyjnego wykonania tam, gdzie możliwe jest równoległe

---

## Model pętli zdarzeń (event loop)

Dart jest językiem **jednowątkowym** w obrębie pojedynczego izolatu. Cały kod asynchroniczny działa dzięki **pętli zdarzeń** (event loop), która nieprzerwanie pobiera zadania z dwóch kolejek i wykonuje je pojedynczo. Dopóki wykonuje się jakiś fragment kodu (synchroniczny), pętla czeka — nic nie może go przerwać. Dopiero gdy bieżące zadanie zakończy się, pętla sięga po kolejne.

Istnieją **dwie kolejki**:

- **Kolejka mikrozadań (microtask queue)** — ma **najwyższy priorytet**. Trafiają tu zadania planowane przez `scheduleMicrotask` oraz kontynuacje `Future` (to, co dzieje się po `await` i w `then`). Pętla opróżnia całą tę kolejkę, zanim tknie kolejkę zdarzeń.
- **Kolejka zdarzeń (event queue)** — ma **niższy priorytet**. Trafiają tu zdarzenia I/O, timery (`Timer`, `Future.delayed`), zdarzenia rysowania oraz zadania planowane przez zwykły `Future(() => ...)`.

### Algorytm pętli (kolejność przetwarzania kolejek)

Poniższy diagram tekstowy pokazuje kolejność, w jakiej pętla zdarzeń poluje na obie kolejki. Kluczowa zasada: **po każdym pojedynczym zdarzeniu z kolejki zdarzeń pętla najpierw opróżnia CAŁĄ kolejkę mikrozadań**, dopiero potem bierze kolejne zdarzenie.

```text
        ┌─────────────────────────────────────────────┐
        │  Uruchom kod synchroniczny w main()          │
        └───────────────────────┬─────────────────────┘
                                 │
                                 ▼
        ┌─────────────────────────────────────────────┐
   ┌───▶│  1. Czy kolejka MIKROZADAŃ jest niepusta?    │
   │    └───────────────────────┬─────────────────────┘
   │                            │ tak
   │                            ▼
   │    ┌─────────────────────────────────────────────┐
   │    │     Wykonaj JEDNO mikrozadanie ──────────────┼──┐
   │    └─────────────────────────────────────────────┘  │
   │                            ▲                          │
   │                            └──────────────────────────┘
   │                            │ kolejka mikrozadań pusta
   │                            ▼
   │    ┌─────────────────────────────────────────────┐
   │    │  2. Czy kolejka ZDARZEŃ jest niepusta?       │
   │    └───────────────────────┬─────────────────────┘
   │                            │ tak
   │                            ▼
   │    ┌─────────────────────────────────────────────┐
   │    │  Wykonaj JEDNO zdarzenie (np. timer, I/O)    │
   │    └───────────────────────┬─────────────────────┘
   │                            │
   └────────────────────────────┘  (wróć do kroku 1 — najpierw mikrozadania!)
```

### Przykładowa sekwencja przetwarzania (3+ kroki)

Prześledźmy konkretny przebieg. Ten przykład ilustruje co najmniej trzy kroki pracy pętli obsługującej obie kolejki i pokazuje, dlaczego mikrozadania wyprzedzają zdarzenia.

```dart
import 'dart:async';

void main() {
  print('1. start (sync)');

  // Trafia do KOLEJKI ZDARZEŃ (niższy priorytet)
  Future(() => print('4. Future (event queue)'));

  // Trafia do KOLEJKI MIKROZADAŃ (wyższy priorytet)
  scheduleMicrotask(() => print('3. scheduleMicrotask (microtask)'));

  print('2. koniec main (sync)');
}
// Kolejność przetwarzania pętli zdarzeń:
//   KROK 1 (sync): wypisz "1. start" i "2. koniec main"
//   KROK 2 (opróżnij mikrozadania): wypisz "3. scheduleMicrotask"
//   KROK 3 (dopiero teraz kolejka zdarzeń): wypisz "4. Future"
// Oczekiwane wyjście:
// 1. start (sync)
// 2. koniec main (sync)
// 3. scheduleMicrotask (microtask)
// 4. Future (event queue)
```

Drugi przykład pokazuje, że nawet gdy zdarzenie z kolejki zdarzeń doda nowe mikrozadanie, to mikrozadanie zostanie wykonane, zanim pętla weźmie kolejne zdarzenie:

```dart
import 'dart:async';

void main() {
  print('start');

  // Dwa zdarzenia w kolejce zdarzeń
  Future(() {
    print('zdarzenie A');
    // Wewnątrz zdarzenia planujemy mikrozadanie...
    scheduleMicrotask(() => print('  -> mikrozadanie z A (przed zdarzeniem B)'));
  });
  Future(() => print('zdarzenie B'));

  print('koniec sync');
}
// Oczekiwane wyjście:
// start
// koniec sync
// zdarzenie A
//   -> mikrozadanie z A (przed zdarzeniem B)
// zdarzenie B
```

---

## Klasa Future

`Future<T>` reprezentuje wartość, która będzie dostępna **w przyszłości** — wynik operacji asynchronicznej. `Future` może być w jednym z trzech stanów: *niezakończony* (pending), *zakończony sukcesem* (completed with value) lub *zakończony błędem* (completed with error).

Najprostsze sposoby utworzenia `Future`:

```dart
import 'dart:async';

void main() {
  // Future zakończony natychmiast wartością — trafia do kolejki mikrozadań
  Future<int> gotowy = Future.value(42);

  // Future zakończony błędem (typ void — tylko sygnalizuje niepowodzenie)
  Future<void> zbledem = Future.error('coś poszło nie tak');

  // Future z opóźnieniem — trafia do kolejki zdarzeń po upływie czasu
  Future<String> zOpoznieniem =
      Future.delayed(Duration(milliseconds: 100), () => 'gotowe');

  gotowy.then((v) => print('Wartość: $v'));
  // Handler catchError dla Future<void> może zwracać void
  zbledem.catchError((e) => print('Błąd: $e'));
  zOpoznieniem.then((v) => print('Po opóźnieniu: $v'));
}
// Oczekiwane wyjście:
// Wartość: 42
// Błąd: coś poszło nie tak
// Po opóźnieniu: gotowe
```

Funkcja oznaczona `async` **zawsze** zwraca `Future`, nawet jeśli w ciele zwraca zwykłą wartość — Dart automatycznie opakuje ją w `Future`:

```dart
// Funkcja async zawsze zwraca Future<T>, choć return zwraca zwykłe T
Future<int> podwoj(int x) async {
  return x * 2; // zwrócone jako Future<int>
}

void main() async {
  // await rozpakowuje Future i zwraca wartość typu int
  int wynik = await podwoj(21);
  print('Wynik: $wynik');
}
// Oczekiwane wyjście:
// Wynik: 42
```

---

## Składnia async/await

Słowo kluczowe `await` **wstrzymuje** wykonanie funkcji `async` do momentu zakończenia oczekiwanego `Future`, a następnie zwraca jego wartość. Kod po `await` staje się kontynuacją planowaną w kolejce mikrozadań. Dzięki temu kod asynchroniczny czyta się jak synchroniczny.

```dart
// await sprawia, że kod asynchroniczny wygląda sekwencyjnie
Future<String> pobierzNazweUzytkownika(int id) async {
  // Symulacja opóźnienia sieciowego
  await Future.delayed(Duration(milliseconds: 50));
  return 'użytkownik_$id';
}

void main() async {
  print('Pobieram...');
  // await czeka na wynik i przypisuje go do zmiennej
  var nazwa = await pobierzNazweUzytkownika(7);
  print('Pobrano: $nazwa');
}
// Oczekiwane wyjście:
// Pobieram...
// Pobrano: użytkownik_7
```

Wyjątki z operacji asynchronicznych łapiemy zwykłym `try`/`catch`/`finally` — to jedna z największych zalet `async`/`await` nad `then`:

```dart
// Obsługa błędów async przez try/catch/finally
Future<double> podzielAsync(int a, int b) async {
  await Future.delayed(Duration(milliseconds: 10));
  if (b == 0) {
    throw ArgumentError('Dzielenie przez zero');
  }
  return a / b;
}

void main() async {
  try {
    var wynik = await podzielAsync(10, 0);
    print('Wynik: $wynik');
  } on ArgumentError catch (e) {
    print('Złapano błąd: ${e.message}');
  } finally {
    print('Blok finally zawsze się wykonuje');
  }
}
// Oczekiwane wyjście:
// Złapano błąd: Dzielenie przez zero
// Blok finally zawsze się wykonuje
```

---

## Future.then / catchError / whenComplete

Zanim pojawiło się `async`/`await`, `Future` obsługiwano metodami łańcuchowymi. Warto je znać — bywają zwięzłe i są używane w wielu bibliotekach.

- `then((wartość) => ...)` — rejestruje callback wykonywany po pomyślnym zakończeniu; zwraca nowy `Future`, więc można łańcuchować.
- `catchError((błąd) => ...)` — przechwytuje błąd z poprzedzającego `Future`.
- `whenComplete(() => ...)` — wykonuje się **zawsze** (sukces albo błąd), odpowiednik `finally`.

```dart
// Łańcuch then przekazuje wynik do kolejnego then
void main() {
  Future.value(5)
      .then((v) => v + 3)         // 5 -> 8
      .then((v) => v * 2)         // 8 -> 16
      .then((v) => print('Wynik łańcucha: $v'));
}
// Oczekiwane wyjście:
// Wynik łańcucha: 16
```

Poniższy przykład łączy wszystkie trzy metody i pokazuje, że `whenComplete` uruchamia się niezależnie od tego, czy wystąpił błąd:

```dart
// then + catchError + whenComplete — pełny cykl obsługi Future
Future<int> ryzykownaOperacja(bool czyBlad) {
  return Future.delayed(Duration(milliseconds: 10), () {
    if (czyBlad) throw StateError('awaria');
    return 100;
  });
}

void main() {
  ryzykownaOperacja(true)
      .then((v) => print('Sukces: $v'))
      .catchError((e) => print('catchError: $e')) // przechwytuje StateError
      .whenComplete(() => print('whenComplete: sprzątanie')); // zawsze
}
// Oczekiwane wyjście:
// catchError: Bad state: awaria
// whenComplete: sprzątanie
```

---

## Kombinatory Future (wait, any, forEach)

Kombinatory pozwalają operować na wielu obiektach `Future` naraz.

### Future.wait — czekaj na wszystkie

`Future.wait` przyjmuje listę obiektów `Future` i zwraca jeden `Future` z listą wyników. Wszystkie operacje ruszają **równolegle**, a wynik jest gotowy, gdy zakończy się ostatnia z nich.

```dart
// Future.wait uruchamia wszystkie zadania równolegle i zbiera wyniki
Future<int> zadanie(int id, int ms) async {
  await Future.delayed(Duration(milliseconds: ms));
  return id * 10;
}

void main() async {
  // Wszystkie trzy startują naraz; łączny czas ~ najdłuższe (30 ms), nie suma
  List<int> wyniki = await Future.wait([
    zadanie(1, 30),
    zadanie(2, 10),
    zadanie(3, 20),
  ]);
  print('Wyniki: $wyniki'); // kolejność zgodna z wejściem, nie z czasem
}
// Oczekiwane wyjście:
// Wyniki: [10, 20, 30]
```

### Future.any — czekaj na pierwszy

`Future.any` zwraca wynik **pierwszego** zakończonego `Future` (sukces lub błąd), ignorując pozostałe. Przydatne np. przy wyścigu z limitem czasu.

```dart
// Future.any zwraca wynik tego Future, który skończy się jako pierwszy
Future<String> zrodlo(String nazwa, int ms) async {
  await Future.delayed(Duration(milliseconds: ms));
  return nazwa;
}

void main() async {
  var pierwszy = await Future.any([
    zrodlo('wolne', 100),
    zrodlo('szybkie', 20),
    zrodlo('średnie', 50),
  ]);
  print('Najszybsze źródło: $pierwszy');
}
// Oczekiwane wyjście:
// Najszybsze źródło: szybkie
```

### Future.forEach — sekwencyjna iteracja asynchroniczna

`Future.forEach` wykonuje asynchroniczną akcję dla każdego elementu **po kolei**, czekając na zakończenie poprzedniej przed rozpoczęciem następnej.

```dart
// Future.forEach przetwarza elementy sekwencyjnie, jeden po drugim
void main() async {
  var elementy = ['a', 'b', 'c'];
  await Future.forEach(elementy, (String e) async {
    await Future.delayed(Duration(milliseconds: 10));
    print('Przetworzono: $e');
  });
  print('Wszystkie elementy przetworzone');
}
// Oczekiwane wyjście:
// Przetworzono: a
// Przetworzono: b
// Przetworzono: c
// Wszystkie elementy przetworzone
```

---

## Klasa Completer i ręczne zarządzanie Future

`Completer<T>` pozwala **ręcznie** utworzyć `Future` i zakończyć go w dowolnym momencie — z wartością (`complete`) lub błędem (`completeError`). Jest to most między światem callbacków (np. starsze API, timery, zdarzenia) a światem `Future`/`async`.

```dart
import 'dart:async';

// Completer buduje Future kończony ręcznie z zewnątrz
Future<String> czekajNaSygnal() {
  var completer = Completer<String>();

  // Symulacja zdarzenia z zewnątrz, które kończy Future po czasie
  Timer(Duration(milliseconds: 30), () {
    completer.complete('sygnał odebrany');
  });

  // Zwracamy Future powiązany z completerem
  return completer.future;
}

void main() async {
  print('Czekam na sygnał...');
  var wynik = await czekajNaSygnal();
  print(wynik);
}
// Oczekiwane wyjście:
// Czekam na sygnał...
// sygnał odebrany
```

`Completer` nadaje się też do zamiany API opartego na callbackach na `Future`. Uwaga: `Completer` można zakończyć **tylko raz** — dlatego warto sprawdzać `isCompleted`.

```dart
import 'dart:async';

// Opakowanie callbacka w Future za pomocą Completer, z ochroną przed podwójnym complete
Future<int> pierwszyWynik(void Function(void Function(int)) rejestruj) {
  var completer = Completer<int>();
  rejestruj((int wartosc) {
    // Zakończ tylko za pierwszym razem — kolejne wywołania są ignorowane
    if (!completer.isCompleted) {
      completer.complete(wartosc);
    }
  });
  return completer.future;
}

void main() async {
  var wynik = await pierwszyWynik((callback) {
    callback(1); // ten zakończy Future
    callback(2); // zignorowany — completer już zakończony
  });
  print('Pierwszy wynik: $wynik');
}
// Oczekiwane wyjście:
// Pierwszy wynik: 1
```

---

## Priorytet mikrozadań nad kolejką zdarzeń

Ten przykład demonstruje wprost priorytet kolejki mikrozadań, planując zadania przez `scheduleMicrotask` oraz `Future`. Wyjście jest opatrzone oczekiwaną kolejnością wykonania. Reguła: **kod synchroniczny → wszystkie mikrozadania → zdarzenia** (a przed każdym kolejnym zdarzeniem znów opróżniane są mikrozadania).

```dart
import 'dart:async';

// Mieszanka zadań pokazująca priorytet mikrozadań nad zdarzeniami
void main() {
  print('A: sync — 1');

  Future(() => print('E: Future/event — 5'));         // kolejka zdarzeń
  scheduleMicrotask(() => print('C: microtask — 3')); // kolejka mikrozadań
  Future.microtask(() => print('D: Future.microtask — 4')); // też mikrozadanie
  Future(() => print('F: Future/event — 6'));         // kolejka zdarzeń

  print('B: sync — 2');
}
// Oczekiwana kolejność wykonania:
//   Najpierw cały kod synchroniczny (1, 2),
//   potem WSZYSTKIE mikrozadania w kolejności zaplanowania (3, 4),
//   na końcu zdarzenia w kolejności zaplanowania (5, 6).
// Oczekiwane wyjście:
// A: sync — 1
// B: sync — 2
// C: microtask — 3
// D: Future.microtask — 4
// E: Future/event — 5
// F: Future/event — 6
```

`Future.microtask` różni się od zwykłego `Future` właśnie tym, do której kolejki trafia zadanie — mikrozadanie wykona się przed każdym zadaniem z kolejki zdarzeń:

```dart
import 'dart:async';

// Future.microtask vs Future — ta sama treść, inna kolejka i priorytet
void main() {
  Future(() => print('2. zwykły Future (event queue)'));
  Future.microtask(() => print('1. Future.microtask (microtask queue)'));
}
// Oczekiwane wyjście:
// 1. Future.microtask (microtask queue)
// 2. zwykły Future (event queue)
```

---

## Typowa pułapka 1: nieobsłużone wyjątki

Jeśli `Future` zakończy się błędem, a nikt go nie obsłuży (`await` w `try/catch`, `catchError` lub `onError`), błąd staje się **nieobsłużonym wyjątkiem asynchronicznym**. Poniżej porównanie błędu **złapanego** i **niezłapanego**.

```dart
// Błąd ZŁAPANY — obsłużony przez catchError, program działa dalej normalnie
void main() {
  Future.error('błąd A')
      .catchError((e) => print('Złapano: $e')); // obsłużone

  print('main kończy się bez problemu');
}
// Oczekiwane wyjście:
// main kończy się bez problemu
// Złapano: błąd A
```

Gdy błędu nikt nie obsłuży, trafia on do handlera nieobsłużonych błędów strefy (Zone). Program nie zatrzymuje pozostałych zadań, ale błąd zostaje zaraportowany:

```dart
import 'dart:async';

// Błąd NIEZŁAPANY — bez catchError trafia do handlera nieobsłużonych błędów strefy
void main() {
  // runZonedGuarded przechwytuje nieobsłużone błędy asynchroniczne
  runZonedGuarded(() {
    Future.error('błąd B'); // brak catchError — błąd nieobsłużony!
    print('zaplanowano błędny Future');
  }, (blad, stos) {
    print('Handler strefy przechwycił nieobsłużony błąd: $blad');
  });
}
// Oczekiwane wyjście:
// zaplanowano błędny Future
// Handler strefy przechwycił nieobsłużony błąd: błąd B
```

---

## Typowa pułapka 2: zapomniany await

Pominięcie `await` sprawia, że kod nie czeka na zakończenie operacji — funkcja rusza dalej, a efekty uboczne pojawiają się w nieoczekiwanej kolejności (lub „za późno"). To jeden z najczęstszych błędów.

```dart
// BRAKUJĄCY await — obserwowalny efekt uboczny w złej kolejności
Future<void> zapisz(String dane) async {
  await Future.delayed(Duration(milliseconds: 20));
  print('  [zapisano: $dane]');
}

void main() async {
  print('start');
  zapisz('plik.txt'); // BŁĄD: brak await — nie czekamy na zapis
  print('koniec'); // wykona się PRZED zakończeniem zapisu
  // Dajemy pętli czas na dokończenie osieroconego Future (tylko dla demonstracji)
  await Future.delayed(Duration(milliseconds: 50));
}
// Oczekiwane wyjście (zapis pojawia się PO "koniec"):
// start
// koniec
//   [zapisano: plik.txt]
```

Wersja poprawna z `await` wymusza właściwą kolejność:

```dart
// POPRAWNIE — await gwarantuje, że zapis zakończy się przed dalszym kodem
Future<void> zapisz(String dane) async {
  await Future.delayed(Duration(milliseconds: 20));
  print('  [zapisano: $dane]');
}

void main() async {
  print('start');
  await zapisz('plik.txt'); // czekamy na zakończenie zapisu
  print('koniec'); // wykona się PO zapisie
}
// Oczekiwane wyjście:
// start
//   [zapisano: plik.txt]
// koniec
```

---

## Typowa pułapka 3: wykonanie sekwencyjne vs równoległe

Gdy operacje są niezależne, oczekiwanie na nie **jedna po drugiej** (`await` w pętli / kolejno) marnuje czas. Uruchomienie ich **równolegle** przez `Future.wait` skraca łączny czas do czasu najdłuższej operacji. Poniżej trzy przykłady z pomiarem czasu.

### Sekwencyjnie — czas sumuje się

```dart
// SEKWENCYJNIE — każde await czeka na poprzednie; czasy się sumują
Future<int> pobierz(int id) async {
  await Future.delayed(Duration(milliseconds: 100));
  return id;
}

void main() async {
  var zegar = Stopwatch()..start();

  var a = await pobierz(1); // ~100 ms
  var b = await pobierz(2); // kolejne ~100 ms
  var c = await pobierz(3); // kolejne ~100 ms

  zegar.stop();
  print('Wyniki: [$a, $b, $c]');
  // Łączny czas ~300 ms (3 × 100 ms)
  print('Sekwencyjnie zajęło >= 300 ms: ${zegar.elapsedMilliseconds >= 300}');
}
// Oczekiwane wyjście:
// Wyniki: [1, 2, 3]
// Sekwencyjnie zajęło >= 300 ms: true
```

### Równolegle — czas najdłuższej operacji

```dart
// RÓWNOLEGLE — Future.wait startuje wszystko naraz; czas ~ najdłuższe zadanie
Future<int> pobierz(int id) async {
  await Future.delayed(Duration(milliseconds: 100));
  return id;
}

void main() async {
  var zegar = Stopwatch()..start();

  // Wszystkie trzy ruszają jednocześnie
  var wyniki = await Future.wait([pobierz(1), pobierz(2), pobierz(3)]);

  zegar.stop();
  print('Wyniki: $wyniki');
  // Łączny czas ~100 ms zamiast ~300 ms
  print('Równolegle zajęło < 300 ms: ${zegar.elapsedMilliseconds < 300}');
}
// Oczekiwane wyjście:
// Wyniki: [1, 2, 3]
// Równolegle zajęło < 300 ms: true
```

### Propagacja błędów w Future.wait

W `Future.wait` błąd dowolnego zadania powoduje, że zwrócony `Future` kończy się błędem — dlatego opakowujemy go w `try/catch`. To trzeci przykład ilustrujący propagację błędów w kodzie asynchronicznym.

```dart
// PROPAGACJA BŁĘDÓW — pojedyncza awaria w Future.wait przerywa całość
Future<int> zadanie(int id) async {
  await Future.delayed(Duration(milliseconds: 10 * id));
  if (id == 2) throw StateError('zadanie $id padło');
  return id;
}

void main() async {
  try {
    await Future.wait([zadanie(1), zadanie(2), zadanie(3)]);
    print('Wszystkie się powiodły');
  } catch (e) {
    // Błąd z zadania 2 propaguje się tutaj
    print('Future.wait zwrócił błąd: $e');
  }
}
// Oczekiwane wyjście:
// Future.wait zwrócił błąd: Bad state: zadanie 2 padło
```

---

## Ćwiczenie 1

### Opis problemu

Napisz **symulację odczytu pliku**. Zaimplementuj asynchroniczną funkcję `Future<String> czytajPlik(String nazwa)`, która symuluje odczyt: czeka 30 ms (`Future.delayed`), a następnie zwraca zawartość w formacie `Zawartość pliku: <nazwa>`. Jeśli `nazwa` jest pusta, funkcja ma rzucić `ArgumentError('Pusta nazwa pliku')`. Napisz funkcję `main`, która przez `try/catch` obsłuży oba przypadki.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `czytajPlik('dane.txt')` | `Zawartość pliku: dane.txt` |
| `czytajPlik('')` | `Błąd: Pusta nazwa pliku` |

### Wskazówki

1. Użyj `await Future.delayed(Duration(milliseconds: 30))`, aby zasymulować opóźnienie odczytu
2. Walidację pustej nazwy wykonaj przed opóźnieniem i rzuć `ArgumentError`
3. Wywołanie w `main` otocz blokiem `try`/`catch`, aby przechwycić błąd asynchroniczny

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
Future<String> czytajPlik(String nazwa) async {
  if (nazwa.isEmpty) {
    throw ArgumentError('Pusta nazwa pliku');
  }
  await Future.delayed(Duration(milliseconds: 30)); // symulacja I/O
  return 'Zawartość pliku: $nazwa';
}

void main() async {
  try {
    print(await czytajPlik('dane.txt'));
  } catch (e) {
    print('Błąd: ${(e as ArgumentError).message}');
  }

  try {
    print(await czytajPlik(''));
  } on ArgumentError catch (e) {
    print('Błąd: ${e.message}');
  }
}
// Wyjście:
// Zawartość pliku: dane.txt
// Błąd: Pusta nazwa pliku
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Zasymuluj **wywołania API ze sztucznym opóźnieniem**, wykonywane **równolegle**. Napisz funkcję `Future<int> pobierzDlugosc(String url)`, która czeka `url.length * 5` milisekund, a potem zwraca `url.length`. Następnie w `main` pobierz długości trzech adresów URL **równolegle** za pomocą `Future.wait` i wypisz sumę wszystkich długości. Zmierz czas i sprawdź, że wykonanie równoległe jest szybsze niż suma opóźnień.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `['ab', 'abc', 'a']` (długości 2, 3, 1) | `Suma długości: 6` |
| `['xxxx', 'yy']` (długości 4, 2) | `Suma długości: 6` |

### Wskazówki

1. Każde wywołanie API to `await Future.delayed(Duration(milliseconds: url.length * 5))`
2. Użyj `Future.wait([...])`, aby uruchomić wszystkie zapytania równolegle
3. Sumę policz np. przez `wyniki.reduce((a, b) => a + b)` lub `fold`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
Future<int> pobierzDlugosc(String url) async {
  await Future.delayed(Duration(milliseconds: url.length * 5)); // symulacja API
  return url.length;
}

Future<int> sumaDlugosci(List<String> adresy) async {
  // Równoległe wywołania — wszystkie startują naraz
  List<int> wyniki = await Future.wait(adresy.map(pobierzDlugosc));
  return wyniki.fold<int>(0, (suma, d) => suma + d);
}

void main() async {
  var zegar = Stopwatch()..start();
  var suma = await sumaDlugosci(['ab', 'abc', 'a']);
  zegar.stop();

  print('Suma długości: $suma'); // 6
  // Równolegle: ~15 ms (najdłuższe), sekwencyjnie byłoby ~30 ms
  print('Szybsze niż sekwencyjnie: ${zegar.elapsedMilliseconds < 30}');
}
// Wyjście:
// Suma długości: 6
// Szybsze niż sekwencyjnie: true
```

</details>

---

## Ćwiczenie 3

### Opis problemu

Zaimplementuj **obsługę limitu czasu** przy użyciu `Future.timeout`. Napisz funkcję `Future<String> zapytanie(int ms)`, która po `ms` milisekundach zwraca `'odpowiedź'`. Następnie napisz `Future<String> zapytanieZLimitem(int ms, int limitMs)`, która wywołuje `zapytanie(ms)` z limitem czasu `limitMs`. Jeśli zapytanie przekroczy limit, funkcja ma zwrócić `'timeout'` zamiast rzucać wyjątek (użyj parametru `onTimeout`).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `zapytanieZLimitem(20, 100)` (mieści się) | `odpowiedź` |
| `zapytanieZLimitem(100, 20)` (przekracza) | `timeout` |

### Wskazówki

1. `future.timeout(Duration(milliseconds: limitMs))` rzuca `TimeoutException` po przekroczeniu czasu
2. Aby zwrócić wartość zamiast wyjątku, podaj `onTimeout: () => 'timeout'`
3. `Future.timeout` jest zdefiniowane w `dart:async`, ale `TimeoutException` i tak nie wystąpi, gdy użyjesz `onTimeout`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:async';

Future<String> zapytanie(int ms) async {
  await Future.delayed(Duration(milliseconds: ms));
  return 'odpowiedź';
}

Future<String> zapytanieZLimitem(int ms, int limitMs) {
  // onTimeout dostarcza wartość zastępczą zamiast rzucać TimeoutException
  return zapytanie(ms).timeout(
    Duration(milliseconds: limitMs),
    onTimeout: () => 'timeout',
  );
}

void main() async {
  print(await zapytanieZLimitem(20, 100));  // mieści się w limicie
  print(await zapytanieZLimitem(100, 20));  // przekracza limit
}
// Wyjście:
// odpowiedź
// timeout
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. Dart działa jednowątkowo, a asynchroniczność realizuje **pętla zdarzeń**, która przetwarza zadania z **kolejki mikrozadań** (wyższy priorytet) i **kolejki zdarzeń** (niższy priorytet), zawsze opróżniając mikrozadania przed pobraniem kolejnego zdarzenia.
2. `scheduleMicrotask` i `Future.microtask` planują zadania w kolejce mikrozadań, natomiast zwykły `Future(...)`, timery i `Future.delayed` trafiają do kolejki zdarzeń — stąd różnica w kolejności wykonania.
3. `async`/`await` pozwala pisać kod asynchroniczny w stylu sekwencyjnym i obsługiwać błędy zwykłym `try`/`catch`/`finally`; funkcja `async` zawsze zwraca `Future`.
4. Metody `then`/`catchError`/`whenComplete` oraz kombinatory `Future.wait`, `Future.any`, `Future.forEach` służą do komponowania wielu operacji asynchronicznych.
5. `Completer` pozwala ręcznie utworzyć i zakończyć `Future` — most między API opartym na callbackach a `async`/`await`; można go zakończyć tylko raz.
6. Najczęstsze pułapki to: nieobsłużone wyjątki asynchroniczne, zapomniany `await` (błędna kolejność efektów ubocznych) oraz zbędne wykonanie sekwencyjne tam, gdzie `Future.wait` daje wykonanie równoległe i wyraźnie krótszy czas.

---

**Następny moduł:** [Streams](02-streams.md)
**Poprzedni moduł:** [Generatory](../04-functions/04-generators.md)
