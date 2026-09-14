---
id: "4.4"
title: "Generatory (sync* i async*)"
difficulty: "intermediate"
section: "04-functions"
prerequisites:
  - "Deklaracje funkcji i parametry"
  - "Closures i funkcje wyższego rzędu"
---

# 4.4 Generatory (sync* i async*)

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Deklaracje funkcji i parametry](../04-functions/01-declarations-params.md), [Closures i funkcje wyższego rzędu](../04-functions/03-closures-hof.md)
- **Cele nauki:**
  1. Rozumieć różnicę między generatorami synchronicznymi (`sync*`) a asynchronicznymi (`async*`) i wiedzieć, kiedy stosować każdy z nich
  2. Stosować instrukcje `yield` i `yield*` do emitowania pojedynczych wartości oraz delegowania do innych generatorów
  3. Projektować leniwe sekwencje (lazy sequences) wykorzystujące generatory do efektywnego przetwarzania dużych zbiorów danych
  4. Tworzyć asynchroniczne strumienie danych za pomocą generatorów `async*`

---

## Generatory synchroniczne (`sync*`)

Generator synchroniczny to funkcja oznaczona modyfikatorem `sync*`, która zwraca `Iterable<T>`. Zamiast obliczać i zwracać całą kolekcję naraz, generator produkuje wartości **leniwie** — kolejna wartość jest obliczana dopiero gdy zostanie zażądana przez iterację.

### Instrukcja `yield`

Instrukcja `yield` emituje pojedynczą wartość z generatora. Wykonanie funkcji jest wstrzymywane po każdym `yield` i wznawiane dopiero gdy konsument zażąda kolejnej wartości.

Poniższy przykład demonstruje podstawowy generator synchroniczny emitujący kolejne liczby naturalne do podanego limitu.

```dart
// Generator sync* zwraca Iterable<int> — wartości produkowane leniwie
Iterable<int> liczbyDo(int max) sync* {
  for (var i = 1; i <= max; i++) {
    yield i; // emituje kolejną wartość i wstrzymuje wykonanie
  }
}

void main() {
  // Wartości są generowane dopiero przy iteracji
  final liczby = liczbyDo(5);
  print(liczby); // Wyświetla instancję Iterable, nie listę
  print(liczby.toList()); // Wymusza ewaluację wszystkich wartości
}
// Oczekiwane wyjście:
// (1, 2, 3, 4, 5)
// [1, 2, 3, 4, 5]
```

Kolejny przykład pokazuje generator produkujący ciąg Fibonacciego — klasyczny przypadek użycia leniwej sekwencji, gdzie obliczamy tylko tyle wartości, ile potrzebujemy.

```dart
// Generator nieskończonego ciągu Fibonacciego
Iterable<int> fibonacci() sync* {
  var a = 0, b = 1;
  while (true) {
    yield a; // emituje bieżącą wartość
    final temp = a + b;
    a = b;
    b = temp;
  }
}

void main() {
  // take() pobiera tylko 10 pierwszych wartości — reszta nigdy nie jest obliczana
  final pierwszeDziesiec = fibonacci().take(10).toList();
  print(pierwszeDziesiec);

  // Wykorzystanie where() do filtrowania — leniwa ewaluacja
  final parzyste = fibonacci().where((n) => n % 2 == 0).take(5).toList();
  print('Parzyste Fibonacci: $parzyste');
}
// Oczekiwane wyjście:
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
// Parzyste Fibonacci: [0, 2, 8, 34, 144]
```

### Instrukcja `yield*` (delegacja)

Instrukcja `yield*` deleguje emisję do innego `Iterable`. Zamiast ręcznie iterować po kolekcji i emitować każdy element osobno, `yield*` przekazuje kontrolę do wskazanego iterable.

Poniższy przykład demonstruje kompozycję generatorów za pomocą `yield*` — łączenie wielu źródeł danych w jedną sekwencję.

```dart
// Generator emitujący zakres liczb
Iterable<int> zakres(int start, int end) sync* {
  for (var i = start; i <= end; i++) {
    yield i;
  }
}

// Generator kompozytowy delegujący do innych generatorów za pomocą yield*
Iterable<int> polaczoneSekwencje() sync* {
  yield* zakres(1, 3);   // deleguje do generatora zakres(1, 3)
  yield 100;              // emituje pojedynczą wartość między delegacjami
  yield* zakres(10, 12); // deleguje do kolejnego generatora
}

void main() {
  print(polaczoneSekwencje().toList());
}
// Oczekiwane wyjście:
// [1, 2, 3, 100, 10, 11, 12]
```

Następny przykład pokazuje rekurencyjny generator — `yield*` umożliwia eleganckie implementacje algorytmów rekurencyjnych produkujących sekwencje wartości.

```dart
// Rekurencyjny generator przechodzenia drzewa (in-order traversal)
class Wezel {
  final int wartosc;
  final Wezel? lewy;
  final Wezel? prawy;

  Wezel(this.wartosc, {this.lewy, this.prawy});
}

// yield* rekurencyjnie deleguje do podgeneratorów dla poddrzew
Iterable<int> przejdzDrzewo(Wezel? wezel) sync* {
  if (wezel == null) return;
  yield* przejdzDrzewo(wezel.lewy);  // delegacja do lewego poddrzewa
  yield wezel.wartosc;                 // emituje wartość bieżącego węzła
  yield* przejdzDrzewo(wezel.prawy); // delegacja do prawego poddrzewa
}

void main() {
  final drzewo = Wezel(4,
    lewy: Wezel(2, lewy: Wezel(1), prawy: Wezel(3)),
    prawy: Wezel(6, lewy: Wezel(5), prawy: Wezel(7)),
  );

  print(przejdzDrzewo(drzewo).toList());
}
// Oczekiwane wyjście:
// [1, 2, 3, 4, 5, 6, 7]
```

---

## Generatory asynchroniczne (`async*`)

Generator asynchroniczny to funkcja oznaczona modyfikatorem `async*`, która zwraca `Stream<T>`. Działa analogicznie do generatora synchronicznego, ale emitowane wartości mogą być produkowane asynchronicznie — generator może czekać na operacje I/O, opóźnienia czy inne zdarzenia asynchroniczne między kolejnymi emisjami.

### Instrukcja `yield` w kontekście `async*`

W generatorze `async*` instrukcja `yield` emituje wartość do strumienia. Między kolejnymi emisjami generator może wykonywać operacje asynchroniczne za pomocą `await`.

Poniższy przykład demonstruje generator asynchroniczny symulujący odczyt danych z zewnętrznego źródła z opóźnieniem między elementami.

```dart
// Generator async* zwraca Stream<String> — wartości emitowane asynchronicznie
Stream<String> pobierzDane(List<String> adresy) async* {
  for (final adres in adresy) {
    // await symuluje opóźnienie sieciowe
    await Future.delayed(Duration(milliseconds: 100));
    yield 'Dane z: $adres'; // emituje wynik do strumienia
  }
}

void main() async {
  final adresy = ['api/users', 'api/posts', 'api/comments'];

  // await for konsumuje strumień element po elemencie
  await for (final wynik in pobierzDane(adresy)) {
    print(wynik);
  }
}
// Oczekiwane wyjście:
// Dane z: api/users
// Dane z: api/posts
// Dane z: api/comments
```

Kolejny przykład pokazuje generator asynchroniczny produkujący nieskończony strumień zdarzeń — typowy wzorzec dla polling lub monitorowania zasobów.

```dart
// Generator produkujący strumień odczytów temperatury co sekundę
Stream<double> monitorTemperatury() async* {
  var bazowa = 20.0;
  var i = 0;
  while (true) {
    await Future.delayed(Duration(seconds: 1));
    // Symulacja fluktuacji temperatury
    final odczyt = bazowa + (i % 5) * 0.3 - 0.6;
    yield odczyt; // emituje odczyt do strumienia
    i++;
  }
}

void main() async {
  // take(3) ogranicza strumień do 3 elementów
  await for (final temp in monitorTemperatury().take(3)) {
    print('Temperatura: ${temp.toStringAsFixed(1)}°C');
  }
  print('Monitoring zakończony');
}
// Oczekiwane wyjście:
// Temperatura: 19.4°C
// Temperatura: 19.7°C
// Temperatura: 20.0°C
// Monitoring zakończony
```

### Instrukcja `yield*` w kontekście `async*`

W generatorze `async*` instrukcja `yield*` deleguje do innego `Stream`. Wszystkie zdarzenia z delegowanego strumienia są przekazywane do strumienia wyjściowego.

Poniższy przykład demonstruje łączenie wielu strumieni asynchronicznych za pomocą `yield*` w sekwencji.

```dart
// Strumień emitujący liczby z opóźnieniem
Stream<int> odliczanie(int od, int do_) async* {
  for (var i = od; i <= do_; i++) {
    await Future.delayed(Duration(milliseconds: 50));
    yield i;
  }
}

// Generator łączący wiele strumieni sekwencyjnie za pomocą yield*
Stream<int> polaczoneStrumienie() async* {
  yield* odliczanie(1, 3);   // deleguje do pierwszego strumienia
  yield 0;                     // emituje separator
  yield* odliczanie(10, 12); // deleguje do drugiego strumienia
}

void main() async {
  final wyniki = await polaczoneStrumienie().toList();
  print(wyniki);
}
// Oczekiwane wyjście:
// [1, 2, 3, 0, 10, 11, 12]
```

Następny przykład pokazuje generator `async*` z obsługą błędów — generator może przechwytywać wyjątki i emitować wartości zastępcze.

```dart
// Symulacja API, które może rzucić wyjątek
Future<String> pobierzZasob(String id) async {
  await Future.delayed(Duration(milliseconds: 50));
  if (id == 'error') throw Exception('Zasób niedostępny: $id');
  return 'Zasób[$id]';
}

// Generator async* z obsługą błędów — emituje wyniki lub komunikaty o błędach
Stream<String> pobierzWszystkie(List<String> identyfikatory) async* {
  for (final id in identyfikatory) {
    try {
      final wynik = await pobierzZasob(id);
      yield wynik; // emituje pomyślny wynik
    } catch (e) {
      yield 'BŁĄD: $e'; // emituje informację o błędzie zamiast przerywać strumień
    }
  }
}

void main() async {
  await for (final wynik in pobierzWszystkie(['a', 'error', 'b'])) {
    print(wynik);
  }
}
// Oczekiwane wyjście:
// Zasób[a]
// BŁĄD: Exception: Zasób niedostępny: error
// Zasób[b]
```

---

## Praktyczne zastosowania leniwej ewaluacji

Generatory są szczególnie przydatne w sytuacjach, gdy:

1. **Dane są zbyt duże, by zmieścić się w pamięci** — generator produkuje elementy na żądanie, bez konieczności przechowywania całej kolekcji.
2. **Sekwencja jest potencjalnie nieskończona** — jak ciąg Fibonacciego czy strumień zdarzeń.
3. **Obliczenia są kosztowne** — leniwa ewaluacja pozwala obliczać tylko tyle, ile potrzeba.
4. **Przetwarzanie potokowe (pipeline)** — generatory można łączyć w potoki transformacji danych.

### Porównanie: eager vs lazy

Poniższy przykład ilustruje różnicę między podejściem zachłannym (eager) a leniwym (lazy) w kontekście przetwarzania dużego zbioru danych.

```dart
// Podejście EAGER — tworzy pełne listy pośrednie w pamięci
List<int> eagerPrzetwarzanie(int n) {
  final wszystkie = List.generate(n, (i) => i); // alokuje listę n elementów
  final przefiltrowane = wszystkie.where((x) => x % 3 == 0).toList(); // nowa lista
  final zmapowane = przefiltrowane.map((x) => x * x).toList(); // kolejna lista
  return zmapowane.take(5).toList();
}

// Podejście LAZY z generatorem — minimalne użycie pamięci
Iterable<int> lazyPrzetwarzanie(int n) sync* {
  var znalezione = 0;
  for (var i = 0; i < n && znalezione < 5; i++) {
    if (i % 3 == 0) {
      yield i * i; // emituje tylko potrzebne wartości
      znalezione++;
    }
  }
}

void main() {
  // Eager: tworzy listę 1000000 elementów, filtruje, mapuje — duże zużycie pamięci
  final eager = eagerPrzetwarzanie(1000000);
  print('Eager: $eager');

  // Lazy: iteruje tylko do znalezienia 5 elementów — minimalna pamięć
  final lazy = lazyPrzetwarzanie(1000000).toList();
  print('Lazy: $lazy');
}
// Oczekiwane wyjście:
// Eager: [0, 9, 36, 81, 144]
// Lazy: [0, 9, 36, 81, 144]
```

### Generatory jako potoki transformacji

Generatory można komponować w potoki, gdzie wyjście jednego generatora jest wejściem następnego — tworząc czytelne i wydajne łańcuchy przetwarzania.

```dart
// Etap 1: generator źródłowy — produkuje dane wejściowe
Iterable<int> zrodlo(int n) sync* {
  for (var i = 0; i < n; i++) {
    yield i;
  }
}

// Etap 2: filtr — przepuszcza tylko elementy spełniające warunek
Iterable<int> filtruj(Iterable<int> wejscie, bool Function(int) predykat) sync* {
  for (final element in wejscie) {
    if (predykat(element)) {
      yield element;
    }
  }
}

// Etap 3: transformacja — mapuje elementy na nowe wartości
Iterable<String> transformuj(Iterable<int> wejscie) sync* {
  for (final element in wejscie) {
    yield 'wartość: ${element * 2}'; // transformacja z formatowaniem
  }
}

void main() {
  // Potok: źródło → filtr → transformacja → ograniczenie
  final wynik = transformuj(
    filtruj(zrodlo(100), (n) => n.isOdd),
  ).take(4).toList();

  print(wynik);
}
// Oczekiwane wyjście:
// [wartość: 2, wartość: 6, wartość: 10, wartość: 14]
```

---

## Ćwiczenie: Implementacja leniwego generatora sekwencji

### Opis problemu

Zaimplementuj generator synchroniczny `potegi(int podstawa)`, który produkuje **nieskończoną** leniwą sekwencję kolejnych potęg danej podstawy: podstawa^0, podstawa^1, podstawa^2, podstawa^3, ...

Następnie napisz generator `zakresPotek(int podstawa, int minWartosc, int maxWartosc)`, który używa `yield*` lub filtrowania, aby emitować tylko te potęgi, które mieszczą się w zadanym zakresie [minWartosc, maxWartosc].

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `potegi(2).take(6).toList()` | `[1, 2, 4, 8, 16, 32]` |
| `potegi(3).take(4).toList()` | `[1, 3, 9, 27]` |
| `zakresPotek(2, 4, 50).toList()` | `[4, 8, 16, 32]` |
| `zakresPotek(3, 10, 100).toList()` | `[27, 81]` |

### Wskazówki

1. Użyj `sync*` i pętli `while (true)` dla nieskończonej sekwencji — leniwa ewaluacja gwarantuje, że generator nie zapętli się w nieskończoność
2. Do obliczania potęg możesz użyć `dart:math` (`pow()`) lub mnożenia iteracyjnego
3. W `zakresPotek` pamiętaj, że potęgi rosną monotonicznie — gdy wartość przekroczy `maxWartosc`, możesz przerwać generowanie (użyj `return` lub `break`)
4. `yield*` pozwala delegować do innego generatora — rozważ użycie generatora `potegi` wewnątrz `zakresPotek`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
/// Generator nieskończonej sekwencji potęg danej podstawy.
/// Produkuje: podstawa^0, podstawa^1, podstawa^2, ...
Iterable<int> potegi(int podstawa) sync* {
  var wynik = 1;
  while (true) {
    yield wynik;
    wynik *= podstawa;
  }
}

/// Generator emitujący potęgi podstawy mieszczące się w zakresie [minWartosc, maxWartosc].
/// Wykorzystuje fakt, że potęgi rosną monotonicznie (dla podstawa > 1),
/// więc przerywa generowanie gdy wartość przekroczy maxWartosc.
Iterable<int> zakresPotek(int podstawa, int minWartosc, int maxWartosc) sync* {
  for (final p in potegi(podstawa)) {
    if (p > maxWartosc) break;   // potęgi rosną — nie ma sensu kontynuować
    if (p >= minWartosc) yield p; // emituje tylko wartości w zakresie
  }
}

void main() {
  print(potegi(2).take(6).toList());
  print(potegi(3).take(4).toList());
  print(zakresPotek(2, 4, 50).toList());
  print(zakresPotek(3, 10, 100).toList());
}
// Oczekiwane wyjście:
// [1, 2, 4, 8, 16, 32]
// [1, 3, 9, 27]
// [4, 8, 16, 32]
// [27, 81]
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`sync*` generuje `Iterable<T>`** — wartości są produkowane leniwie, na żądanie konsumenta, co minimalizuje zużycie pamięci
2. **`async*` generuje `Stream<T>`** — umożliwia asynchroniczną produkcję wartości z użyciem `await` między emisjami
3. **`yield` emituje pojedynczą wartość** — wstrzymuje wykonanie generatora do momentu zażądania kolejnej wartości
4. **`yield*` deleguje do innego iterable/strumienia** — pozwala na kompozycję i rekurencyjne generatory
5. **Leniwa ewaluacja** jest kluczową zaletą generatorów — umożliwia pracę z nieskończonymi sekwencjami i minimalizuje alokacje pamięci
6. **Generatory jako potoki** — łączenie generatorów tworzy czytelne i wydajne łańcuchy przetwarzania danych

---

**Poprzedni moduł:** [Closures i funkcje wyższego rzędu](../04-functions/03-closures-hof.md)
**Następny moduł:** [Klasy i konstruktory](../05-oop/01-classes-constructors.md)
