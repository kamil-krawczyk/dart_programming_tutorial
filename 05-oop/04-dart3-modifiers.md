---
id: "5.4"
title: "Modyfikatory klas Dart 3 i rekordy"
difficulty: "intermediate"
section: "05-oop"
prerequisites:
  - "Dziedziczenie"
  - "Mixiny"
---

# 5.4 Modyfikatory klas Dart 3 i rekordy

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Dziedziczenie](../05-oop/02-inheritance.md), [Mixiny](../05-oop/03-mixins.md)
- **Cele nauki:**
  1. Rozumieć przeznaczenie modyfikatorów klas wprowadzonych w Dart 3 (`sealed`, `base`, `interface`, `final`, `mixin`) i wiedzieć, jaki poziom kontroli nad dziedziczeniem zapewnia każdy z nich
  2. Wykorzystywać klasy zapieczętowane (`sealed`) do modelowania zamkniętych hierarchii typów i uzyskiwania sprawdzania kompletności (exhaustiveness) w wyrażeniach `switch`
  3. Stosować modyfikatory `base`, `interface` i `final`, aby precyzyjnie kontrolować, w jaki sposób inne biblioteki mogą rozszerzać i implementować typy
  4. Tworzyć i wykorzystywać rekordy (records) — lekkie, niemutowalne agregaty wartości z polami pozycyjnymi i nazwanymi, destrukturyzacją oraz strukturalną równością

---

## Wprowadzenie: po co modyfikatory klas?

Przed Dart 3 każda klasa mogła być jednocześnie rozszerzana (`extends`), implementowana jako interfejs (`implements`) oraz — jeśli nie miała konstruktora z argumentami — używana jako mixin. Dawało to autorom bibliotek niewielką kontrolę nad tym, jak ich typy są używane przez kod klienta.

Dart 3 wprowadza **modyfikatory klas**, które pozwalają autorowi jawnie określić, jakie operacje są dozwolone na danym typie. Modyfikatory umieszcza się przed słowem kluczowym `class` (lub `mixin`). Poniższa tabela podsumowuje ich efekt.

| Modyfikator | `extends` (z tej samej biblioteki) | `extends` (z innej biblioteki) | `implements` (z innej biblioteki) | Instancjonowanie |
|-------------|-----------------------------------|-------------------------------|-----------------------------------|------------------|
| (brak) | tak | tak | tak | tak |
| `base` | tak | tak | nie | tak |
| `interface` | tak | nie | tak | tak |
| `final` | tak | nie | nie | tak |
| `sealed` | tak | nie | nie | nie (abstrakcyjna) |
| `abstract` | tak | tak | tak | nie |

Modyfikatory można łączyć (np. `abstract base class`), z zachowaniem określonych reguł. W kolejnych sekcjach omawiamy każdy z nich osobno.

---

## Klasy zapieczętowane (`sealed`)

Modyfikator `sealed` oznacza klasę, której **nie można instancjonować** ani rozszerzać/implementować **poza biblioteką**, w której została zadeklarowana. Wszystkie jej bezpośrednie podtypy muszą być zdefiniowane w tym samym pliku. Dzięki temu kompilator zna pełen, zamknięty zbiór podtypów.

Największą zaletą klas zapieczętowanych jest **sprawdzanie kompletności (exhaustiveness)** w wyrażeniach `switch`: jeśli obsłużysz wszystkie podtypy, kompilator nie wymaga klauzuli `default`; jeśli któryś pominiesz, zgłosi błąd kompilacji.

Poniższy przykład modeluje kształty geometryczne jako zapieczętowaną hierarchię i oblicza pole powierzchni za pomocą wyczerpującego `switch`.

```dart
// sealed — zamknięta hierarchia; wszystkie podtypy w tym samym pliku
sealed class Ksztalt {}

class Kolo extends Ksztalt {
  final double promien;
  Kolo(this.promien);
}

class Prostokat extends Ksztalt {
  final double szerokosc;
  final double wysokosc;
  Prostokat(this.szerokosc, this.wysokosc);
}

class Kwadrat extends Ksztalt {
  final double bok;
  Kwadrat(this.bok);
}

double pole(Ksztalt k) => switch (k) {
      // Kompilator wie, że to wszystkie możliwe podtypy — brak potrzeby default
      Kolo(:final promien) => 3.14159 * promien * promien,
      Prostokat(:final szerokosc, :final wysokosc) => szerokosc * wysokosc,
      Kwadrat(:final bok) => bok * bok,
    };

void main() {
  final ksztalty = <Ksztalt>[Kolo(2), Prostokat(3, 4), Kwadrat(5)];
  for (final k in ksztalty) {
    print('Pole: ${pole(k).toStringAsFixed(2)}');
  }
}
// Oczekiwane wyjście:
// Pole: 12.57
// Pole: 12.00
// Pole: 25.00
```

Drugi przykład pokazuje typową sytuację modelowania stanu — np. wyniku operacji sieciowej. Zapieczętowana hierarchia gwarantuje, że każdy stan zostanie obsłużony.

```dart
// Modelowanie wyniku operacji jako zapieczętowanej hierarchii stanów
sealed class WynikPobierania {}

class Ladowanie extends WynikPobierania {}

class Sukces extends WynikPobierania {
  final String dane;
  Sukces(this.dane);
}

class Blad extends WynikPobierania {
  final String komunikat;
  Blad(this.komunikat);
}

String opisz(WynikPobierania wynik) => switch (wynik) {
      Ladowanie() => 'Trwa ładowanie...',
      Sukces(:final dane) => 'Pobrano: $dane',
      Blad(:final komunikat) => 'Błąd: $komunikat',
    };

void main() {
  print(opisz(Ladowanie()));
  print(opisz(Sukces('profil użytkownika')));
  print(opisz(Blad('brak połączenia')));
}
// Oczekiwane wyjście:
// Trwa ładowanie...
// Pobrano: profil użytkownika
// Błąd: brak połączenia
```

### Częsty błąd: pominięcie podtypu w `switch`

Gdy hierarchia jest zapieczętowana, pominięcie choćby jednego podtypu w wyrażeniu `switch` (bez `default`) powoduje **błąd kompilacji**, a nie cichy błąd w czasie działania. To główna zaleta `sealed`.

```dart
sealed class Platnosc {}

class Karta extends Platnosc {}

class Gotowka extends Platnosc {}

// BŁĄD KOMPILACJI: nieobsłużony podtyp Gotowka
String metoda(Platnosc p) => switch (p) {
      Karta() => 'karta',
      // brakuje przypadku Gotowka() — switch nie jest wyczerpujący
    };
// Błąd analizatora:
// error: The type 'Platnosc' is not exhaustively matched by the switch cases
// since it doesn't match 'Gotowka()'.
```

---

## Modyfikator `base`

Modyfikator `base` wymusza **dziedziczenie zamiast implementacji**. Typ oznaczony jako `base` może być rozszerzany (`extends`), ale **nie może być implementowany** (`implements`) poza biblioteką, w której go zdefiniowano. Każdy podtyp klasy `base` (w tym pośredni) również musi być oznaczony jako `base`, `final` lub `sealed`.

Zastosowanie: gwarantujesz, że każda instancja podtypu faktycznie odziedziczy implementację twoich metod (a nie dostarczy własnej przez `implements`), dzięki czemu chronione niezmienniki i stan prywatny są zawsze zainicjowane.

Poniższy przykład pokazuje klasę bazową repozytorium, która ma zawsze być dziedziczona, aby współdzielić logikę logowania.

```dart
// base — można extends, nie można implements spoza biblioteki
base class Repozytorium {
  final List<String> _log = [];

  void zapiszLog(String akcja) {
    _log.add(akcja); // wspólny stan, którego nie wolno obejść przez implements
  }

  int get liczbaOperacji => _log.length;
}

// Podtyp base musi również być base/final/sealed
base class RepozytoriumUzytkownikow extends Repozytorium {
  void dodajUzytkownika(String nazwa) {
    zapiszLog('dodano: $nazwa'); // korzysta z odziedziczonej implementacji
  }
}

void main() {
  final repo = RepozytoriumUzytkownikow();
  repo.dodajUzytkownika('Ala');
  repo.dodajUzytkownika('Bartek');
  print('Liczba operacji: ${repo.liczbaOperacji}');
}
// Oczekiwane wyjście:
// Liczba operacji: 2
```

Drugi przykład ilustruje niedozwolone użycie `implements` na klasie `base` z innej biblioteki oraz poprawną alternatywę przez `extends`.

```dart
// Poniżej Repozytorium jest zdefiniowane w tym samym pliku tylko dla
// kompletności przykładu — reguła opisana niżej dotyczy sytuacji, w której
// Repozytorium pochodzi z INNEJ biblioteki niż kod, który próbuje go użyć.
base class Repozytorium {
  final List<String> _log = [];

  void zapiszLog(String akcja) {
    _log.add(akcja);
  }

  int get liczbaOperacji => _log.length;
}

// BŁĄD (gdyby Repozytorium pochodziło z innej biblioteki):
// nie można implements klasy base spoza jej biblioteki
// base class BlednaImplementacja implements Repozytorium {}
// error: The class 'Repozytorium' can't be implemented outside of its library
// because it's a base class.

// POPRAWNIE: dziedziczenie przez extends działa niezależnie od biblioteki
base class PoprawnaSubklasa extends Repozytorium {
  void wykonaj() => zapiszLog('operacja');
}

void main() {
  final r = PoprawnaSubklasa();
  r.wykonaj();
  print('OK, operacji: ${r.liczbaOperacji}');
}
// Oczekiwane wyjście:
// OK, operacji: 1
```

---

## Modyfikator `interface`

Modyfikator `interface` wymusza sytuację odwrotną do `base`: typ oznaczony jako `interface` może być **implementowany** (`implements`) przez inne biblioteki, ale **nie może być rozszerzany** (`extends`) poza własną biblioteką. Klasa `interface` definiuje kontrakt (zestaw sygnatur), a klienci dostarczają własną implementację.

Zastosowanie: publikujesz kontrakt, którego szczegóły implementacji chcesz zachować dla siebie, jednocześnie pozwalając klientom tworzyć własne, zgodne implementacje.

Poniższy przykład definiuje interfejs logera, który klienci implementują na własne sposoby.

```dart
// interface — można implements, nie można extends spoza biblioteki
interface class Loger {
  void loguj(String wiadomosc) {
    print('[LOG] $wiadomosc'); // domyślna implementacja (opcjonalna)
  }
}

// Klient dostarcza własną implementację kontraktu
class LogerKonsolowy implements Loger {
  @override
  void loguj(String wiadomosc) => print('KONSOLA: $wiadomosc');
}

class LogerZPrefiksem implements Loger {
  final String prefiks;
  LogerZPrefiksem(this.prefiks);

  @override
  void loguj(String wiadomosc) => print('$prefiks $wiadomosc');
}

void main() {
  final List<Loger> logery = [
    LogerKonsolowy(),
    LogerZPrefiksem('[APP]'),
  ];
  for (final l in logery) {
    l.loguj('start systemu');
  }
}
// Oczekiwane wyjście:
// KONSOLA: start systemu
// [APP] start systemu
```

Drugi przykład pokazuje niedozwolone `extends` na klasie `interface` z innej biblioteki wraz z poprawnym `implements`.

```dart
// Poniżej Loger jest zdefiniowany w tym samym pliku tylko dla kompletności
// przykładu — reguła opisana niżej dotyczy sytuacji, w której Loger pochodzi
// z INNEJ biblioteki niż kod, który próbuje go rozszerzyć.
interface class Loger {
  void loguj(String wiadomosc) {
    print('[LOG] $wiadomosc');
  }
}

// BŁĄD (gdyby Loger pochodził z innej biblioteki):
// nie można extends klasy interface spoza jej biblioteki
// class BlednyLoger extends Loger {}
// error: The class 'Loger' can't be extended outside of its library
// because it's an interface class.

// POPRAWNIE: implementacja przez implements (własna implementacja WSZYSTKICH metod)
class LogerCichy implements Loger {
  @override
  void loguj(String wiadomosc) {
    // celowo nic nie robi — tryb cichy
  }
}

void main() {
  LogerCichy().loguj('to się nie pojawi');
  print('Gotowe');
}
// Oczekiwane wyjście:
// Gotowe
```

---

## Modyfikator `final`

Modyfikator `final` zamyka typ dla rozszerzeń **poza własną biblioteką**: nie można go ani `extends`, ani `implements` z innej biblioteki. Wewnątrz własnej biblioteki nadal możesz go rozszerzać. Łączy więc ograniczenia `base` i `interface`.

Zastosowanie: masz pewność, że nikt spoza twojej biblioteki nie zmieni ani nie podszyje się pod dany typ, dzięki czemu możesz bezpiecznie dodawać do niego nowe metody w przyszłości bez psucia kodu klientów.

Poniższy przykład pokazuje klasę konfiguracji, która ma być używana wyłącznie w postaci dostarczonej przez bibliotekę.

```dart
// final — nie można extends ani implements spoza biblioteki
final class Konfiguracja {
  final String srodowisko;
  final int limitPolaczen;

  Konfiguracja({required this.srodowisko, required this.limitPolaczen});

  bool get produkcyjne => srodowisko == 'prod';
}

void main() {
  final konf = Konfiguracja(srodowisko: 'prod', limitPolaczen: 100);
  print('Produkcja: ${konf.produkcyjne}, limit: ${konf.limitPolaczen}');
}
// Oczekiwane wyjście:
// Produkcja: true, limit: 100
```

Drugi przykład demonstruje niedozwolone rozszerzanie klasy `final` z innej biblioteki.

```dart
// Poniżej Konfiguracja jest zdefiniowana w tym samym pliku tylko dla
// kompletności przykładu — reguła opisana niżej dotyczy sytuacji, w której
// Konfiguracja pochodzi z INNEJ biblioteki niż kod, który próbuje ją rozszerzyć
// lub zaimplementować.
final class Konfiguracja {
  final String srodowisko;
  final int limitPolaczen;

  Konfiguracja({required this.srodowisko, required this.limitPolaczen});

  bool get produkcyjne => srodowisko == 'prod';
}

// BŁĄD (gdyby Konfiguracja pochodziła z innej biblioteki):
// nie można extends klasy final spoza jej biblioteki
// class RozszerzonaKonf extends Konfiguracja {}
// error: The class 'Konfiguracja' can't be extended outside of its library
// because it's a final class.

// BŁĄD (gdyby Konfiguracja pochodziła z innej biblioteki):
// nie można też implements klasy final spoza jej biblioteki
// class UdawanaKonf implements Konfiguracja {}
// error: The class 'Konfiguracja' can't be implemented outside of its library
// because it's a final class.

// POPRAWNIE: kompozycja zamiast dziedziczenia
class UslugaZKonfiguracja {
  final Konfiguracja konf;
  UslugaZKonfiguracja(this.konf); // przechowujemy instancję, nie dziedziczymy

  void uruchom() => print('Uruchomiono w: ${konf.srodowisko}');
}

void main() {
  final usluga = UslugaZKonfiguracja(
    Konfiguracja(srodowisko: 'dev', limitPolaczen: 10),
  );
  usluga.uruchom();
}
// Oczekiwane wyjście:
// Uruchomiono w: dev
```

---

## Modyfikator `mixin` i klasy mixinowe (`mixin class`)

Dart 3 rozdziela pojęcia klasy i mixinu. Deklaracja `mixin` tworzy typ przeznaczony wyłącznie do dołączania przez `with` — nie można go instancjonować. Deklaracja `mixin class` tworzy typ, który może pełnić **obie role**: być użyty jako mixin (`with`) oraz jako zwykła klasa (`extends`/instancjonowanie).

Ograniczenie `on` w deklaracji `mixin` określa, że mixin może być dołączony wyłącznie do klas będących podtypem wskazanego typu — dzięki temu mixin może bezpiecznie odwoływać się do składowych tego typu.

Poniższy przykład pokazuje `mixin` z ograniczeniem `on` oraz `mixin class` używaną w obu rolach.

```dart
// Klasa bazowa, do której mixin będzie ograniczony
class Pojazd {
  double predkosc = 0;
}

// mixin z ograniczeniem on — może być dołączony tylko do podtypów Pojazd
mixin Turbo on Pojazd {
  void wlaczTurbo() {
    predkosc *= 2; // dostęp do składowej Pojazd dzięki ograniczeniu on
  }
}

class SamochodSportowy extends Pojazd with Turbo {}

void main() {
  final auto = SamochodSportowy()..predkosc = 100;
  auto.wlaczTurbo();
  print('Prędkość: ${auto.predkosc}');
}
// Oczekiwane wyjście:
// Prędkość: 200.0
```

Drugi przykład prezentuje `mixin class` — typ używany zarówno jako mixin, jak i jako samodzielna klasa.

```dart
// mixin class — może być i mixinem (with), i zwykłą klasą (extends/instancja)
mixin class Identyfikowalny {
  int _id = 0;
  void nadajId(int id) => _id = id;
  int get id => _id;
}

// Użycie jako mixin
class Dokument with Identyfikowalny {
  final String tytul;
  Dokument(this.tytul);
}

// Użycie jako zwykła klasa bazowa
class Encja extends Identyfikowalny {}

void main() {
  final dok = Dokument('Raport')..nadajId(42);
  print('Dokument ${dok.tytul}, id=${dok.id}');

  final e = Encja()..nadajId(7); // instancjonowanie mixin class bezpośrednio
  print('Encja id=${e.id}');
}
// Oczekiwane wyjście:
// Dokument Raport, id=42
// Encja id=7
```

---

## Rekordy (records)

Rekord to lekki, **niemutowalny** agregat wielu wartości, potencjalnie różnych typów. Rekordy pozwalają grupować dane bez definiowania osobnej klasy — idealne np. do zwracania wielu wartości z funkcji.

### Typy rekordów: pola pozycyjne i nazwane

Rekord tworzy się w nawiasach. Pola mogą być **pozycyjne** (dostęp przez `$1`, `$2`, ...) lub **nazwane** (dostęp przez nazwę). Typ rekordu zapisuje się analogicznie do jego wartości.

```dart
void main() {
  // Rekord z polami pozycyjnymi — typ (String, int)
  final osoba = ('Ala', 30);
  print('${osoba.$1} ma ${osoba.$2} lat'); // dostęp przez $1, $2

  // Rekord z polami nazwanymi — typ ({String imie, int wiek})
  final ({String imie, int wiek}) uzytkownik = (imie: 'Bartek', wiek: 25);
  print('${uzytkownik.imie}, ${uzytkownik.wiek}'); // dostęp przez nazwy

  // Rekord mieszany: pola pozycyjne i nazwane razem
  final punkt = (1.5, 2.5, etykieta: 'A');
  print('(${punkt.$1}, ${punkt.$2}) [${punkt.etykieta}]');
}
// Oczekiwane wyjście:
// Ala ma 30 lat
// Bartek, 25
// (1.5, 2.5) [A]
```

Drugi przykład pokazuje typowe zastosowanie rekordu jako typu zwracanego funkcji zwracającej wiele wartości.

```dart
// Funkcja zwracająca rekord z polami nazwanymi — czytelny wielowartościowy wynik
({int min, int max, double srednia}) statystyki(List<int> dane) {
  final minimum = dane.reduce((a, b) => a < b ? a : b);
  final maksimum = dane.reduce((a, b) => a > b ? a : b);
  final suma = dane.reduce((a, b) => a + b);
  return (min: minimum, max: maksimum, srednia: suma / dane.length);
}

void main() {
  final wynik = statystyki([4, 8, 15, 16, 23, 42]);
  print('min=${wynik.min}, max=${wynik.max}, śr=${wynik.srednia.toStringAsFixed(2)}');
}
// Oczekiwane wyjście:
// min=4, max=42, śr=18.00
```

### Destrukturyzacja rekordów

Rekordy można **destrukturyzować** — rozpakować ich pola do osobnych zmiennych w jednym kroku, wykorzystując dopasowywanie wzorców (patterns).

```dart
({int min, int max, double srednia}) statystyki(List<int> dane) {
  final minimum = dane.reduce((a, b) => a < b ? a : b);
  final maksimum = dane.reduce((a, b) => a > b ? a : b);
  final suma = dane.reduce((a, b) => a + b);
  return (min: minimum, max: maksimum, srednia: suma / dane.length);
}

void main() {
  // Destrukturyzacja rekordu z polami nazwanymi do zmiennych lokalnych
  final (:min, :max, :srednia) = statystyki([10, 20, 30]);
  print('min=$min max=$max śr=$srednia');

  // Destrukturyzacja rekordu pozycyjnego
  final wspolrzedne = (52.23, 21.01);
  final (szerokosc, dlugosc) = wspolrzedne; // rozpakowanie do dwóch zmiennych
  print('lat=$szerokosc lon=$dlugosc');
}
// Oczekiwane wyjście:
// min=10 max=30 śr=20.0
// lat=52.23 lon=21.01
```

### Równość strukturalna rekordów

Dwa rekordy są **równe** (`==`), gdy mają ten sam kształt (te same pola pozycyjne i nazwane) oraz równe wartości odpowiadających pól. Rekordy mają też spójny `hashCode`, więc mogą być kluczami w `Map` czy elementami `Set`. To zachowanie różni się od domyślnej równości referencyjnej klas.

```dart
void main() {
  final a = (imie: 'Ala', wiek: 30);
  final b = (imie: 'Ala', wiek: 30);
  final c = (imie: 'Ola', wiek: 30);

  // Równość strukturalna — porównywane są wartości pól, nie referencje
  print(a == b); // true — ten sam kształt i wartości
  print(a == c); // false — różne wartości pola imie

  // Rekordy mają spójny hashCode, więc działają jako klucze/elementy zbiorów
  final zbior = {a, b, c};
  print('Unikalnych rekordów: ${zbior.length}'); // b jest duplikatem a
}
// Oczekiwane wyjście:
// true
// false
// Unikalnych rekordów: 2
```

---

## Ćwiczenie 1: Zapieczętowana hierarchia figur (podstawowe/intermediate)

### Opis problemu

Zaprojektuj zapieczętowaną hierarchię reprezentującą figury geometryczne, składającą się z **co najmniej 3 klas** i **co najmniej jednego poziomu dziedziczenia**:

- `sealed class Figura` — typ bazowy (poziom 1).
- `class Kolo extends Figura` z polem `promien` (poziom 2).
- `class Wielokat extends Figura` — pośredni typ dla figur wielobocznych (poziom 2).
- `class Kwadrat extends Wielokat` z polem `bok` (poziom 3 — drugi poziom dziedziczenia).

Napisz funkcję `double obwod(Figura f)` używającą **wyczerpującego** wyrażenia `switch` (bez `default`) do obliczenia obwodu każdej figury.

**Oczekiwane relacje klas:** `Kolo` i `Wielokat` dziedziczą bezpośrednio po `Figura`; `Kwadrat` dziedziczy po `Wielokat`. Cała hierarchia jest zapieczętowana (`sealed`), więc `switch` po `Figura` musi być wyczerpujący.

**Oczekiwane zachowanie:** dla `Kolo` obwód to `2 * pi * promien`; dla `Kwadrat` obwód to `4 * bok`. Poprawne rozwiązanie kompiluje się bez klauzuli `default` i zwraca właściwe wartości.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `obwod(Kolo(1))` | `6.283185307179586` |
| `obwod(Kwadrat(5))` | `20.0` |

### Wskazówki

1. Umieść wszystkie klasy hierarchii w tym samym pliku — jest to wymóg `sealed`.
2. Ponieważ `Wielokat` jest sam abstrakcyjnym poziomem pośrednim bez własnych instancji, rozważ oznaczenie go jako `sealed` lub `abstract`, aby uzyskać wyczerpujący `switch` po jego podtypach.
3. Do obliczeń użyj stałej `pi` z `dart:math`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:math' as math;

sealed class Figura {}

class Kolo extends Figura {
  final double promien;
  Kolo(this.promien);
}

// Pośredni poziom hierarchii — również sealed dla wyczerpującego switch
sealed class Wielokat extends Figura {}

class Kwadrat extends Wielokat {
  final double bok;
  Kwadrat(this.bok);
}

double obwod(Figura f) => switch (f) {
      Kolo(:final promien) => 2 * math.pi * promien,
      Kwadrat(:final bok) => 4 * bok,
    };

void main() {
  print(obwod(Kolo(1)));
  print(obwod(Kwadrat(5)));
}
// Oczekiwane wyjście:
// 6.283185307179586
// 20.0
```

</details>

---

## Ćwiczenie 2: Kontrolowana hierarchia płatności z `base` (intermediate)

### Opis problemu

Zaprojektuj hierarchię metod płatności składającą się z **co najmniej 3 klas** i **co najmniej jednego poziomu dziedziczenia**, w której klasa bazowa jest oznaczona jako `base`, aby wymusić dziedziczenie (a nie implementację) i zagwarantować wspólną logikę naliczania prowizji:

- `base class Platnosc` — klasa bazowa z chronionym polem `_prowizja` (double) i metodą `double kwotaZProwizja(double kwota)`, która dodaje prowizję (poziom 1).
- `base class PlatnoscKarta extends Platnosc` — ustawia prowizję na 2% (poziom 2).
- `base class PlatnoscBLIK extends Platnosc` — ustawia prowizję na 0% (poziom 2).

Każda podklasa ma metodę `String opis()` zwracającą nazwę metody płatności.

**Oczekiwane relacje klas:** `PlatnoscKarta` i `PlatnoscBLIK` dziedziczą po `Platnosc`; wszystkie klasy są `base`, więc żadna biblioteka kliencka nie może ich `implements` — tylko `extends`.

**Oczekiwane zachowanie:** `kwotaZProwizja` zwraca `kwota * (1 + _prowizja)`. Dla karty prowizja wynosi 0.02, dla BLIK 0.0. Poprawne rozwiązanie kompiluje się (podklasy są `base`) i zwraca właściwe kwoty.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `PlatnoscKarta().kwotaZProwizja(100)` | `102.0` |
| `PlatnoscBLIK().kwotaZProwizja(100)` | `100.0` |

### Wskazówki

1. Pamiętaj, że podtyp klasy `base` musi również być oznaczony jako `base`, `final` lub `sealed`.
2. Ustaw wartość `_prowizja` w konstruktorze każdej podklasy.
3. Prowizję inicjuj w konstruktorze podklasy, wywołując konstruktor bazowy lub przypisując pole bezpośrednio.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
base class Platnosc {
  double _prowizja = 0;

  double kwotaZProwizja(double kwota) => kwota * (1 + _prowizja);

  String opis() => 'Płatność';
}

base class PlatnoscKarta extends Platnosc {
  PlatnoscKarta() {
    _prowizja = 0.02; // 2% prowizji
  }

  @override
  String opis() => 'Karta';
}

base class PlatnoscBLIK extends Platnosc {
  PlatnoscBLIK() {
    _prowizja = 0.0; // brak prowizji
  }

  @override
  String opis() => 'BLIK';
}

void main() {
  final List<Platnosc> metody = [PlatnoscKarta(), PlatnoscBLIK()];
  for (final m in metody) {
    print('${m.opis()}: ${m.kwotaZProwizja(100)}');
  }
}
// Oczekiwane wyjście:
// Karta: 102.0
// BLIK: 100.0
```

</details>

---

## Ćwiczenie 3: Interfejs powiadomień z rekordami wyników (advanced)

### Opis problemu

Zaprojektuj hierarchię systemu powiadomień składającą się z **co najmniej 3 klas** i **co najmniej jednego poziomu dziedziczenia**, wykorzystującą modyfikator `interface` oraz rekordy do zwracania wyniku wysyłki:

- `interface class Kanal` — kontrakt z metodą `({bool sukces, String szczegoly}) wyslij(String tresc)` (poziom 1).
- `class KanalBazowy implements Kanal` — abstrakcyjna baza dostarczająca wspólną walidację treści przez metodę `bool _poprawna(String tresc)` (poziom 2).
- `class KanalEmail extends KanalBazowy` oraz `class KanalSMS extends KanalBazowy` — konkretne kanały (poziom 3 — drugi poziom dziedziczenia).

Metoda `wyslij` zwraca **rekord** `(sukces: bool, szczegoly: String)`. Jeśli treść jest pusta, zwraca `(sukces: false, szczegoly: 'pusta treść')`. W przeciwnym razie `(sukces: true, szczegoly: '<nazwa kanału>: <tresc>')`.

**Oczekiwane relacje klas:** `KanalBazowy` implementuje interfejs `Kanal`; `KanalEmail` i `KanalSMS` dziedziczą po `KanalBazowy` (drugi poziom). SMS dodatkowo ogranicza długość treści do 160 znaków — dłuższa treść daje `(sukces: false, szczegoly: 'za długa')`.

**Oczekiwane zachowanie:** wywołanie `wyslij` na kanale zwraca rekord, który można destrukturyzować. Dla poprawnej treści `sukces` to `true`, a `szczegoly` zawiera nazwę kanału i treść. Poprawne rozwiązanie wykorzystuje strukturalną równość rekordów do porównania wyników.

**Poziom trudności:** advanced

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `KanalEmail().wyslij('Cześć')` | `(sukces: true, szczegoly: EMAIL: Cześć)` |
| `KanalSMS().wyslij('')` | `(sukces: false, szczegoly: pusta treść)` |

### Wskazówki

1. Zadeklaruj typ rekordu wyniku jako `({bool sukces, String szczegoly})` — użyj go jako typu zwracanego metody `wyslij`.
2. Wspólną walidację (pusta treść) umieść w `KanalBazowy`, aby podklasy jej nie powielały.
3. W `KanalSMS` nadpisz `wyslij`, dodając kontrolę długości, a następnie deleguj do implementacji bazowej dla pozostałych przypadków (np. przez `super.wyslij(...)`).
4. Rekordy porównuj operatorem `==` — mają równość strukturalną, więc `(sukces: true, szczegoly: 'x') == (sukces: true, szczegoly: 'x')` jest `true`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// Kontrakt kanału — interface pozwala na implements, nie na extends spoza biblioteki
interface class Kanal {
  ({bool sukces, String szczegoly}) wyslij(String tresc) =>
      (sukces: false, szczegoly: 'niezaimplementowane');
}

// Wspólna baza z walidacją — implementuje kontrakt Kanal
class KanalBazowy implements Kanal {
  String get nazwa => 'KANAL';

  bool _poprawna(String tresc) => tresc.isNotEmpty;

  @override
  ({bool sukces, String szczegoly}) wyslij(String tresc) {
    if (!_poprawna(tresc)) {
      return (sukces: false, szczegoly: 'pusta treść');
    }
    return (sukces: true, szczegoly: '$nazwa: $tresc');
  }
}

class KanalEmail extends KanalBazowy {
  @override
  String get nazwa => 'EMAIL';
}

class KanalSMS extends KanalBazowy {
  @override
  String get nazwa => 'SMS';

  @override
  ({bool sukces, String szczegoly}) wyslij(String tresc) {
    if (tresc.length > 160) {
      return (sukces: false, szczegoly: 'za długa');
    }
    return super.wyslij(tresc); // delegacja do walidacji i formatowania bazy
  }
}

void main() {
  final wynikEmail = KanalEmail().wyslij('Cześć');
  print(wynikEmail);

  final wynikPusty = KanalSMS().wyslij('');
  print(wynikPusty);

  // Wykorzystanie równości strukturalnej rekordów
  final oczekiwany = (sukces: true, szczegoly: 'EMAIL: Cześć');
  print('Zgodny z oczekiwanym: ${wynikEmail == oczekiwany}');

  // Destrukturyzacja rekordu wyniku
  final (:sukces, :szczegoly) = KanalSMS().wyslij('Test SMS');
  print('sukces=$sukces, szczegoly=$szczegoly');
}
// Oczekiwane wyjście:
// (sukces: true, szczegoly: EMAIL: Cześć)
// (sukces: false, szczegoly: pusta treść)
// Zgodny z oczekiwanym: true
// sukces=true, szczegoly=SMS: Test SMS
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`sealed`** tworzy zamkniętą hierarchię (wszystkie podtypy w jednym pliku), umożliwiając kompilatorowi sprawdzanie kompletności wyrażeń `switch` — pominięty podtyp to błąd kompilacji.
2. **`base`** wymusza dziedziczenie: typ można `extends`, ale nie `implements` spoza biblioteki — gwarantuje odziedziczenie implementacji i niezmienników.
3. **`interface`** wymusza implementację: typ można `implements`, ale nie `extends` spoza biblioteki — definiuje czysty kontrakt.
4. **`final`** zamyka typ całkowicie: ani `extends`, ani `implements` spoza biblioteki — bezpieczna ewolucja API.
5. **`mixin` i `mixin class`** rozdzielają role: `mixin` służy wyłącznie do `with` (z opcjonalnym ograniczeniem `on`), a `mixin class` pełni obie role — mixinu i zwykłej klasy.
6. **Rekordy** to niemutowalne agregaty z polami pozycyjnymi (`$1`, `$2`) i nazwanymi, obsługujące destrukturyzację oraz **strukturalną równość** — idealne do zwracania wielu wartości.

---

**Poprzedni moduł:** [Mixiny](../05-oop/03-mixins.md)
**Następny moduł:** [Klasy generyczne](../06-generics/01-generic-classes.md)
