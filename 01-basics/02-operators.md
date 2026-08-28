---
id: "1.2"
title: "Operatory"
difficulty: "beginner"
section: "01-basics"
prerequisites:
  - "Zmienne i typy danych"
---

# 1.2 Operatory

## Informacje o module

- **Poziom trudności:** beginner
- **Wymagania wstępne:** [Zmienne i typy danych](01-variables-types.md)
- **Cele nauki:**
  1. Znać wszystkie kategorie operatorów w Dart i ich zachowanie
  2. Rozumieć priorytet (precedencję) operatorów i kolejność ich ewaluacji
  3. Stosować operatory kaskadowe (..) i spread (...) do budowania czytelnego kodu

---

## Priorytet operatorów

Dart ewaluuje operatory zgodnie z ich priorytetem — operatory o wyższym priorytecie są obliczane najpierw. Poniższa tabela przedstawia priorytety od najwyższego do najniższego:

| Priorytet | Kategoria | Operatory |
|-----------|-----------|-----------|
| 16 | Jednoargumentowe (postfix) | `e++`, `e--`, `()`, `[]`, `?.`, `.`, `!` |
| 15 | Jednoargumentowe (prefix) | `-e`, `!e`, `~e`, `++e`, `--e`, `await` |
| 14 | Multiplikatywne | `*`, `/`, `~/`, `%` |
| 13 | Addytywne | `+`, `-` |
| 12 | Przesunięcia | `<<`, `>>`, `>>>` |
| 11 | Bitowe AND | `&` |
| 10 | Bitowe XOR | `^` |
| 9 | Bitowe OR | `|` |
| 8 | Relacyjne i testy typów | `>=`, `>`, `<=`, `<`, `as`, `is`, `is!` |
| 7 | Równości | `==`, `!=` |
| 6 | Logiczne AND | `&&` |
| 5 | Logiczne OR | `||` |
| 4 | Null-aware | `??` |
| 3 | Warunkowy | `? :` |
| 2 | Kaskadowy | `..`, `?..` |
| 1 | Przypisania | `=`, `*=`, `/=`, `+=`, `-=`, `&=`, `^=`, etc. |

---

## Operatory arytmetyczne

Operatory arytmetyczne wykonują standardowe operacje matematyczne. Operator `~/` wykonuje dzielenie całkowite (obcinając część ułamkową).

```dart
void main() {
  var a = 10;
  var b = 3;

  print('a + b = ${a + b}');   // 13 — dodawanie
  print('a - b = ${a - b}');   // 7 — odejmowanie
  print('a * b = ${a * b}');   // 30 — mnożenie
  print('a / b = ${a / b}');   // 3.3333... — dzielenie (zwraca double)
  print('a ~/ b = ${a ~/ b}'); // 3 — dzielenie całkowite
  print('a % b = ${a % b}');   // 1 — modulo (reszta z dzielenia)
  print('-a = ${-a}');         // -10 — negacja
}
// Oczekiwane wyjście:
// a + b = 13
// a - b = 7
// a * b = 30
// a / b = 3.3333333333333335
// a ~/ b = 3
// a % b = 1
// -a = -10
```

Operatory inkrementacji i dekrementacji mogą być prefixowe lub postfixowe, co wpływa na zwracaną wartość:

```dart
void main() {
  var x = 5;

  // Postfix — zwraca wartość PRZED zmianą
  print(x++); // 5 (wypisuje 5, potem x staje się 6)
  print(x);   // 6

  // Prefix — zwraca wartość PO zmianie
  print(++x); // 7 (najpierw x staje się 7, potem wypisuje)
  print(x);   // 7

  var y = 10;
  print(--y); // 9 — prefix dekrementacja
  print(y--); // 9 — postfix (wypisuje 9, potem y staje się 8)
  print(y);   // 8
}
// Oczekiwane wyjście:
// 5
// 6
// 7
// 7
// 9
// 9
// 8
```

---

## Operatory równości

Operatory równości porównują wartości obiektów. Operator `==` wywołuje metodę `operator ==` obiektu, a `identical()` sprawdza identyczność referencji.

```dart
void main() {
  var a = 'hello';
  var b = 'hello';
  var c = [1, 2, 3];
  var d = [1, 2, 3];

  // == porównuje wartość (dla typów wbudowanych)
  print(a == b); // true — stringi o tej samej wartości
  print(c == d); // false — listy nie implementują domyślnie == po wartości

  // identical() sprawdza czy to ten sam obiekt w pamięci
  print(identical(a, b)); // true — Dart internuje stałe stringi
  print(identical(c, d)); // false — dwa różne obiekty List

  // Operator !=
  print(1 != 2);   // true
  print('a' != 'a'); // false
}
// Oczekiwane wyjście:
// true
// false
// true
// false
// true
// false
```

Porównanie z null — operator `==` bezpiecznie obsługuje wartości null:

```dart
void main() {
  String? tekst = null;

  // Bezpieczne porównanie z null
  print(tekst == null);  // true
  print(null == null);   // true
  print(tekst != null);  // false

  tekst = 'Dart';
  print(tekst == 'Dart'); // true
  print(tekst == null);   // false
}
// Oczekiwane wyjście:
// true
// true
// false
// true
// false
```

---

## Operatory relacyjne

Operatory relacyjne porównują wielkości i zwracają wartość `bool`. Działają na typach implementujących `Comparable`.

```dart
void main() {
  var x = 10;
  var y = 20;

  print('x < y: ${x < y}');   // true — mniejszy niż
  print('x > y: ${x > y}');   // false — większy niż
  print('x <= 10: ${x <= 10}'); // true — mniejszy lub równy
  print('x >= 10: ${x >= 10}'); // true — większy lub równy
}
// Oczekiwane wyjście:
// x < y: true
// x > y: false
// x <= 10: true
// x >= 10: true
```

Porównywanie stringów — leksykograficznie (według kodów Unicode):

```dart
void main() {
  // Stringi porównywane leksykograficznie
  print('apple' .compareTo('banana')); // -1 (apple < banana)
  print('b' .compareTo('a'));          // 1 (b > a)

  // Operatory relacyjne nie działają bezpośrednio na String
  // print('a' < 'b'); // Błąd kompilacji
  // Zamiast tego używamy compareTo()

  // Ale działają na num
  print(3.14 < 4.0);  // true
  print(100 >= 100);   // true
}
// Oczekiwane wyjście:
// -1
// 1
// true
// true
```

---

## Operatory testów typów (`is`, `is!`)

Operator `is` sprawdza czy obiekt jest danego typu. Operator `is!` jest jego negacją. Po pozytywnym teście `is` Dart automatycznie promuje zmienną do sprawdzonego typu (smart cast).

```dart
void main() {
  Object wartosc = 'Hello Dart';

  // is — sprawdzenie typu
  if (wartosc is String) {
    // Smart cast — wartosc automatycznie promowana do String
    print(wartosc.toUpperCase()); // HELLO DART
    print('Długość: ${wartosc.length}');
  }

  // is! — negacja testu typu
  if (wartosc is! int) {
    print('wartosc nie jest liczbą całkowitą');
  }
}
// Oczekiwane wyjście:
// HELLO DART
// Długość: 10
// wartosc nie jest liczbą całkowitą
```

Test typów w połączeniu z kolekcjami heterogenicznymi:

```dart
void main() {
  var elementy = <Object>[42, 'tekst', 3.14, true, [1, 2]];

  for (var element in elementy) {
    // Wielokrotne testy typów z pattern matching
    if (element is int) {
      print('int: $element (podwojony: ${element * 2})');
    } else if (element is String) {
      print('String: "$element" (długość: ${element.length})');
    } else if (element is double) {
      print('double: $element (zaokrąglony: ${element.round()})');
    } else if (element is List) {
      print('List: $element (rozmiar: ${element.length})');
    } else {
      print('inny: $element');
    }
  }
}
// Oczekiwane wyjście:
// int: 42 (podwojony: 84)
// String: "tekst" (długość: 5)
// double: 3.14 (zaokrąglony: 3)
// inny: true
// List: [1, 2] (rozmiar: 2)
```

---

## Operatory przypisania

Oprócz standardowego `=`, Dart oferuje operatory złożonego przypisania, które łączą operację z przypisaniem. Operator `??=` przypisuje wartość tylko jeśli zmienna jest `null`.

```dart
void main() {
  var x = 10;

  x += 5;   // x = x + 5 → 15
  x -= 3;   // x = x - 3 → 12
  x *= 2;   // x = x * 2 → 24
  x ~/= 5;  // x = x ~/ 5 → 4
  x %= 3;   // x = x % 3 → 1

  print('x = $x'); // 1
}
// Oczekiwane wyjście:
// x = 1
```

Operator `??=` — przypisanie warunkowe na null:

```dart
void main() {
  String? nazwa;

  // ??= przypisuje wartość TYLKO jeśli zmienna jest null
  nazwa ??= 'domyślna';
  print(nazwa); // domyślna

  nazwa ??= 'inna'; // nie przypisze — nazwa już nie jest null
  print(nazwa); // domyślna

  int? liczba = 5;
  liczba ??= 10; // nie przypisze — liczba nie jest null
  print(liczba); // 5
}
// Oczekiwane wyjście:
// domyślna
// domyślna
// 5
```

---

## Operatory logiczne

Operatory logiczne działają na wartościach `bool`. Dart używa ewaluacji skróconej (short-circuit) — jeśli wynik jest znany po lewej stronie, prawa strona nie jest obliczana.

```dart
void main() {
  var prawda = true;
  var falsz = false;

  // && — logiczne AND (oba muszą być true)
  print('true && true: ${prawda && prawda}');   // true
  print('true && false: ${prawda && falsz}');   // false

  // || — logiczne OR (jeden musi być true)
  print('false || true: ${falsz || prawda}');   // true
  print('false || false: ${falsz || falsz}');   // false

  // ! — negacja logiczna
  print('!true: ${!prawda}');  // false
  print('!false: ${!falsz}'); // true
}
// Oczekiwane wyjście:
// true && true: true
// true && false: false
// false || true: true
// false || false: false
// !true: false
// !false: true
```

Ewaluacja skrócona (short-circuit) — prawa strona nie jest obliczana gdy wynik jest już znany:

```dart
bool sprawdz(String nazwa) {
  print('Sprawdzam: $nazwa');
  return true;
}

void main() {
  // Short-circuit z && — jeśli lewa strona jest false, prawa nie jest obliczana
  var wynik1 = false && sprawdz('A'); // 'Sprawdzam: A' NIE zostanie wypisane
  print('wynik1: $wynik1');

  // Short-circuit z || — jeśli lewa strona jest true, prawa nie jest obliczana
  var wynik2 = true || sprawdz('B'); // 'Sprawdzam: B' NIE zostanie wypisane
  print('wynik2: $wynik2');

  // Tu prawa strona BĘDZIE obliczona
  var wynik3 = true && sprawdz('C'); // 'Sprawdzam: C' zostanie wypisane
  print('wynik3: $wynik3');
}
// Oczekiwane wyjście:
// wynik1: false
// wynik2: true
// Sprawdzam: C
// wynik3: true
```

---

## Operatory bitowe

Operatory bitowe działają na poziomie bitów liczb całkowitych. Przydatne przy maskach bitowych, flagach i niskopoziomowej manipulacji danymi.

```dart
void main() {
  var a = 0xF0; // 11110000 w binarnym (240)
  var b = 0x0F; // 00001111 w binarnym (15)

  // & — bitowe AND
  print('a & b = ${(a & b).toRadixString(2).padLeft(8, '0')}'); // 00000000

  // | — bitowe OR
  print('a | b = ${(a | b).toRadixString(2).padLeft(8, '0')}'); // 11111111

  // ^ — bitowe XOR
  print('a ^ b = ${(a ^ b).toRadixString(2).padLeft(8, '0')}'); // 11111111

  // ~ — bitowa negacja (dopełnienie do jedynki)
  print('~a = ${(~a & 0xFF).toRadixString(2).padLeft(8, '0')}'); // 00001111
}
// Oczekiwane wyjście:
// a & b = 00000000
// a | b = 11111111
// a ^ b = 11111111
// ~a = 00001111
```

Operatory przesunięcia bitowego:

```dart
void main() {
  var x = 0x0F; // 00001111 = 15

  // << — przesunięcie w lewo (mnożenie przez potęgę 2)
  print('x << 2 = ${x << 2}');  // 60 (00111100)

  // >> — przesunięcie w prawo (dzielenie przez potęgę 2)
  print('x >> 1 = ${x >> 1}');  // 7 (00000111)

  // >>> — przesunięcie w prawo bez znaku (unsigned shift)
  var neg = -1;
  print('neg >>> 28 = ${neg >>> 28}'); // 15 (zeruje bity znaku)

  // Praktyczne użycie — flagi bitowe
  const read = 1 << 0;    // 001 = 1
  const write = 1 << 1;   // 010 = 2
  const execute = 1 << 2; // 100 = 4

  var permissions = read | write; // 011 = 3
  print('Ma odczyt: ${(permissions & read) != 0}');     // true
  print('Ma wykonanie: ${(permissions & execute) != 0}'); // false
}
// Oczekiwane wyjście:
// x << 2 = 60
// x >> 1 = 7
// neg >>> 28 = 15
// Ma odczyt: true
// Ma wykonanie: false
```

---

## Operator warunkowy (ternary)

Operator `? :` to skrócona forma `if-else`, która zwraca jedną z dwóch wartości na podstawie warunku.

```dart
void main() {
  var wiek = 20;

  // warunek ? wartość_jeśli_true : wartość_jeśli_false
  var status = wiek >= 18 ? 'dorosły' : 'niepełnoletni';
  print(status); // dorosły

  // Zagnieżdżone ternary (lepiej unikać dla czytelności)
  var ocena = 85;
  var stopien = ocena >= 90
      ? 'celujący'
      : ocena >= 75
          ? 'dobry'
          : 'dostateczny';
  print('Stopień: $stopien'); // dobry
}
// Oczekiwane wyjście:
// dorosły
// Stopień: dobry
```

Operator warunkowy w wyrażeniach i argumentach:

```dart
void main() {
  var lista = [1, 2, 3, 4, 5];

  // Użycie w interpolacji
  print('Lista jest ${lista.isEmpty ? "pusta" : "niepusta"}');

  // Użycie jako argument funkcji
  var posortowana = List.of(lista)..sort((a, b) => a > b ? -1 : 1);
  print('Malejąco: $posortowana');
}
// Oczekiwane wyjście:
// Lista jest niepusta
// Malejąco: [5, 4, 3, 2, 1]
```

---

## Operator kaskadowy (`..` i `?..`)

Operator kaskadowy pozwala wykonać serię operacji na tym samym obiekcie bez powtarzania referencji. Zwraca obiekt, na którym operuje, umożliwiając łańcuchowe wywołania.

```dart
void main() {
  // Bez kaskady — powtarzanie referencji
  var lista1 = <int>[];
  lista1.add(1);
  lista1.add(2);
  lista1.add(3);

  // Z kaskadą — bardziej zwięzły zapis
  var lista2 = <int>[]
    ..add(10)   // .. zwraca ten sam obiekt (listę)
    ..add(20)
    ..add(30)
    ..sort();   // sortuje in-place

  print('lista1: $lista1');
  print('lista2: $lista2');
}
// Oczekiwane wyjście:
// lista1: [1, 2, 3]
// lista2: [10, 20, 30]
```

Kaskada jest szczególnie przydatna przy konfiguracji obiektów i wzorcu builder:

```dart
class Zapytanie {
  String url = '';
  String metoda = 'GET';
  Map<String, String> naglowki = {};
  String? body;

  @override
  String toString() => '$metoda $url headers=$naglowki body=$body';
}

void main() {
  // Kaskada do konfiguracji obiektu
  var request = Zapytanie()
    ..url = 'https://api.example.com/users'
    ..metoda = 'POST'
    ..naglowki['Content-Type'] = 'application/json'
    ..naglowki['Authorization'] = 'Bearer token123'
    ..body = '{"name": "Jan"}';

  print(request);

  // ?.. — null-aware cascade (bezpieczna gdy obiekt może być null)
  Zapytanie? mozeNull;
  mozeNull
    ?..url = 'test' // nie wykona się gdy mozeNull jest null
    ..metoda = 'DELETE';

  print('mozeNull: $mozeNull'); // null
}
// Oczekiwane wyjście:
// POST https://api.example.com/users headers={Content-Type: application/json, Authorization: Bearer token123} body={"name": "Jan"}
// mozeNull: null
```

---

## Operator spread (`...` i `...?`)

Operator spread „rozprasza" elementy kolekcji do innej kolekcji. `...?` to wariant null-aware, który pomija kolekcję jeśli jest null.

```dart
void main() {
  var bazowe = [1, 2, 3];
  var dodatkowe = [4, 5, 6];

  // ... wstawia wszystkie elementy jednej kolekcji do drugiej
  var polaczona = [...bazowe, ...dodatkowe];
  print('Połączona: $polaczona'); // [1, 2, 3, 4, 5, 6]

  // Spread w Set
  var set1 = {1, 2, 3};
  var set2 = {3, 4, 5};
  var polaczonySet = {...set1, ...set2};
  print('Set: $polaczonySet'); // {1, 2, 3, 4, 5}

  // Spread w Map
  var defaults = {'color': 'blue', 'size': 'medium'};
  var custom = {'color': 'red', 'weight': 'bold'};
  var merged = {...defaults, ...custom}; // późniejsze wartości nadpisują wcześniejsze
  print('Map: $merged');
}
// Oczekiwane wyjście:
// Połączona: [1, 2, 3, 4, 5, 6]
// Set: {1, 2, 3, 4, 5}
// Map: {color: red, size: medium, weight: bold}
```

Null-aware spread `...?` bezpiecznie obsługuje nullable kolekcje:

```dart
void main() {
  List<int>? opcjonalna = null;
  var pewna = [1, 2, 3];

  // ...? pomija kolekcję null zamiast rzucać błąd
  var wynik = [...pewna, ...?opcjonalna];
  print('Wynik: $wynik'); // [1, 2, 3]

  opcjonalna = [4, 5];
  var wynik2 = [...pewna, ...?opcjonalna];
  print('Wynik2: $wynik2'); // [1, 2, 3, 4, 5]

  // Spread w połączeniu z collection if
  var debug = true;
  var config = [
    'prod_setting',
    if (debug) ...['debug_a', 'debug_b'], // spread wewnątrz collection if
  ];
  print('Config: $config');
}
// Oczekiwane wyjście:
// Wynik: [1, 2, 3]
// Wynik2: [1, 2, 3, 4, 5]
// Config: [prod_setting, debug_a, debug_b]
```

---

## Priorytet w praktyce

Zrozumienie priorytetu operatorów pozwala unikać błędów logicznych i pisać czytelny kod. W razie wątpliwości — używaj nawiasów.

Poniższy przykład demonstruje jak priorytet wpływa na wynik wyrażeń:

```dart
void main() {
  // Mnożenie ma wyższy priorytet niż dodawanie
  var wynik1 = 2 + 3 * 4;    // 2 + (3 * 4) = 14, NIE (2 + 3) * 4 = 20
  print('2 + 3 * 4 = $wynik1');

  // Logiczne AND ma wyższy priorytet niż OR
  var wynik2 = true || false && false; // true || (false && false) = true
  print('true || false && false = $wynik2');

  // Relacyjne mają wyższy priorytet niż równości
  var x = 5;
  var wynik3 = x == 5 && x > 3; // (x == 5) && (x > 3) = true
  print('x == 5 && x > 3 = $wynik3');

  // ?? ma niższy priorytet niż porównania
  String? s = null;
  var wynik4 = s ?? 'domyślna'; // null ?? 'domyślna' = 'domyślna'
  print('s ?? "domyślna" = $wynik4');
}
// Oczekiwane wyjście:
// 2 + 3 * 4 = 14
// true || false && false = true
// x == 5 && x > 3 = true
// s ?? "domyślna" = domyślna
```

Nawiasy wymuszają inną kolejność obliczania:

```dart
void main() {
  // Bez nawiasów: priorytet decyduje
  print(2 + 3 * 4);     // 14
  print((2 + 3) * 4);   // 20 — nawiasy wymuszają dodawanie najpierw

  // Złożone wyrażenia — nawiasy poprawiają czytelność
  var a = 10;
  var b = 3;
  var c = 7;

  // Bez nawiasów: a > b && b < c || a == c
  // Z priorytetem: ((a > b) && (b < c)) || (a == c)
  var bezNawiasow = a > b && b < c || a == c;

  // Z nawiasami: jawna intencja
  var zNawiasami = (a > b) && (b < c || a == c);

  print('Bez nawiasów: $bezNawiasow'); // true
  print('Z nawiasami: $zNawiasami');   // true (ale inna logika!)
}
// Oczekiwane wyjście:
// 14
// 20
// Bez nawiasów: true
// Z nawiasami: true
```

---

## Ćwiczenie 1

### Opis problemu

Napisz program, który implementuje prostą kalkulator uprawnień pliku w stylu Unix. Użyj operatorów bitowych do ustawiania, sprawdzania i odbierania uprawnień. Uprawnienia to: read (4), write (2), execute (1). Program powinien wypisać stan uprawnień po serii operacji.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| Nadaj: read, write. Sprawdź: read, execute. Odbierz: write. | `Uprawnienia: rw- (6)\nMa read: true\nMa execute: false\nPo odebraniu write: r-- (4)` |
| Nadaj: read, write, execute. Sprawdź: write. Odbierz: read, execute. | `Uprawnienia: rwx (7)\nMa write: true\nPo odebraniu read, execute: -w- (2)` |

### Wskazówki

1. Zdefiniuj stałe: `read = 1 << 2` (4), `write = 1 << 1` (2), `execute = 1 << 0` (1)
2. Nadawanie uprawnienia: `permissions |= read` (bitowe OR)
3. Sprawdzanie uprawnienia: `(permissions & read) != 0`
4. Odbieranie uprawnienia: `permissions &= ~write` (AND z zanegowaną maską)

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  const read = 1 << 2;    // 4 = 100
  const write = 1 << 1;   // 2 = 010
  const execute = 1 << 0; // 1 = 001

  // Funkcja formatująca uprawnienia jako rwx
  String formatPermissions(int perm) {
    var r = (perm & read) != 0 ? 'r' : '-';
    var w = (perm & write) != 0 ? 'w' : '-';
    var x = (perm & execute) != 0 ? 'x' : '-';
    return '$r$w$x ($perm)';
  }

  // Nadanie uprawnień
  var permissions = 0;
  permissions |= read;
  permissions |= write;
  print('Uprawnienia: ${formatPermissions(permissions)}');

  // Sprawdzenie uprawnień
  print('Ma read: ${(permissions & read) != 0}');
  print('Ma execute: ${(permissions & execute) != 0}');

  // Odebranie uprawnienia
  permissions &= ~write;
  print('Po odebraniu write: ${formatPermissions(permissions)}');
}
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Napisz program wykorzystujący operator kaskadowy (`..`) i spread (`...`) do zbudowania konfiguracji aplikacji. Stwórz klasę `AppConfig` z polami: `name`, `version`, `features` (List<String>), `settings` (Map<String, String>). Użyj kaskady do zbudowania obiektu i spread do złączenia domyślnych i niestandardowych ustawień.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| name: `'MyApp'`, version: `'2.0'`, defaultFeatures: `['auth', 'logging']`, extraFeatures: `['analytics']`, defaultSettings: `{'theme': 'dark'}`, customSettings: `{'lang': 'pl'}` | `MyApp v2.0\nFeatures: [auth, logging, analytics]\nSettings: {theme: dark, lang: pl}` |
| name: `'TestApp'`, version: `'1.0'`, defaultFeatures: `['core']`, extraFeatures: `null`, defaultSettings: `{'env': 'dev'}`, customSettings: `{'debug': 'true'}` | `TestApp v1.0\nFeatures: [core]\nSettings: {env: dev, debug: true}` |

### Wskazówki

1. Użyj `..` do ustawienia wielu pól obiektu w jednym wyrażeniu
2. Użyj `...` do połączenia list features
3. Użyj `...?` dla opcjonalnych (nullable) kolekcji features
4. Użyj `...` na mapach do połączenia ustawień (późniejsze wpisy nadpiszą wcześniejsze)

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class AppConfig {
  String name = '';
  String version = '';
  List<String> features = [];
  Map<String, String> settings = {};

  @override
  String toString() =>
      '$name v$version\nFeatures: $features\nSettings: $settings';
}

void main() {
  var defaultFeatures = ['auth', 'logging'];
  List<String>? extraFeatures = ['analytics'];
  var defaultSettings = {'theme': 'dark'};
  var customSettings = {'lang': 'pl'};

  // Kaskada do budowy obiektu + spread do łączenia kolekcji
  var config = AppConfig()
    ..name = 'MyApp'
    ..version = '2.0'
    ..features = [...defaultFeatures, ...?extraFeatures]
    ..settings = {...defaultSettings, ...customSettings};

  print(config);
}
```

</details>

---

## Ćwiczenie 3

### Opis problemu

Napisz funkcję `ocenWynik(int punkty)` używającą operatora warunkowego (ternary) do klasyfikacji wyników testu. Zasady: >= 90 → „celujący", >= 75 → „dobry", >= 60 → „dostateczny", >= 50 → „dopuszczający", < 50 → „niedostateczny". Dodatkowo użyj operatorów logicznych aby sprawdzić czy wynik mieści się w poprawnym zakresie (0-100).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `punkty = 92` | `92 pkt → celujący` |
| `punkty = 45` | `45 pkt → niedostateczny` |
| `punkty = -5` | `Błąd: wynik poza zakresem 0-100` |

### Wskazówki

1. Najpierw sprawdź poprawność zakresu z `&&`: `punkty >= 0 && punkty <= 100`
2. Zagnieżdżony operator ternary: `warunek1 ? a : warunek2 ? b : c`
3. Choć zagnieżdżone ternary działają, rozważ czytelność — w prawdziwym kodzie `switch` byłby lepszy

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String ocenWynik(int punkty) {
  // Walidacja zakresu z operatorami logicznymi
  if (!(punkty >= 0 && punkty <= 100)) {
    return 'Błąd: wynik poza zakresem 0-100';
  }

  // Zagnieżdżony operator warunkowy
  var ocena = punkty >= 90
      ? 'celujący'
      : punkty >= 75
          ? 'dobry'
          : punkty >= 60
              ? 'dostateczny'
              : punkty >= 50
                  ? 'dopuszczający'
                  : 'niedostateczny';

  return '$punkty pkt → $ocena';
}

void main() {
  print(ocenWynik(92));
  print(ocenWynik(45));
  print(ocenWynik(-5));
}
```

</details>

---

**Następny moduł:** [Null safety](03-null-safety.md)
**Poprzedni moduł:** [Zmienne i typy danych](01-variables-types.md)
