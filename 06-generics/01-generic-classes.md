---
id: "6.1"
title: "Klasy generyczne"
difficulty: "intermediate"
section: "06-generics"
prerequisites:
  - "Klasy i konstruktory"
  - "Zmienne i typy danych"
---

# 6.1 Klasy generyczne

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Zmienne i typy danych](../01-basics/01-variables-types.md)
- **Cele nauki:**
  1. Rozumieć, czym są typy generyczne i jaki problem rozwiązują — bezpieczeństwo typów i reużywalność kodu bez duplikacji
  2. Korzystać z generycznych typów kolekcji z biblioteki standardowej: `List<T>`, `Set<T>` oraz `Map<K, V>`
  3. Projektować własne klasy generyczne z jednym oraz z wieloma parametrami typowymi
  4. Stosować generyczne aliasy typów (`typedef`) do nadawania czytelnych nazw złożonym typom generycznym

---

## Po co typy generyczne?

Typ generyczny to typ sparametryzowany innym typem. Zamiast pisać osobną klasę dla pudełka na `int`, pudełka na `String` i pudełka na `bool`, piszemy jedną klasę `Pudelko<T>`, gdzie `T` jest parametrem typowym uzupełnianym w momencie użycia.

Generyki dają dwie zasadnicze korzyści:

- **Bezpieczeństwo typów** — kompilator wie, jaki typ przechowuje kolekcja czy klasa, i wyłapuje błędy już na etapie kompilacji, zanim program się uruchomi.
- **Reużywalność bez duplikacji** — jeden kod działa dla wielu typów, bez rzutowania i bez kopiowania implementacji.

Poniższy przykład pokazuje różnicę między kolekcją bez określonego typu elementu a kolekcją generyczną. Wersja generyczna wyłapuje błąd w czasie kompilacji.

```dart
void main() {
  // Lista typu dynamic — akceptuje wszystko, błędy dopiero w czasie działania
  List<dynamic> luznaLista = [1, 'dwa', true];
  luznaLista.add(3.14); // dozwolone, ale tracimy kontrolę nad typami

  // Lista generyczna — typ elementu ustalony na int
  List<int> liczby = [1, 2, 3];
  liczby.add(4); // OK — int pasuje
  // liczby.add('pięć'); // BŁĄD KOMPILACJI: The argument type 'String' can't
  //                     // be assigned to the parameter type 'int'.

  print('Luźna lista: $luznaLista');
  print('Liczby: $liczby');
}
// Oczekiwane wyjście:
// Luźna lista: [1, dwa, true, 3.14]
// Liczby: [1, 2, 3, 4]
```

Drugi przykład ilustruje reużywalność: ta sama generyczna funkcja `pierwszy` działa dla dowolnego typu listy, zachowując przy tym poprawny typ zwracany.

```dart
// Jedna generyczna funkcja obsługuje listy dowolnego typu elementu
T pierwszy<T>(List<T> lista) => lista.first;

void main() {
  var slowo = pierwszy(['a', 'b', 'c']); // T wywnioskowane jako String
  var liczba = pierwszy([10, 20, 30]);   // T wywnioskowane jako int

  print('${slowo.toUpperCase()}');  // działa — kompilator wie, że to String
  print('${liczba + 1}');           // działa — kompilator wie, że to int
}
// Oczekiwane wyjście:
// A
// 11
```

---

## Generyczne typy kolekcji: `List<T>`

`List<T>` to uporządkowana kolekcja elementów typu `T`. Parametr `T` określa, jakiego typu elementy lista może przechowywać.

```dart
// List<T> — jednorodna, uporządkowana kolekcja elementów typu T
void main() {
  // Lista łańcuchów znaków
  List<String> owoce = ['jabłko', 'gruszka', 'śliwka'];
  owoce.add('banan');
  print('Owoce: $owoce');
  print('Pierwszy: ${owoce.first}, liczba: ${owoce.length}');

  // Lista liczb zmiennoprzecinkowych z operacją map (zachowuje typ)
  List<double> ceny = [9.99, 19.50, 4.20];
  List<double> zVat = ceny.map((c) => c * 1.23).toList();
  print('Ceny z VAT: $zVat');
}
// Oczekiwane wyjście:
// Owoce: [jabłko, gruszka, śliwka, banan]
// Pierwszy: jabłko, liczba: 4
// Ceny z VAT: [12.287700000000001, 23.985, 5.166]
```

Drugi przykład pokazuje listę obiektów własnej klasy oraz zagnieżdżenie generyków (`List<List<int>>`).

```dart
// List może przechowywać obiekty własnych klas i być zagnieżdżona
class Punkt {
  final int x;
  final int y;
  Punkt(this.x, this.y);

  @override
  String toString() => '($x, $y)';
}

void main() {
  // Lista obiektów Punkt
  List<Punkt> trasa = [Punkt(0, 0), Punkt(1, 2), Punkt(3, 5)];
  print('Trasa: $trasa');

  // Lista list liczb — macierz 2x2
  List<List<int>> macierz = [
    [1, 2],
    [3, 4],
  ];
  print('Element [1][0]: ${macierz[1][0]}');
}
// Oczekiwane wyjście:
// Trasa: [(0, 0), (1, 2), (3, 5)]
// Element [1][0]: 3
```

---

## Generyczne typy kolekcji: `Set<T>`

`Set<T>` to nieuporządkowana kolekcja **unikalnych** elementów typu `T`. Duplikaty są automatycznie pomijane.

```dart
// Set<T> — kolekcja unikalnych elementów, ignoruje duplikaty
void main() {
  Set<int> liczby = [1, 2, 2, 3, 3, 3].toSet();
  print('Zbiór: $liczby'); // duplikaty usunięte

  liczby.add(4);
  liczby.add(2); // już istnieje — brak zmiany
  print('Po dodaniu: $liczby');
  print('Zawiera 3? ${liczby.contains(3)}');
}
// Oczekiwane wyjście:
// Zbiór: {1, 2, 3}
// Po dodaniu: {1, 2, 3, 4}
// Zawiera 3? true
```

Drugi przykład prezentuje operacje teoriomnogościowe na zbiorach łańcuchów: sumę, przecięcie i różnicę.

```dart
// Set<T> udostępnia operacje teoriomnogościowe
void main() {
  Set<String> zespolA = {'Ala', 'Bartek', 'Cezary'};
  Set<String> zespolB = {'Bartek', 'Cezary', 'Dorota'};

  // Suma — wszyscy z obu zespołów
  print('Suma: ${zespolA.union(zespolB)}');
  // Przecięcie — osoby w obu zespołach
  print('Wspólni: ${zespolA.intersection(zespolB)}');
  // Różnica — tylko z zespołu A
  print('Tylko A: ${zespolA.difference(zespolB)}');
}
// Oczekiwane wyjście:
// Suma: {Ala, Bartek, Cezary, Dorota}
// Wspólni: {Bartek, Cezary}
// Tylko A: {Ala}
```

---

## Generyczne typy kolekcji: `Map<K, V>`

`Map<K, V>` to kolekcja par klucz–wartość, gdzie klucze mają typ `K`, a wartości typ `V`. To przykład typu generycznego z **dwoma** parametrami typowymi.

```dart
// Map<K, V> — dwa parametry typowe: typ klucza i typ wartości
void main() {
  // Klucz String, wartość int — magazyn z ilością sztuk
  Map<String, int> magazyn = {
    'jabłka': 50,
    'gruszki': 30,
  };
  magazyn['śliwki'] = 20; // dodanie nowej pary
  magazyn['jabłka'] = 45; // aktualizacja istniejącej

  print('Magazyn: $magazyn');
  print('Jabłka: ${magazyn['jabłka']}');
  print('Klucze: ${magazyn.keys.toList()}');
}
// Oczekiwane wyjście:
// Magazyn: {jabłka: 45, gruszki: 30, śliwki: 20}
// Jabłka: 45
// Klucze: [jabłka, gruszki, śliwki]
```

Drugi przykład pokazuje mapę o wartościach będących listami (`Map<String, List<String>>`) oraz iterację po wpisach.

```dart
// Wartości mapy mogą być kolekcjami generycznymi — Map<String, List<String>>
void main() {
  Map<String, List<String>> kursy = {
    'Ala': ['Dart', 'Flutter'],
    'Bartek': ['Rust'],
  };

  // Dodanie kursu do istniejącej listy
  kursy['Ala']!.add('SQL');

  // Iteracja po wpisach mapy
  for (final wpis in kursy.entries) {
    print('${wpis.key}: ${wpis.value.join(", ")}');
  }
}
// Oczekiwane wyjście:
// Ala: Dart, Flutter, SQL
// Bartek: Rust
```

---

## Własna klasa generyczna z jednym parametrem

Klasę generyczną deklaruje się, umieszczając parametr typowy w nawiasach ostrokątnych po nazwie klasy: `class Nazwa<T> { ... }`. Parametru `T` można używać w polach, parametrach metod i typach zwracanych.

```dart
// Generyczne pudełko przechowujące jedną wartość dowolnego typu
class Pudelko<T> {
  T zawartosc;

  Pudelko(this.zawartosc);

  // Metoda zwracająca przechowywany typ
  T pobierz() => zawartosc;

  // Metoda przyjmująca wartość tego samego typu
  void wloz(T nowa) => zawartosc = nowa;

  @override
  String toString() => 'Pudelko<$T>($zawartosc)';
}

void main() {
  var pudelkoInt = Pudelko<int>(42);
  print(pudelkoInt);
  print('Podwójnie: ${pudelkoInt.pobierz() * 2}');

  // Typ może być wywnioskowany z argumentu konstruktora
  var pudelkoTekst = Pudelko('cześć');
  pudelkoTekst.wloz('nowa wartość');
  print(pudelkoTekst);
}
// Oczekiwane wyjście:
// Pudelko<int>(42)
// Podwójnie: 84
// Pudelko<String>(nowa wartość)
```

Drugi przykład to prosty generyczny stos (LIFO) — klasyczna struktura danych, która zyskuje na generyczności.

```dart
// Generyczny stos (LIFO) — działa dla dowolnego typu elementu
class Stos<T> {
  final List<T> _elementy = [];

  void wstaw(T element) => _elementy.add(element);

  // Zdejmuje i zwraca element ze szczytu
  T zdejmij() => _elementy.removeLast();

  bool get pusty => _elementy.isEmpty;
  int get rozmiar => _elementy.length;
}

void main() {
  var stos = Stos<String>();
  stos.wstaw('a');
  stos.wstaw('b');
  stos.wstaw('c');
  print('Rozmiar: ${stos.rozmiar}');
  print('Zdjęto: ${stos.zdejmij()}'); // ostatni wstawiony wychodzi pierwszy
  print('Zdjęto: ${stos.zdejmij()}');
  print('Pusty? ${stos.pusty}');
}
// Oczekiwane wyjście:
// Rozmiar: 3
// Zdjęto: c
// Zdjęto: b
// Pusty? false
```

---

## Własna klasa generyczna z wieloma parametrami

Klasa może mieć więcej niż jeden parametr typowy — rozdziela się je przecinkiem: `class Nazwa<K, V> { ... }`. To przydatne, gdy klasa łączy dwie niezależne wartości o różnych typach.

```dart
// Generyczna para z dwoma niezależnymi parametrami typowymi
class Para<A, B> {
  final A pierwszy;
  final B drugi;

  Para(this.pierwszy, this.drugi);

  // Zamiana miejscami zwraca parę o odwróconych typach
  Para<B, A> odwroc() => Para(drugi, pierwszy);

  @override
  String toString() => '($pierwszy, $drugi)';
}

void main() {
  // Para łącząca String z int
  var wiek = Para<String, int>('Ala', 30);
  print('Para: $wiek');

  // odwroc() zwraca Para<int, String>
  Para<int, String> odwrocona = wiek.odwroc();
  print('Odwrócona: $odwrocona');
}
// Oczekiwane wyjście:
// Para: (Ala, 30)
// Odwrócona: (30, Ala)
```

Drugi przykład to generyczny wynik operacji (`Wynik<W, B>`) rozróżniający sukces (typ `W`) od błędu (typ `B`) — częsty wzorzec przy obsłudze operacji, które mogą się nie powieść.

```dart
// Generyczny kontener rozróżniający wartość sukcesu (W) i błędu (B)
class Wynik<W, B> {
  final W? wartosc;
  final B? blad;

  Wynik.sukces(this.wartosc) : blad = null;
  Wynik.porazka(this.blad) : wartosc = null;

  bool get czySukces => blad == null;
}

// Parsowanie tekstu na liczbę — zwraca int przy sukcesie lub String z błędem
Wynik<int, String> parsuj(String tekst) {
  final liczba = int.tryParse(tekst);
  if (liczba == null) {
    return Wynik.porazka('Niepoprawna liczba: "$tekst"');
  }
  return Wynik.sukces(liczba);
}

void main() {
  var ok = parsuj('123');
  var zle = parsuj('abc');

  print(ok.czySukces ? 'Wartość: ${ok.wartosc}' : 'Błąd: ${ok.blad}');
  print(zle.czySukces ? 'Wartość: ${zle.wartosc}' : 'Błąd: ${zle.blad}');
}
// Oczekiwane wyjście:
// Wartość: 123
// Błąd: Niepoprawna liczba: "abc"
```

---

## Generyczne aliasy typów (`typedef`)

Za pomocą `typedef` można nadać krótką, czytelną nazwę złożonemu typowi generycznemu. Alias może też sam być generyczny, przyjmując własne parametry typowe.

```dart
// Generyczny alias typu — czytelna nazwa dla mapy list
typedef Katalog<T> = Map<String, List<T>>;

void main() {
  // Zamiast pisać Map<String, List<int>> używamy Katalog<int>
  Katalog<int> ocenyUczniow = {
    'Ala': [5, 4, 5],
    'Bartek': [3, 4],
  };

  ocenyUczniow['Ala']!.add(4);

  for (final wpis in ocenyUczniow.entries) {
    final srednia =
        wpis.value.reduce((a, b) => a + b) / wpis.value.length;
    print('${wpis.key}: średnia ${srednia.toStringAsFixed(2)}');
  }
}
// Oczekiwane wyjście:
// Ala: średnia 4.50
// Bartek: średnia 3.50
```

Drugi przykład używa aliasu do nazwania generycznego typu funkcyjnego (predykatu), co poprawia czytelność sygnatur.

```dart
// Alias dla generycznego typu funkcyjnego — predykat na wartości typu T
typedef Predykat<T> = bool Function(T wartosc);

// Funkcja filtrująca korzystająca z aliasu w sygnaturze
List<T> filtruj<T>(List<T> lista, Predykat<T> warunek) {
  return [
    for (final element in lista)
      if (warunek(element)) element,
  ];
}

void main() {
  var liczby = [1, 2, 3, 4, 5, 6];
  var parzyste = filtruj(liczby, (n) => n % 2 == 0);
  print('Parzyste: $parzyste');

  var slowa = ['kot', 'słoń', 'pies', 'mysz'];
  var krotkie = filtruj(slowa, (s) => s.length <= 3);
  print('Krótkie: $krotkie');
}
// Oczekiwane wyjście:
// Parzyste: [2, 4, 6]
// Krótkie: [kot]
```

---

## Ćwiczenie 1: Generyczna kolejka (intermediate)

### Opis problemu

Zaimplementuj generyczną klasę `Kolejka<T>` reprezentującą kolejkę FIFO (pierwszy wchodzi, pierwszy wychodzi). Klasa powinna udostępniać:

1. Metodę `void dodaj(T element)` — dodaje element na koniec kolejki.
2. Metodę `T usun()` — usuwa i zwraca element z początku kolejki.
3. Getter `bool czyPusta` — zwraca, czy kolejka jest pusta.
4. Getter `int dlugosc` — zwraca liczbę elementów.

Kolejka musi działać dla dowolnego typu elementu.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `dodaj(1)`, `dodaj(2)`, potem `usun()` | `1` |
| `dodaj('a')`, `dodaj('b')`, `usun()`, potem `dlugosc` | `1` |

### Wskazówki

1. Do przechowywania elementów użyj wewnętrznej `List<T>`.
2. `dodaj` powinno używać `add` (dopisuje na koniec), a `usun` — `removeAt(0)` (zdejmuje z początku).
3. Getter `czyPusta` może korzystać z `isEmpty` listy wewnętrznej.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Kolejka<T> {
  final List<T> _elementy = [];

  void dodaj(T element) => _elementy.add(element);

  // FIFO — zdejmujemy z początku listy
  T usun() => _elementy.removeAt(0);

  bool get czyPusta => _elementy.isEmpty;
  int get dlugosc => _elementy.length;
}

void main() {
  var kolejka = Kolejka<int>();
  kolejka.dodaj(1);
  kolejka.dodaj(2);
  print(kolejka.usun()); // 1 — pierwszy wstawiony

  var tekstowa = Kolejka<String>();
  tekstowa.dodaj('a');
  tekstowa.dodaj('b');
  tekstowa.usun();
  print(tekstowa.dlugosc); // 1
}
// Oczekiwane wyjście:
// 1
// 1
```

</details>

---

## Ćwiczenie 2: Generyczny słownik dwukierunkowy (advanced)

### Opis problemu

Zaimplementuj generyczną klasę `SlownikDwukierunkowy<K, V>` przechowującą pary klucz–wartość, umożliwiającą wyszukiwanie w **obu kierunkach**: po kluczu i po wartości. Wykorzystaj dwa parametry typowe. Klasa powinna udostępniać:

1. Metodę `void dodaj(K klucz, V wartosc)` — dodaje powiązanie w obie strony.
2. Metodę `V? poKluczu(K klucz)` — zwraca wartość dla klucza lub `null`.
3. Metodę `K? poWartosci(V wartosc)` — zwraca klucz dla wartości lub `null`.

Załóż, że powiązania są jednoznaczne (każdy klucz i każda wartość występują raz).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `dodaj('PL', 48)`, potem `poKluczu('PL')` | `48` |
| `dodaj('PL', 48)`, potem `poWartosci(48)` | `PL` |

### Wskazówki

1. Przechowuj dwie mapy: `Map<K, V>` (klucz → wartość) oraz `Map<V, K>` (wartość → klucz).
2. W metodzie `dodaj` aktualizuj obie mapy jednocześnie, aby pozostały spójne.
3. Metody wyszukiwania po prostu odczytują z odpowiedniej mapy — operator `[]` zwraca `null`, gdy klucza nie ma.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class SlownikDwukierunkowy<K, V> {
  final Map<K, V> _wPrzod = {};
  final Map<V, K> _wTyl = {};

  void dodaj(K klucz, V wartosc) {
    _wPrzod[klucz] = wartosc;
    _wTyl[wartosc] = klucz; // utrzymujemy odwrotne powiązanie
  }

  V? poKluczu(K klucz) => _wPrzod[klucz];
  K? poWartosci(V wartosc) => _wTyl[wartosc];
}

void main() {
  var kody = SlownikDwukierunkowy<String, int>();
  kody.dodaj('PL', 48);
  kody.dodaj('DE', 49);

  print(kody.poKluczu('PL'));   // 48
  print(kody.poWartosci(48));   // PL
  print(kody.poWartosci(49));   // DE
  print(kody.poKluczu('FR'));   // null — brak powiązania
}
// Oczekiwane wyjście:
// 48
// PL
// DE
// null
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **Typy generyczne** parametryzują kod typem, zapewniając bezpieczeństwo typów w czasie kompilacji i eliminując duplikację implementacji.
2. **`List<T>`** to uporządkowana kolekcja elementów jednego typu; **`Set<T>`** przechowuje wyłącznie unikalne elementy; **`Map<K, V>`** wiąże klucze z wartościami i jest przykładem typu z dwoma parametrami.
3. Własną klasę generyczną deklaruje się przez `class Nazwa<T>`, a parametr `T` można wykorzystywać w polach, parametrach metod i typach zwracanych.
4. Klasa może mieć **wiele parametrów typowych** (`<A, B>`, `<K, V>`), gdy łączy wartości o niezależnych typach.
5. **Generyczne aliasy** (`typedef Nazwa<T> = ...`) nadają czytelne nazwy złożonym typom, w tym typom funkcyjnym.
6. Dart potrafi **wywnioskować** argumenty typowe z kontekstu, więc często nie trzeba pisać ich jawnie (np. `Pudelko('x')` zamiast `Pudelko<String>('x')`).

---

**Poprzedni moduł:** [Modyfikatory klas Dart 3 i rekordy](../05-oop/04-dart3-modifiers.md)
**Następny moduł:** [Metody generyczne i ograniczenia typów](02-methods-constraints.md)
