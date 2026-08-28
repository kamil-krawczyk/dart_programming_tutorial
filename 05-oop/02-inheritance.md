---
id: "5.2"
title: "Dziedziczenie i interfejsy"
difficulty: "intermediate"
section: "05-oop"
prerequisites:
  - "Klasy i konstruktory"
  - "Deklaracje funkcji i parametry"
---

# 5.2 Dziedziczenie i interfejsy

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Deklaracje funkcji i parametry](../04-functions/01-declarations-params.md)
- **Cele nauki:**
  1. Stosować dziedziczenie za pomocą słowa kluczowego `extends` oraz nadpisywać metody z użyciem `@override` i `super`
  2. Rozumieć różnicę między dziedziczeniem implementacji (`extends`) a implementowaniem interfejsu (`implements`)
  3. Projektować klasy abstrakcyjne definiujące kontrakty niekompletnych implementacji
  4. Znać rolę klasy `Object` jako korzenia hierarchii typów oraz poprawnie nadpisywać `toString`, `operator ==` i `hashCode`

---

## Dziedziczenie z `extends`

Dziedziczenie pozwala jednej klasie (podklasie) przejąć instancje pól i metody innej klasy (nadklasy). W Dart klasa dziedziczy po dokładnie jednej nadklasie za pomocą słowa kluczowego `extends`. Podklasa może wykorzystać odziedziczone składniki, dodać własne oraz **nadpisać** (override) zachowanie odziedziczonych metod.

Konstruktor podklasy wywołuje konstruktor nadklasy za pomocą `super`. Jeśli nadklasa ma konstruktor bezargumentowy, wywołanie `super()` jest dodawane automatycznie.

### Nadpisywanie metod i `super`

Poniższy przykład modeluje hierarchię pracowników w systemie kadrowym. Podklasa `Menedzer` nadpisuje metodę obliczania wynagrodzenia, korzystając z implementacji nadklasy przez `super`.

```dart
class Pracownik {
  final String imie;
  final double pensjaBazowa;

  Pracownik(this.imie, this.pensjaBazowa);

  // Metoda przeznaczona do nadpisania w podklasach
  double miesiecznaWyplata() => pensjaBazowa;

  String opis() => '$imie: ${miesiecznaWyplata()} zł';
}

class Menedzer extends Pracownik {
  final double premia;

  // super(...) przekazuje argumenty do konstruktora nadklasy
  Menedzer(String imie, double pensjaBazowa, this.premia)
      : super(imie, pensjaBazowa);

  @override // adnotacja sygnalizuje zamiar nadpisania metody
  double miesiecznaWyplata() => super.miesiecznaWyplata() + premia; // super wywołuje wersję z nadklasy
}

void main() {
  final p = Pracownik('Anna', 5000);
  final m = Menedzer('Bartek', 8000, 2000);
  print(p.opis());
  print(m.opis());
}
// Oczekiwane wyjście:
// Anna: 5000.0 zł
// Bartek: 10000.0 zł
```

Kolejny przykład pokazuje wielopoziomową hierarchię (trzy poziomy dziedziczenia) oraz odziedziczone metody, które nie wymagają nadpisania.

```dart
class Pojazd {
  final String marka;
  Pojazd(this.marka);

  String uruchom() => '$marka: uruchamiam silnik';
}

class Samochod extends Pojazd {
  final int liczbaDrzwi;
  Samochod(String marka, this.liczbaDrzwi) : super(marka);

  String otworzBagaznik() => '$marka: bagażnik otwarty';
}

// Trzeci poziom dziedziczenia — SamochodElektryczny dziedziczy po Samochod
class SamochodElektryczny extends Samochod {
  final int pojemnoscBaterii;
  SamochodElektryczny(String marka, int liczbaDrzwi, this.pojemnoscBaterii)
      : super(marka, liczbaDrzwi);

  @override
  String uruchom() => '$marka: cichy start (${pojemnoscBaterii}kWh)';
}

void main() {
  final auto = SamochodElektryczny('Tesla', 4, 75);
  print(auto.uruchom());          // nadpisana metoda
  print(auto.otworzBagaznik());   // odziedziczona z Samochod
  print('Marka: ${auto.marka}');  // odziedziczone pole z Pojazd
}
// Oczekiwane wyjście:
// Tesla: cichy start (75kWh)
// Tesla: bagażnik otwarty
// Marka: Tesla
```

### Błędne użycie: brak wywołania konstruktora nadklasy

Gdy nadklasa nie posiada konstruktora bezargumentowego, podklasa **musi** jawnie wywołać jeden z jej konstruktorów. Pominięcie tego wywołania jest błędem kompilacji.

```dart
class Konto {
  final String numer;
  Konto(this.numer); // brak konstruktora bezargumentowego
}

class KontoOszczednosciowe extends Konto {
  final double oprocentowanie;

  // BŁĄD: nie wywołano super(...), a Konto nie ma konstruktora bezargumentowego
  KontoOszczednosciowe(this.oprocentowanie);
}

void main() {
  KontoOszczednosciowe(0.05);
}
// Błąd kompilacji:
// The implicitly invoked unnamed constructor from 'Konto' has required
// parameters. Try adding an explicit super parameter with the required
// arguments.

// Poprawna wersja:
// KontoOszczednosciowe(this.oprocentowanie, String numer) : super(numer);
```

---

## Klasy abstrakcyjne

Klasa abstrakcyjna (`abstract class`) nie może być instancjonowana bezpośrednio — służy jako baza definiująca wspólny kontrakt dla podklas. Może zawierać zarówno metody **abstrakcyjne** (bez ciała, do zaimplementowania przez podklasy), jak i metody **konkretne** (z gotową implementacją).

Poniższy przykład definiuje abstrakcyjną figurę geometryczną z abstrakcyjną metodą `pole()` i konkretną metoda `opis()` wspólną dla wszystkich figur.

```dart
abstract class Figura {
  double pole(); // metoda abstrakcyjna — brak ciała

  // metoda konkretna — współdzielona przez wszystkie podklasy
  String opis() => 'Figura o polu ${pole().toStringAsFixed(2)}';
}

class Kolo extends Figura {
  final double promien;
  Kolo(this.promien);

  @override
  double pole() => 3.14159 * promien * promien; // implementacja wymagana
}

class Prostokat extends Figura {
  final double a, b;
  Prostokat(this.a, this.b);

  @override
  double pole() => a * b;
}

void main() {
  final figury = <Figura>[Kolo(2), Prostokat(3, 4)];
  for (final f in figury) {
    print(f.opis()); // korzysta z konkretnej metody opis()
  }
}
// Oczekiwane wyjście:
// Figura o polu 12.57
// Figura o polu 12.00
```

Drugi przykład pokazuje klasę abstrakcyjną jako bazę repozytorium danych — definiuje kontrakt operacji, ale pozostawia szczegóły przechowywania podklasom.

```dart
abstract class Repozytorium<T> {
  void zapisz(T element); // abstrakcyjne
  List<T> pobierzWszystkie(); // abstrakcyjne

  // konkretna metoda korzystająca z metod abstrakcyjnych
  int liczba() => pobierzWszystkie().length;
}

class RepozytoriumPamieciowe extends Repozytorium<String> {
  final List<String> _dane = [];

  @override
  void zapisz(String element) => _dane.add(element);

  @override
  List<String> pobierzWszystkie() => List.unmodifiable(_dane);
}

void main() {
  final repo = RepozytoriumPamieciowe();
  repo.zapisz('a');
  repo.zapisz('b');
  print('Liczba elementów: ${repo.liczba()}');
  print(repo.pobierzWszystkie());
}
// Oczekiwane wyjście:
// Liczba elementów: 2
// [a, b]
```

### Błędne użycie: instancjonowanie klasy abstrakcyjnej

Próba utworzenia instancji klasy abstrakcyjnej jest błędem kompilacji.

```dart
abstract class Figura {
  double pole();
}

void main() {
  // BŁĄD: nie można utworzyć instancji klasy abstrakcyjnej
  final f = Figura();
  print(f);
}
// Błąd kompilacji:
// Abstract classes can't be instantiated.
// Try creating an instance of a concrete subtype.
```

---

## Interfejsy i `implements`

W Dart nie istnieje osobne słowo kluczowe `interface` do definiowania interfejsów (poza modyfikatorem klasy `interface`, omówionym w module Dart 3). Zamiast tego **każda klasa niejawnie definiuje interfejs** złożony ze wszystkich swoich publicznych składników. Za pomocą słowa kluczowego `implements` klasa zobowiązuje się dostarczyć własną implementację **wszystkich** składników wskazanego interfejsu.

Kluczowa różnica względem `extends`:
- `extends` — dziedziczy implementację; można nadpisywać wybrane metody i wywoływać `super`.
- `implements` — dziedziczy tylko kontrakt (sygnatury); trzeba zaimplementować **wszystko** od zera, bez dostępu do `super`.

Poniższy przykład pokazuje klasę implementującą interfejs zdefiniowany przez inną klasę.

```dart
// Ta klasa pełni rolę interfejsu — definiuje kontrakt
class ZrodloDanych {
  String pobierz() => 'domyślne dane';
}

// implements zobowiązuje do zaimplementowania WSZYSTKICH składników interfejsu
class ZrodloTestowe implements ZrodloDanych {
  @override
  String pobierz() => 'dane testowe'; // musi być zaimplementowane od zera
}

void main() {
  ZrodloDanych zrodlo = ZrodloTestowe();
  print(zrodlo.pobierz());
}
// Oczekiwane wyjście:
// dane testowe
```

Drugi przykład demonstruje implementowanie **wielu** interfejsów jednocześnie — możliwość, której nie daje `extends`.

```dart
abstract class Drukowalne {
  String drukuj();
}

abstract class Serializowalne {
  Map<String, dynamic> naJson();
}

// Klasa może implementować wiele interfejsów naraz
class Faktura implements Drukowalne, Serializowalne {
  final int numer;
  final double kwota;
  Faktura(this.numer, this.kwota);

  @override
  String drukuj() => 'Faktura #$numer: $kwota zł';

  @override
  Map<String, dynamic> naJson() => {'numer': numer, 'kwota': kwota};
}

void main() {
  final f = Faktura(101, 250.0);
  print(f.drukuj());
  print(f.naJson());
}
// Oczekiwane wyjście:
// Faktura #101: 250.0 zł
// {numer: 101, kwota: 250.0}
```

### Błędne użycie: niepełna implementacja interfejsu

Przy użyciu `implements` klasa musi dostarczyć implementację każdego składnika interfejsu. Pominięcie choćby jednego to błąd kompilacji.

```dart
abstract class Logger {
  void info(String wiadomosc);
  void blad(String wiadomosc);
}

// BŁĄD: brakuje implementacji metody blad()
class KonsolaLogger implements Logger {
  @override
  void info(String wiadomosc) => print('INFO: $wiadomosc');
}

void main() {
  KonsolaLogger().info('start');
}
// Błąd kompilacji:
// Missing concrete implementation of 'Logger.blad'.
// Try implementing the missing method, or make the class abstract.
```

---

## Klasa `Object` i jej metody

W Dart każda klasa dziedziczy pośrednio lub bezpośrednio po klasie `Object` — jest ona korzeniem całej hierarchii typów (z wyjątkiem `Null`). Oznacza to, że każdy obiekt posiada metody zdefiniowane w `Object`, w tym `toString()`, `operator ==` oraz getter `hashCode`.

### `toString()`

Domyślna implementacja `toString()` zwraca mało czytelną nazwę typu (np. `Instance of 'Punkt'`). Nadpisanie jej daje przydatną reprezentację tekstową, wykorzystywaną m.in. przez `print` i interpolację napisów.

```dart
class Punkt {
  final int x, y;
  Punkt(this.x, this.y);

  @override
  String toString() => 'Punkt($x, $y)'; // nadpisanie Object.toString
}

void main() {
  final p = Punkt(3, 5);
  print(p);            // print używa toString()
  print('Punkt: $p');  // interpolacja również używa toString()
}
// Oczekiwane wyjście:
// Punkt(3, 5)
// Punkt: Punkt(3, 5)
```

### `operator ==` i `hashCode`

Domyślnie `==` porównuje **tożsamość** obiektów (czy to ta sama instancja). Aby porównywać obiekty według wartości pól, należy nadpisać `operator ==`. Zgodnie z kontraktem `Object`, jeśli nadpisujemy `==`, **musimy** również nadpisać `hashCode` — dwa obiekty równe według `==` muszą mieć identyczny `hashCode`, aby poprawnie działały w kolekcjach opartych na haszowaniu (`Set`, `Map`).

```dart
class Wspolrzedne {
  final double szerokosc, dlugosc;
  Wspolrzedne(this.szerokosc, this.dlugosc);

  @override
  bool operator ==(Object other) => // parametr typu Object zgodnie z sygnaturą
      other is Wspolrzedne &&
      other.szerokosc == szerokosc &&
      other.dlugosc == dlugosc;

  @override
  int get hashCode => Object.hash(szerokosc, dlugosc); // spójny z operatorem ==
}

void main() {
  final a = Wspolrzedne(52.2, 21.0);
  final b = Wspolrzedne(52.2, 21.0);
  final c = Wspolrzedne(50.0, 19.9);

  print(a == b);            // true — równe według wartości
  print(a == c);            // false
  print(identical(a, b));   // false — to różne instancje

  // Poprawne działanie w Set dzięki spójnym == i hashCode
  final zbior = {a, b, c};
  print('Rozmiar zbioru: ${zbior.length}'); // a i b traktowane jako jeden element
}
// Oczekiwane wyjście:
// true
// false
// false
// Rozmiar zbioru: 2
```

### Błędne użycie: nadpisanie `==` bez `hashCode`

Nadpisanie `operator ==` bez nadpisania `hashCode` łamie kontrakt `Object` i powoduje błędne działanie kolekcji haszujących. Reguła lintera `hash_and_equals` (włączona m.in. w zestawach `package:lints/recommended.yaml` i Flutter) zgłasza w tym przypadku ostrzeżenie.

```dart
class Etykieta {
  final String nazwa;
  Etykieta(this.nazwa);

  // ŹLE: nadpisano tylko ==, hashCode pozostaje domyślny (oparty na tożsamości)
  @override
  bool operator ==(Object other) =>
      other is Etykieta && other.nazwa == nazwa;
}

void main() {
  final zbior = <Etykieta>{};
  zbior.add(Etykieta('pilne'));
  zbior.add(Etykieta('pilne')); // różny hashCode → traktowane jako inny element

  // Set zawiera duplikaty, bo hashCode nie jest spójny z ==
  print('Rozmiar zbioru: ${zbior.length}'); // 2 zamiast oczekiwanego 1
}
// Ostrzeżenie lintera (hash_and_equals):
// Missing a corresponding override of 'hashCode'.
// Try overriding 'hashCode' or removing '=='.
// Oczekiwane wyjście (błędne z punktu widzenia logiki):
// Rozmiar zbioru: 2
```

---

## Ćwiczenie 1: Hierarchia figur z polem i obwodem

### Opis problemu

Zaprojektuj hierarchię klas modelującą figury geometryczne:

- Abstrakcyjna klasa `Ksztalt` z dwiema metodami abstrakcyjnymi: `double pole()` oraz `double obwod()`, a także konkretną metodą `String podsumowanie()` zwracającą tekst w formacie `Pole: X, Obwod: Y` (wartości zaokrąglone do 2 miejsc po przecinku).
- Klasa `Kwadrat` dziedzicząca po `Ksztalt` (pole `bok`).
- Klasa `Kolo` dziedzicząca po `Ksztalt` (pole `promien`, użyj wartości pi = 3.14159).

Napisz funkcję `main`, która tworzy listę kształtów i wypisuje podsumowanie każdego z nich.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Kwadrat(5).podsumowanie()` | `Pole: 25.00, Obwod: 20.00` |
| `Kolo(3).podsumowanie()` | `Pole: 28.27, Obwod: 18.85` |

### Wskazówki

1. Umieść wspólną logikę formatowania w konkretnej metodzie `podsumowanie()` w klasie abstrakcyjnej — podklasy jej nie nadpisują
2. Użyj `toStringAsFixed(2)` do zaokrąglenia wartości
3. Obwód koła to `2 * pi * promien`, pole to `pi * promien * promien`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
abstract class Ksztalt {
  double pole();
  double obwod();

  // konkretna metoda współdzielona przez wszystkie podklasy
  String podsumowanie() =>
      'Pole: ${pole().toStringAsFixed(2)}, Obwod: ${obwod().toStringAsFixed(2)}';
}

class Kwadrat extends Ksztalt {
  final double bok;
  Kwadrat(this.bok);

  @override
  double pole() => bok * bok;

  @override
  double obwod() => 4 * bok;
}

class Kolo extends Ksztalt {
  static const double pi = 3.14159;
  final double promien;
  Kolo(this.promien);

  @override
  double pole() => pi * promien * promien;

  @override
  double obwod() => 2 * pi * promien;
}

void main() {
  final ksztalty = <Ksztalt>[Kwadrat(5), Kolo(3)];
  for (final k in ksztalty) {
    print(k.podsumowanie());
  }
}
// Oczekiwane wyjście:
// Pole: 25.00, Obwod: 20.00
// Pole: 28.27, Obwod: 18.85
```

</details>

---

## Ćwiczenie 2: Interfejs i równość według wartości

### Opis problemu

Zdefiniuj klasę `Wersja` reprezentującą wersję oprogramowania w formacie semantycznym (`major`, `minor`, `patch`). Klasa ma:

- Implementować interfejs `Comparable<Wersja>` (metoda `int compareTo(Wersja other)`), porównując najpierw `major`, potem `minor`, na końcu `patch`.
- Nadpisywać `operator ==`, `hashCode` oraz `toString()` (format `major.minor.patch`).

Napisz funkcję `main`, która sortuje listę wersji i weryfikuje równość według wartości.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Wersja(1, 2, 0) == Wersja(1, 2, 0)` | `true` |
| `[Wersja(1,2,0), Wersja(1,0,5)]..sort()` | `[1.0.5, 1.2.0]` |

### Wskazówki

1. `Comparable<T>` wymaga metody `int compareTo(T other)` zwracającej wartość ujemną, zero lub dodatnią
2. Możesz użyć operatora `??` na wyniku `compareTo` kolejnych pól: porównaj `major`, a jeśli równe (wynik 0), przejdź do `minor`
3. Nadpisując `==`, nadpisz też `hashCode` — użyj `Object.hash(major, minor, patch)`
4. `List.sort()` domyślnie korzysta z `compareTo`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Wersja implements Comparable<Wersja> {
  final int major, minor, patch;
  Wersja(this.major, this.minor, this.patch);

  @override
  int compareTo(Wersja other) {
    // porównuje kolejne pola aż do znalezienia różnicy
    if (major != other.major) return major.compareTo(other.major);
    if (minor != other.minor) return minor.compareTo(other.minor);
    return patch.compareTo(other.patch);
  }

  @override
  bool operator ==(Object other) =>
      other is Wersja &&
      other.major == major &&
      other.minor == minor &&
      other.patch == patch;

  @override
  int get hashCode => Object.hash(major, minor, patch);

  @override
  String toString() => '$major.$minor.$patch';
}

void main() {
  print(Wersja(1, 2, 0) == Wersja(1, 2, 0)); // true

  final wersje = [Wersja(1, 2, 0), Wersja(1, 0, 5), Wersja(2, 0, 0)];
  wersje.sort(); // korzysta z compareTo
  print(wersje);
}
// Oczekiwane wyjście:
// true
// [1.0.5, 1.2.0, 2.0.0]
```

</details>

---

**Poprzedni moduł:** [Klasy i konstruktory](../05-oop/01-classes-constructors.md)
**Następny moduł:** [Mixiny](../05-oop/03-mixins.md)
