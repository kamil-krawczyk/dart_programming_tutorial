---
id: "4.2"
title: "Lambdy i typedef"
difficulty: "intermediate"
section: "04-functions"
prerequisites:
  - "Deklaracje funkcji i parametry"
---

# 4.2 Lambdy i typedef

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Deklaracje funkcji i parametry](01-declarations-params.md)
- **Cele nauki:**
  1. Tworzyć i stosować funkcje anonimowe (lambdy) w różnych kontekstach
  2. Definiować aliasy typów funkcji za pomocą `typedef` i stosować je w deklaracjach
  3. Rozumieć typ `Function` i jego rolę w systemie typów Dart

---

## Funkcje anonimowe (lambdy)

Funkcja anonimowa (lambda) to funkcja bez nazwy, którą można przypisać do zmiennej, przekazać jako argument lub zwrócić z innej funkcji. W Dart istnieją dwie formy zapisu: blokowa i strzałkowa.

### Składnia blokowa

Pełna forma funkcji anonimowej używa nawiasów klamrowych i może zawierać wiele instrukcji:

```dart
void main() {
  // Funkcja anonimowa przypisana do zmiennej
  var powitaj = (String imie) {
    var wielkie = imie.toUpperCase();
    return 'Cześć, $wielkie!';
  };

  print(powitaj('Anna'));  // wywołanie lambdy
  print(powitaj('Jan'));
}
// Oczekiwane wyjście:
// Cześć, ANNA!
// Cześć, JAN!
```

Lambdy blokowe są przydatne gdy logika wymaga kilku kroków:

```dart
void main() {
  var liczby = [5, 2, 8, 1, 9, 3];

  // Lambda jako argument metody sort — porównuje elementy malejąco
  liczby.sort((a, b) {
    // Odwrócona kolejność dla sortowania malejącego
    return b.compareTo(a);
  });

  print(liczby);
}
// Oczekiwane wyjście:
// [9, 8, 5, 3, 2, 1]
```

### Składnia strzałkowa (arrow syntax)

Gdy funkcja anonimowa zawiera tylko jedno wyrażenie, można użyć zapisu strzałkowego `=>`:

```dart
void main() {
  // Strzałkowa lambda — zwraca wynik wyrażenia
  var kwadrat = (int n) => n * n;
  var czyParzysta = (int n) => n % 2 == 0;

  print(kwadrat(5));        // 25
  print(czyParzysta(4));    // true
  print(czyParzysta(7));    // false
}
// Oczekiwane wyjście:
// 25
// true
// false
```

Strzałkowe lambdy są szczególnie czytelne w połączeniu z metodami kolekcji:

```dart
void main() {
  var produkty = ['jabłko', 'banan', 'czereśnia', 'daktyl'];

  // Łańcuch operacji z lambdami strzałkowymi
  var wynik = produkty
      .where((p) => p.length > 5)       // filtrowanie: dłuższe niż 5 znaków
      .map((p) => p.toUpperCase())       // transformacja: wielkie litery
      .toList();

  print(wynik);
}
// Oczekiwane wyjście:
// [JABŁKO, CZEREŚNIA, DAKTYL]
```

### Lambdy bez parametrów

Funkcja anonimowa może nie przyjmować żadnych parametrów — używamy wtedy pustych nawiasów:

```dart
void main() {
  // Lambda bez parametrów
  var losujPowitanie = () {
    var powitania = ['Hej!', 'Siema!', 'Witaj!'];
    return powitania[1]; // dla deterministycznego przykładu
  };

  // Lambda strzałkowa bez parametrów
  var aktualnyRok = () => 2024;

  print(losujPowitanie());
  print('Rok: ${aktualnyRok()}');
}
// Oczekiwane wyjście:
// Siema!
// Rok: 2024
```

Lambdy bez parametrów są często używane jako callbacki:

```dart
void wykonajPozniej(void Function() akcja) {
  print('Przed akcją...');
  akcja(); // wywołanie przekazanej lambdy
  print('Po akcji.');
}

void main() {
  wykonajPozniej(() {
    print('Wykonuję zadanie!');
  });
}
// Oczekiwane wyjście:
// Przed akcją...
// Wykonuję zadanie!
// Po akcji.
```

---

## `typedef` — aliasy typów funkcji

Słowo kluczowe `typedef` pozwala nadać nazwę sygnaturze typu funkcji, co zwiększa czytelność kodu i umożliwia wielokrotne użycie tego samego typu.

### Podstawowa składnia typedef

Nowoczesna składnia `typedef` w Dart używa przypisania typu:

```dart
// Definicja aliasu typu — funkcja przyjmująca int i zwracająca bool
typedef Predykat = bool Function(int);

// Alias typu dla funkcji transformującej string
typedef Transformacja = String Function(String wejscie);

bool czyDodatnia(int liczba) => liczba > 0;
bool czyParzysta(int liczba) => liczba % 2 == 0;

String wielkie(String s) => s.toUpperCase();

void main() {
  // Zmienna typu Predykat może przechowywać dowolną pasującą funkcję
  Predykat sprawdzenie = czyDodatnia;
  print(sprawdzenie(5));   // true
  print(sprawdzenie(-3));  // false

  sprawdzenie = czyParzysta; // zmiana na inną funkcję o tej samej sygnaturze
  print(sprawdzenie(4));   // true

  Transformacja t = wielkie;
  print(t('hello'));       // HELLO
}
// Oczekiwane wyjście:
// true
// false
// true
// HELLO
```

### typedef z parametrami generycznymi

`typedef` można parametryzować typami generycznymi, co daje elastyczne aliasy:

```dart
// Generyczny typedef — komparator dwóch wartości tego samego typu
typedef Komparator<T> = int Function(T a, T b);

// Generyczny typedef — mapper z jednego typu na drugi
typedef Mapper<T, R> = R Function(T wartosc);

void sortujListe<T>(List<T> lista, Komparator<T> porownaj) {
  lista.sort(porownaj);
}

void main() {
  var liczby = [3, 1, 4, 1, 5];

  // Komparator<int> — porównywanie rosnąco
  Komparator<int> rosnaco = (a, b) => a.compareTo(b);
  sortujListe(liczby, rosnaco);
  print(liczby);

  // Mapper<int, String> — konwersja int na String
  Mapper<int, String> naString = (n) => 'Liczba: $n';
  var teksty = liczby.map((n) => naString(n)).toList();
  print(teksty);
}
// Oczekiwane wyjście:
// [1, 1, 3, 4, 5]
// [Liczba: 1, Liczba: 1, Liczba: 3, Liczba: 4, Liczba: 5]
```

### typedef dla typów nie-funkcyjnych (Dart 2.13+)

Od Dart 2.13 `typedef` może tworzyć aliasy dla dowolnych typów, nie tylko funkcji:

```dart
// Alias dla złożonego typu generycznego
typedef JSON = Map<String, dynamic>;
typedef Lista2D<T> = List<List<T>>;

void wyswietlJSON(JSON dane) {
  for (var entry in dane.entries) {
    print('  ${entry.key}: ${entry.value}');
  }
}

void main() {
  // Użycie aliasu JSON zamiast Map<String, dynamic>
  JSON konfiguracja = {
    'host': 'localhost',
    'port': 8080,
    'debug': true,
  };

  print('Konfiguracja:');
  wyswietlJSON(konfiguracja);

  // Lista2D jest aliasem dla List<List<int>>
  Lista2D<int> macierz = [
    [1, 2, 3],
    [4, 5, 6],
  ];
  print('Macierz: $macierz');
}
// Oczekiwane wyjście:
// Konfiguracja:
//   host: localhost
//   port: 8080
//   debug: true
// Macierz: [[1, 2, 3], [4, 5, 6]]
```

---

## Typ `Function`

W Dart wszystkie funkcje są obiektami typu `Function`. Ten typ jest nadtypem dla wszystkich typów funkcyjnych i pozwala na dynamiczne przechowywanie dowolnej funkcji.

### `Function` jako typ ogólny

Typ `Function` (bez sygnatury) akceptuje dowolną funkcję, ale nie zapewnia bezpieczeństwa typów przy wywołaniu:

```dart
void main() {
  // Function akceptuje dowolną funkcję
  Function dowolna;

  dowolna = (int x) => x * 2;
  print(Function.apply(dowolna, [5])); // 10 — wywołanie z listą argumentów

  dowolna = (String a, String b) => '$a $b';
  print(Function.apply(dowolna, ['Hello', 'World'])); // Hello World
}
// Oczekiwane wyjście:
// 10
// Hello World
```

Precyzyjne sygnatury typów funkcyjnych dają lepsze bezpieczeństwo typów niż ogólny typ `Function`:

```dart
void main() {
  // Precyzyjna sygnatura — kompilator pilnuje typów
  int Function(int) podwoj = (x) => x * 2;
  String Function(String, int) powtorz = (s, n) => s * n;

  print(podwoj(7));          // 14
  print(powtorz('ha', 3));   // hahaha

  // Próba przypisania niezgodnej funkcji — błąd kompilacji:
  // podwoj = (String s) => s.length; // Błąd: typ nie pasuje
}
// Oczekiwane wyjście:
// 14
// hahaha
```

### Funkcje jako wartości — przechowywanie w kolekcjach

Ponieważ funkcje są obiektami, można je przechowywać w listach, mapach i innych strukturach danych:

```dart
typedef Operacja = double Function(double a, double b);

void main() {
  // Mapa operacji matematycznych
  var operacje = <String, Operacja>{
    'dodaj': (a, b) => a + b,
    'odejmij': (a, b) => a - b,
    'pomnóż': (a, b) => a * b,
    'podziel': (a, b) => a / b,
  };

  // Dynamiczny wybór operacji
  var op = operacje['pomnóż']!;
  print('3 * 4 = ${op(3, 4)}');

  // Iterowanie po wszystkich operacjach
  for (var entry in operacje.entries) {
    print('${entry.key}(10, 3) = ${entry.value(10, 3)}');
  }
}
// Oczekiwane wyjście:
// 3 * 4 = 12.0
// dodaj(10, 3) = 13.0
// odejmij(10, 3) = 7.0
// pomnóż(10, 3) = 30.0
// podziel(10, 3) = 3.3333333333333335
```

### Sprawdzanie typu funkcji

Można sprawdzić czy zmienna jest funkcją lub czy pasuje do konkretnej sygnatury:

```dart
void main() {
  dynamic cos = (int x) => x * x;

  // Sprawdzenie czy jest funkcją
  print(cos is Function);                    // true

  // Sprawdzenie konkretnej sygnatury
  print(cos is int Function(int));           // true
  print(cos is String Function(String));     // false

  // Typ runtime
  print(cos.runtimeType); // (int) => int
}
// Oczekiwane wyjście:
// true
// true
// false
// (int) => int
```

---

## Ćwiczenie 1

### Opis problemu

Napisz program, który implementuje prosty kalkulator operacji na listach. Zdefiniuj `typedef` dla operacji na liście liczb (`List<int>` → `int`) i stwórz mapę dostępnych operacji: `suma`, `max`, `min`, `iloczyn`. Następnie program powinien zastosować każdą operację do podanej listy i wypisać wyniki.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `[3, 7, 2, 9, 4]` | `suma: 25\nmax: 9\nmin: 2\niloczyn: 1512` |
| `[1, 5, 3]` | `suma: 9\nmax: 5\nmin: 1\niloczyn: 15` |

### Wskazówki

1. Zdefiniuj `typedef OperacjaListy = int Function(List<int>)`
2. Użyj metody `reduce` do implementacji sumy i iloczynu
3. Dart posiada metody rozszerzające na `Iterable<int>` — ale możesz też użyć `reduce` z odpowiednim porównaniem dla min/max

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
typedef OperacjaListy = int Function(List<int> liczby);

void main() {
  var operacje = <String, OperacjaListy>{
    'suma': (lista) => lista.reduce((a, b) => a + b),
    'max': (lista) => lista.reduce((a, b) => a > b ? a : b),
    'min': (lista) => lista.reduce((a, b) => a < b ? a : b),
    'iloczyn': (lista) => lista.reduce((a, b) => a * b),
  };

  var dane = [3, 7, 2, 9, 4];

  for (var entry in operacje.entries) {
    print('${entry.key}: ${entry.value(dane)}');
  }
}
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Stwórz system walidacji danych oparty na lambdach. Zdefiniuj typ `Walidator<T>` jako funkcję przyjmującą wartość typu `T` i zwracającą `String?` (null oznacza brak błędu, string to komunikat błędu). Napisz funkcję `waliduj`, która przyjmuje wartość i listę walidatorów, a zwraca listę wszystkich komunikatów błędów. Przetestuj system na walidacji stringa (niepusty, minimalna długość, zawiera cyfrę).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `''` (walidatory: niepusty, min 5 znaków, zawiera cyfrę) | `['Wartość nie może być pusta', 'Minimalna długość: 5', 'Musi zawierać cyfrę']` |
| `'abc'` (walidatory: niepusty, min 5 znaków, zawiera cyfrę) | `['Minimalna długość: 5', 'Musi zawierać cyfrę']` |
| `'haslo123'` (walidatory: niepusty, min 5 znaków, zawiera cyfrę) | `[]` |

### Wskazówki

1. `typedef Walidator<T> = String? Function(T wartość)`
2. Użyj `RegExp(r'\d')` do sprawdzenia czy string zawiera cyfrę
3. Przefiltruj wyniki walidatorów używając `.where((msg) => msg != null)`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
typedef Walidator<T> = String? Function(T wartosc);

List<String> waliduj<T>(T wartosc, List<Walidator<T>> walidatory) {
  return walidatory
      .map((w) => w(wartosc))
      .where((msg) => msg != null)
      .cast<String>()
      .toList();
}

void main() {
  var walidatory = <Walidator<String>>[
    (s) => s.isEmpty ? 'Wartość nie może być pusta' : null,
    (s) => s.length < 5 ? 'Minimalna długość: 5' : null,
    (s) => !RegExp(r'\d').hasMatch(s) ? 'Musi zawierać cyfrę' : null,
  ];

  print(waliduj('', walidatory));
  print(waliduj('abc', walidatory));
  print(waliduj('haslo123', walidatory));
}
```

</details>

---

**Poprzedni moduł:** [Deklaracje funkcji i parametry](01-declarations-params.md)
**Następny moduł:** [Domknięcia i funkcje wyższego rzędu](03-closures-hof.md)
