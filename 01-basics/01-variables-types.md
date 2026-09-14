---
id: "1.1"
title: "Zmienne i typy danych"
difficulty: "beginner"
section: "01-basics"
prerequisites: []
---

# 1.1 Zmienne i typy danych

## Informacje o module

- **Poziom trudności:** beginner
- **Wymagania wstępne:** brak
- **Cele nauki:**
  1. Rozróżniać deklaracje `var`, `final`, `const` i `late` oraz stosować je w odpowiednich kontekstach
  2. Znać wbudowane typy danych Dart i ich podstawowe operacje
  3. Rozumieć mechanizm wnioskowania typów (type inference) oraz świadomie stosować adnotacje typów

---

## Deklaracje zmiennych

Dart oferuje cztery sposoby deklarowania zmiennych, z których każdy ma inne właściwości dotyczące zmienności i czasu inicjalizacji.

### `var` — zmienna z inferencją typu

Słowo kluczowe `var` deklaruje zmienną, której typ jest wnioskowany z przypisanej wartości. Po przypisaniu typ jest ustalony i nie można go zmienić.

```dart
void main() {
  var imie = 'Anna'; // typ wnioskowany jako String
  var wiek = 28;     // typ wnioskowany jako int

  imie = 'Bartek'; // OK — nowa wartość tego samego typu
  // imie = 42;    // Błąd kompilacji — nie można przypisać int do String

  print('$imie ma $wiek lat');
}
// Oczekiwane wyjście:
// Bartek ma 28 lat
```

Można też jawnie podać typ zamiast `var`:

```dart
void main() {
  String miasto = 'Kraków'; // jawna adnotacja typu
  int populacja = 800000;

  // dynamic pozwala na zmianę typu w runtime
  dynamic wartosc = 'tekst';
  wartosc = 123; // OK — dynamic akceptuje każdy typ

  print('$miasto: $populacja mieszkańców');
  print('wartosc = $wartosc');
}
// Oczekiwane wyjście:
// Kraków: 800000 mieszkańców
// wartosc = 123
```

### `final` — jednokrotne przypisanie

Zmienna `final` może być przypisana tylko raz. Jej wartość jest ustalana w czasie wykonania (runtime).

```dart
void main() {
  final nazwa = 'Dart';       // typ wnioskowany jako String
  final int wersja = 3;       // jawna adnotacja typu

  // nazwa = 'Java'; // Błąd kompilacji — final nie może być ponownie przypisany

  // final pozwala na inicjalizację w runtime
  final teraz = DateTime.now(); // wartość obliczona w momencie wykonania
  print('$nazwa $wersja — czas: $teraz');
}
// Oczekiwane wyjście:
// Dart 3 — czas: (bieżąca data i godzina)
```

Kolekcje zadeklarowane jako `final` nie mogą być ponownie przypisane, ale ich zawartość może się zmieniać:

```dart
void main() {
  final lista = [1, 2, 3]; // referencja jest final
  lista.add(4);            // OK — modyfikacja zawartości dozwolona
  // lista = [5, 6];       // Błąd — nie można zmienić referencji

  print(lista); // [1, 2, 3, 4]
}
// Oczekiwane wyjście:
// [1, 2, 3, 4]
```

### `const` — stała czasu kompilacji

Zmienne `const` muszą mieć wartość znaną już w czasie kompilacji. Obiekt `const` jest głęboko niezmienny (deeply immutable).

```dart
void main() {
  const pi = 3.14159;         // stała czasu kompilacji
  const int maxRetries = 5;   // jawna adnotacja

  // const lista jest głęboko niezmienna
  const kolory = ['red', 'green', 'blue'];
  // kolory.add('yellow'); // Błąd runtime — Cannot add to an unmodifiable list

  // const wymaga wartości znanej w czasie kompilacji
  // const teraz = DateTime.now(); // Błąd kompilacji — DateTime.now() nie jest const

  print('pi = $pi, max = $maxRetries');
  print('kolory: $kolory');
}
// Oczekiwane wyjście:
// pi = 3.14159, max = 5
// kolory: [red, green, blue]
```

Dart kanonizuje obiekty `const` — dwa identyczne wyrażenia `const` wskazują na ten sam obiekt w pamięci:

```dart
void main() {
  const a = [1, 2, 3];
  const b = [1, 2, 3];

  // identical() sprawdza czy to dokładnie ten sam obiekt w pamięci
  print(identical(a, b)); // true — kanonizacja const
}
// Oczekiwane wyjście:
// true
```

### `late` — opóźniona inicjalizacja

Modyfikator `late` pozwala zadeklarować zmienną non-nullable bez natychmiastowej inicjalizacji. Przydatny gdy inicjalizacja jest kosztowna lub zależy od zewnętrznych czynników.

```dart
class Konfiguracja {
  // late pozwala odłożyć inicjalizację zmiennej non-nullable
  late String bazaDanych;

  void zaladuj(String url) {
    bazaDanych = url;
  }
}

void main() {
  final config = Konfiguracja();
  config.zaladuj('postgres://localhost/app');
  print('Baza: ${config.bazaDanych}');
}
// Oczekiwane wyjście:
// Baza: postgres://localhost/app
```

`late` z inicjalizatorem zapewnia leniwą ewaluację — wyrażenie jest obliczane dopiero przy pierwszym odczycie:

```dart
late final ciezkaOperacja = _oblicz(); // obliczone dopiero przy pierwszym użyciu

int _oblicz() {
  print('Obliczam...');
  return 42;
}

void main() {
  print('Przed odczytem');
  print('Wynik: $ciezkaOperacja'); // tu następuje obliczenie
  print('Ponownie: $ciezkaOperacja'); // cache — bez ponownego obliczenia
}
// Oczekiwane wyjście:
// Przed odczytem
// Obliczam...
// Wynik: 42
// Ponownie: 42
```

#### Porównanie momentu inicjalizacji: `final` vs `late final`

Kluczowa różnica polega na tym, **kiedy** następuje ewaluacja wyrażenia inicjalizującego. Zmienna `final` jest obliczana natychmiast w momencie deklaracji, natomiast `late final` odkłada obliczenie do pierwszego odczytu:

```dart
int _oblicz(String skad) {
  print('Obliczam ($skad)...');
  return 42;
}

void main() {
  print('--- final lokalne ---');
  final a = _oblicz('final'); // oblicza się TERAZ — w momencie deklaracji
  print('Przed odczytem a');
  print('Wynik: $a');

  print('');
  print('--- late final lokalne ---');
  late final b = _oblicz('late final'); // oblicza się dopiero przy pierwszym odczycie
  print('Przed odczytem b');
  print('Wynik: $b'); // dopiero tu następuje wywołanie _oblicz('late final')
}
// Oczekiwane wyjście:
// --- final lokalne ---
// Obliczam (final)...
// Przed odczytem a
// Wynik: 42
//
// --- late final lokalne ---
// Przed odczytem b
// Obliczam (late final)...
// Wynik: 42
```

Zauważ kolejność wypisywanych komunikatów — przy `final` komunikat "Obliczam" pojawia się **przed** "Przed odczytem", a przy `late final` — **po**. To oznacza, że `late` jest idealny do kosztownych inicjalizacji, które mogą nigdy nie być potrzebne.

---

## Wbudowane typy danych

Dart posiada zestaw wbudowanych typów, które stanowią fundament systemu typów.

### `int` i `double`

Typ `int` reprezentuje liczby całkowite, a `double` — liczby zmiennoprzecinkowe. Oba dziedziczą po abstrakcyjnej klasie `num`.

```dart
void main() {
  int licznik = 100;
  int hex = 0xFF;           // zapis szesnastkowy
  int binarny = int.parse('1010', radix: 2); // zapis binarny przez int.parse — wartość 10

  double temperatura = 36.6;
  double notacja = 1.5e3;  // notacja naukowa — 1500.0

  // num może przechowywać zarówno int jak i double
  num wynik = licznik + temperatura;

  print('hex=$hex, bin=$binarny');
  print('temperatura=$temperatura, notacja=$notacja');
  print('wynik=$wynik');
}
// Oczekiwane wyjście:
// hex=255, bin=10
// temperatura=36.6, notacja=1500.0
// wynik=136.6
```

Konwersje i metody liczbowe:

```dart
void main() {
  // Parsowanie ze Stringa
  var a = int.parse('42');       // 42
  var b = double.parse('3.14'); // 3.14

  // Konwersje między typami
  double c = a.toDouble();  // 42.0
  int d = b.toInt();        // 3 (obcięcie, nie zaokrąglenie)
  int e = b.round();        // 3 (zaokrąglenie)

  // Przydatne metody
  print((-5).abs());        // 5 — wartość bezwzględna
  print(7.remainder(3));    // 1 — reszta z dzielenia
  print(3.14.ceil());       // 4 — zaokrąglenie w górę

  print('a=$a, b=$b, c=$c, d=$d, e=$e');
}
// Oczekiwane wyjście:
// 5
// 1
// 4
// a=42, b=3.14, c=42.0, d=3, e=3
```

### `String`

Ciągi znaków w Dart to sekwencje jednostek kodu UTF-16. Można je definiować w apostrofach lub cudzysłowach.

```dart
void main() {
  var pojedyncze = 'Hello';
  var podwojne = "World";

  // Interpolacja — wstawianie wyrażeń w string
  var powitanie = '$pojedyncze $podwojne!';
  var obliczenie = 'Suma: ${2 + 3}';

  // Wieloliniowe stringi
  var wieloliniowy = '''
Linia 1
Linia 2
Linia 3''';

  // Raw string — bez interpretacji escape sequences
  var sciezka = r'C:\Users\dart\projekt'; // \U i \d nie są escape'owane

  print(powitanie);
  print(obliczenie);
  print(wieloliniowy);
  print(sciezka);
}
// Oczekiwane wyjście:
// Hello World!
// Suma: 5
// Linia 1
// Linia 2
// Linia 3
// C:\Users\dart\projekt
```

Podstawowe operacje na stringach:

```dart
void main() {
  var tekst = '  Dart jest super  ';

  print(tekst.trim());              // 'Dart jest super' — usunięcie białych znaków
  print(tekst.trim().toUpperCase()); // 'DART JEST SUPER'
  print('Dart'.contains('ar'));     // true
  print('a,b,c'.split(','));        // [a, b, c]
  print('Hello'.replaceAll('l', 'L')); // HeLLo
  print('Dart'.padLeft(8, '-'));    // ----Dart
}
// Oczekiwane wyjście:
// Dart jest super
// DART JEST SUPER
// true
// [a, b, c]
// HeLLo
// ----Dart
```

### `bool`

Typ logiczny przyjmuje jedną z dwóch wartości: `true` lub `false`.

```dart
void main() {
  bool aktywny = true;
  bool zalogowany = false;

  // Dart nie konwertuje automatycznie wartości na bool
  // if (1) {} // Błąd kompilacji — int nie jest bool

  // Operacje logiczne
  var wynik = aktywny && !zalogowany; // true AND NOT false = true
  print('aktywny=$aktywny, zalogowany=$zalogowany, wynik=$wynik');
}
// Oczekiwane wyjście:
// aktywny=true, zalogowany=false, wynik=true
```

Porównania zwracają wartości `bool`:

```dart
void main() {
  var x = 10;
  var y = 20;

  bool wiekszy = x > y;      // false
  bool rowny = x == 10;      // true
  bool rozny = x != y;       // true

  print('wiekszy=$wiekszy, rowny=$rowny, rozny=$rozny');
}
// Oczekiwane wyjście:
// wiekszy=false, rowny=true, rozny=true
```

### `List`

`List` to uporządkowana kolekcja elementów z dostępem indeksowym (indeksowanie od 0).

```dart
void main() {
  // Literał listowy z inferencją typu
  var owoce = ['jabłko', 'banan', 'czereśnia'];

  // Jawna deklaracja z typem
  List<int> liczby = [10, 20, 30];
  print(liczby); // [10, 20, 30]

  // Dostęp i modyfikacja
  print(owoce[0]);         // jabłko — pierwszy element
  owoce.add('daktyl');     // dodanie na koniec
  owoce.insert(1, 'awokado'); // wstawienie na pozycji 1

  print(owoce);
  print('Długość: ${owoce.length}');
  print('Zawiera banan: ${owoce.contains("banan")}');
}
// Oczekiwane wyjście:
// [10, 20, 30]
// jabłko
// [jabłko, awokado, banan, czereśnia, daktyl]
// Długość: 5
// Zawiera banan: true
```

Tworzenie list z użyciem konstruktorów i operatorów:

```dart
void main() {
  // List.generate — tworzenie z generatora
  var kwadraty = List.generate(5, (i) => i * i); // [0, 1, 4, 9, 16]

  // List.filled — lista wypełniona wartością
  var zera = List.filled(3, 0); // [0, 0, 0]

  // Spread operator w literale
  var wszystkie = [...kwadraty, ...zera]; // [0, 1, 4, 9, 16, 0, 0, 0]

  // Collection if i for
  var parzyste = [for (var k in kwadraty) if (k % 2 == 0) k]; // [0, 4, 16]

  print('kwadraty: $kwadraty');
  print('wszystkie: $wszystkie');
  print('parzyste: $parzyste');
}
// Oczekiwane wyjście:
// kwadraty: [0, 1, 4, 9, 16]
// wszystkie: [0, 1, 4, 9, 16, 0, 0, 0]
// parzyste: [0, 4, 16]
```

### `Set`

`Set` to nieuporządkowana kolekcja unikalnych elementów.

```dart
void main() {
  // Literał setowy
  var unikalne = {'Dart', 'Java', 'Python', 'Dart'}; // duplikat usunięty

  // Jawna deklaracja
  Set<int> primes = {2, 3, 5, 7, 11};
  print(primes); // {2, 3, 5, 7, 11}

  unikalne.add('Go');
  unikalne.add('Dart'); // ignorowane — już istnieje

  print(unikalne);
  print('Zawiera Java: ${unikalne.contains("Java")}');
  print('Liczba: ${unikalne.length}');
}
// Oczekiwane wyjście:
// {2, 3, 5, 7, 11}
// {Dart, Java, Python, Go}
// Zawiera Java: true
// Liczba: 4
```

Operacje zbiorowe:

```dart
void main() {
  var a = {1, 2, 3, 4};
  var b = {3, 4, 5, 6};

  // Operacje matematyczne na zbiorach
  print(a.union(b));         // {1, 2, 3, 4, 5, 6} — suma zbiorów
  print(a.intersection(b));  // {3, 4} — część wspólna
  print(a.difference(b));    // {1, 2} — różnica (elementy w a, których nie ma w b)
}
// Oczekiwane wyjście:
// {1, 2, 3, 4, 5, 6}
// {3, 4}
// {1, 2}
```

### `Map`

`Map` to kolekcja par klucz-wartość. Klucze muszą być unikalne.

```dart
void main() {
  // Literał mapowy
  var stolice = {
    'Polska': 'Warszawa',
    'Niemcy': 'Berlin',
    'Francja': 'Paryż',
  };

  // Jawna deklaracja z typami
  Map<String, int> oceny = {'matematyka': 5, 'fizyka': 4};

  // Dostęp i modyfikacja
  print(stolice['Polska']);    // Warszawa
  stolice['Włochy'] = 'Rzym'; // dodanie nowej pary
  oceny.remove('fizyka');      // usunięcie wpisu

  print(stolice);
  print('Klucze ocen: ${oceny.keys}');
}
// Oczekiwane wyjście:
// Warszawa
// {Polska: Warszawa, Niemcy: Berlin, Francja: Paryż, Włochy: Rzym}
// Klucze ocen: (matematyka)
```

Iterowanie po mapie:

```dart
void main() {
  var produkty = {'chleb': 4.50, 'mleko': 3.20, 'masło': 7.99};

  // Iterowanie po parach klucz-wartość
  for (var entry in produkty.entries) {
    print('${entry.key}: ${entry.value} zł');
  }

  // Metody przydatne
  print('Zawiera klucz "chleb": ${produkty.containsKey("chleb")}');
  print('Wartości: ${produkty.values.toList()}');
}
// Oczekiwane wyjście:
// chleb: 4.5 zł
// mleko: 3.2 zł
// masło: 7.99 zł
// Zawiera klucz "chleb": true
// Wartości: [4.5, 3.2, 7.99]
```

### `Symbol`

`Symbol` to nieprzezroczysty identyfikator używany głównie w refleksji (dart:mirrors). Definiuje się go za pomocą prefiksu `#`.

```dart
void main() {
  // Literał symbolowy
  Symbol s1 = #mojaZmienna;
  Symbol s2 = Symbol('mojaZmienna');

  // Symbole o tej samej nazwie są równe
  print(s1 == s2); // true
  print(s1);       // Symbol("mojaZmienna")
}
// Oczekiwane wyjście:
// true
// Symbol("mojaZmienna")
```

Symbole są przydatne przy refleksji i jako klucze stałe:

```dart
void main() {
  // Symbole jako stałe klucze w mapach — bardziej wydajne niż stringi
  var metadane = <Symbol, dynamic>{
    #nazwa: 'Widget',
    #wersja: 2,
    #aktywny: true,
  };

  print(metadane[#nazwa]);   // Widget
  print(metadane[#wersja]);  // 2
}
// Oczekiwane wyjście:
// Widget
// 2
```

### `Null`

Typ `Null` ma jedyną wartość `null`. W Dart z null safety, zmienna może być `null` tylko jeśli jej typ jest jawnie oznaczony jako nullable (z `?`).

```dart
void main() {
  // Typ nullable — akceptuje null
  String? opcjonalny = null;
  int? wynik;  // domyślnie null dla typów nullable

  print(opcjonalny);  // null
  print(wynik);       // null

  // Typ non-nullable — nie akceptuje null
  String wymagany = 'wartość';
  // wymagany = null; // Błąd kompilacji — non-nullable nie akceptuje null

  print(wymagany);
}
// Oczekiwane wyjście:
// null
// null
// wartość
```

Sprawdzanie wartości null:

```dart
void main() {
  String? tekst = null;

  // Sprawdzenie czy wartość jest null
  if (tekst == null) {
    print('tekst jest null');
  }

  tekst = 'hello';
  // Po sprawdzeniu Dart promuje typ do non-nullable
  if (tekst != null) {
    print(tekst.toUpperCase()); // OK — Dart wie, że tekst nie jest null
  }
}
// Oczekiwane wyjście:
// tekst jest null
// HELLO
```

---

## Wnioskowanie typów (type inference)

Dart posiada zaawansowany system wnioskowania typów, który automatycznie ustala typ zmiennej na podstawie przypisanej wartości.

Poniższy przykład ilustruje, jak Dart wnioskuje typy z kontekstu:

```dart
void main() {
  var nazwa = 'Dart';     // wnioskowany: String
  var liczba = 42;        // wnioskowany: int
  var pi = 3.14;          // wnioskowany: double
  var lista = [1, 2, 3];  // wnioskowany: List<int>
  var mapa = {'a': 1};    // wnioskowany: Map<String, int>

  // Typ jest ustalony — próba przypisania innego typu da błąd
  // nazwa = 42; // Błąd: A value of type 'int' can't be assigned to a variable of type 'String'

  print(nazwa.runtimeType);  // String
  print(liczba.runtimeType); // int
  print(pi.runtimeType);     // double
  print(lista.runtimeType);  // List<int>
  print(mapa.runtimeType);   // _Map<String, int>
}
// Oczekiwane wyjście:
// String
// int
// double
// List<int>
// _Map<String, int>
```

Inferencja działa również z kolekcjami heterogenicznymi:

```dart
void main() {
  // Dart wnioskuje najwęższy wspólny typ
  var mieszana = [1, 2.5, 3]; // List<num> — wspólny nadtyp int i double
  var obiekty = [1, 'tekst', true]; // List<Object> — najbardziej ogólny typ

  print(mieszana.runtimeType); // List<num>
  print(obiekty.runtimeType);  // List<Object>
}
// Oczekiwane wyjście:
// List<num>
// List<Object>
```

---

## Adnotacje typów

Jawne adnotacje typów są opcjonalne gdy Dart może sam wnioskować typ, ale są zalecane w deklaracjach publicznych API i dla zwiększenia czytelności.

Poniższy przykład pokazuje kiedy adnotacje typów są przydatne:

```dart
// Publiczne API — adnotacje typów zwiększają czytelność
String formatujAdres(String ulica, int numer, {String? mieszkanie}) {
  var adres = '$ulica $numer'; // tutaj var jest wystarczający — prosta lokalna zmienna
  if (mieszkanie != null) {
    adres += '/$mieszkanie';
  }
  return adres;
}

void main() {
  // Zmienne lokalne — var jest wystarczający
  var wynik = formatujAdres('Marszałkowska', 10, mieszkanie: '5A');
  print(wynik);
}
// Oczekiwane wyjście:
// Marszałkowska 10/5A
```

Kiedy jawny typ jest niezbędny — deklaracja bez inicjalizacji:

```dart
void main() {
  // Bez inicjalizacji Dart nie może wnioskować typu
  String tekst; // trzeba podać typ jawnie
  List<int> numery; // konieczna adnotacja

  tekst = 'zainicjalizowane później';
  numery = [1, 2, 3];

  // Pusty literał kolekcji wymaga adnotacji typu
  var pustaLista = <String>[]; // bez <String> byłoby List<dynamic>
  pustaLista.add('element');

  print(tekst);
  print(numery);
  print(pustaLista);
}
// Oczekiwane wyjście:
// zainicjalizowane później
// [1, 2, 3]
// [element]
```

---

## Ćwiczenie 1

### Opis problemu

Napisz program, który deklaruje zmienne przechowujące informacje o studencie: imię (`String`), wiek (`int`), średnia ocen (`double`), aktywny status (`bool`), i listę przedmiotów (`List<String>`). Program powinien wypisać sformatowany profil studenta. Użyj odpowiednich deklaracji (`final` tam, gdzie wartość nie powinna się zmienić, `var` dla zmiennych).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| imię: `'Anna'`, wiek: `22`, średnia: `4.5`, aktywny: `true`, przedmioty: `['Matematyka', 'Fizyka']` | `Profil studenta:\nImię: Anna\nWiek: 22\nŚrednia: 4.5\nStatus: aktywny\nPrzedmioty: Matematyka, Fizyka` |
| imię: `'Jan'`, wiek: `20`, średnia: `3.8`, aktywny: `false`, przedmioty: `['Chemia']` | `Profil studenta:\nImię: Jan\nWiek: 20\nŚrednia: 3.8\nStatus: nieaktywny\nPrzedmioty: Chemia` |

### Wskazówki

1. Użyj `final` dla wartości, które nie zmienią się po inicjalizacji (np. imię)
2. Użyj interpolacji stringów (`$zmienna` lub `${wyrażenie}`) do formatowania wyjścia
3. Metoda `join(', ')` na liście łączy elementy w jeden string

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  final String imie = 'Anna';
  int wiek = 22;
  double srednia = 4.5;
  var aktywny = true;
  var przedmioty = ['Matematyka', 'Fizyka'];

  var status = aktywny ? 'aktywny' : 'nieaktywny';

  print('Profil studenta:');
  print('Imię: $imie');
  print('Wiek: $wiek');
  print('Średnia: $srednia');
  print('Status: $status');
  print('Przedmioty: ${przedmioty.join(", ")}');
}
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Napisz program, który demonstruje różnicę między `const` i `final`. Stwórz dwie listy: jedną `const` i jedną `final`. Spróbuj dodać element do obu i obsłuż ewentualny błąd. Program powinien wypisać informację o tym, która operacja się powiodła, a która nie.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| lista const: `[1, 2, 3]`, lista final: `[10, 20, 30]` | `Lista final po dodaniu: [10, 20, 30, 40]\nLista const: nie można modyfikować (Unsupported operation: Cannot add to an unmodifiable list)` |
| lista const: `['a', 'b']`, lista final: `['x', 'y']` | `Lista final po dodaniu: [x, y, z]\nLista const: nie można modyfikować (Unsupported operation: Cannot add to an unmodifiable list)` |

### Wskazówki

1. Użyj bloku `try-catch` do przechwycenia błędu przy modyfikacji listy `const`
2. `const` sprawia, że obiekt jest głęboko niezmienny — nie można dodawać, usuwać ani modyfikować elementów
3. `final` chroni jedynie referencję — sama kolekcja może być modyfikowana

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  final listaFinal = [10, 20, 30];
  const listaConst = [1, 2, 3];

  // Modyfikacja listy final — dozwolona
  listaFinal.add(40);
  print('Lista final po dodaniu: $listaFinal');

  // Próba modyfikacji listy const — błąd runtime
  try {
    listaConst.add(4);
  } on UnsupportedError catch (e) {
    print('Lista const: nie można modyfikować ($e)');
  }
}
```

</details>

---

## Ćwiczenie 3

### Opis problemu

Napisz program, który tworzy `Map<String, List<int>>` przechowującą oceny uczniów. Dodaj co najmniej 3 uczniów z różną liczbą ocen. Następnie oblicz i wypisz średnią ocen każdego ucznia, posortowaną od najwyższej do najniższej.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `{'Anna': [5, 4, 5], 'Jan': [3, 4, 3], 'Ewa': [5, 5, 5, 4]}` | `Ewa: 4.75\nAnna: 4.67\nJan: 3.33` |
| `{'Tomek': [5, 5], 'Kasia': [4, 3, 4], 'Piotr': [2, 3]}` | `Tomek: 5.00\nKasia: 3.67\nPiotr: 2.50` |

### Wskazówki

1. Oblicz sumę elementów listy za pomocą metody `reduce((a, b) => a + b)` lub `fold(0, (a, b) => a + b)`
2. Użyj `toStringAsFixed(2)` aby sformatować średnią do 2 miejsc po przecinku
3. `entries.toList()..sort(...)` pozwala posortować wpisy mapy

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  final oceny = <String, List<int>>{
    'Anna': [5, 4, 5],
    'Jan': [3, 4, 3],
    'Ewa': [5, 5, 5, 4],
  };

  // Obliczanie średnich i sortowanie
  var srednie = oceny.entries.map((entry) {
    var suma = entry.value.fold(0, (a, b) => a + b);
    var srednia = suma / entry.value.length;
    return MapEntry(entry.key, srednia);
  }).toList();

  // Sortowanie malejąco po średniej
  srednie.sort((a, b) => b.value.compareTo(a.value));

  for (var entry in srednie) {
    print('${entry.key}: ${entry.value.toStringAsFixed(2)}');
  }
}
```

</details>

---

**Następny moduł:** [Operatory](02-operators.md)
