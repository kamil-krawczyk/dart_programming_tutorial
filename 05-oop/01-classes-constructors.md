---
id: "5.1"
title: "Klasy i konstruktory"
difficulty: "intermediate"
section: "05-oop"
prerequisites:
  - "Deklaracje funkcji i parametry"
  - "Zmienne i typy danych"
---

# 5.1 Klasy i konstruktory

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Deklaracje funkcji i parametry](../04-functions/01-declarations-params.md), [Zmienne i typy danych](../01-basics/01-variables-types.md)
- **Cele nauki:**
  1. Deklarować klasy z polami instancji, metodami, getterami i setterami oraz rozumieć różnicę między składową instancji a statyczną
  2. Definiować i stosować wszystkie rodzaje konstruktorów: domyślny, nazwany, fabryczny (`factory`), stały (`const`) oraz przekierowujący (redirecting)
  3. Rozpoznawać typowe błędy związane z konstruktorami i składowymi klas oraz rozumieć komunikaty kompilatora
  4. Modelować encje domenowe (e-commerce, komponenty UI, przetwarzanie danych) przy pomocy dobrze zaprojektowanych klas

---

## Deklaracje klas

Klasa w Dart to szablon opisujący stan (pola instancji) i zachowanie (metody) obiektów. Każdy obiekt jest instancją dokładnie jednej klasy, a wszystkie klasy dziedziczą pośrednio po klasie `Object`.

### Podstawowa deklaracja klasy

Klasę deklaruje się słowem kluczowym `class`, po którym następuje nazwa (konwencja: `UpperCamelCase`) i ciało w nawiasach klamrowych.

```dart
// Klasa modelująca produkt w sklepie internetowym
class Produkt {
  // Pola instancji — przechowują stan każdego obiektu
  String nazwa;
  double cena;
  int stanMagazynowy;

  // Konstruktor generatywny z inicjalizacją skróconą (this.pole)
  Produkt(this.nazwa, this.cena, this.stanMagazynowy);

  // Metoda instancji — operuje na stanie obiektu
  bool czyDostepny() => stanMagazynowy > 0;
}

void main() {
  // Tworzenie instancji — słowo new jest opcjonalne od Dart 2
  var laptop = Produkt('Laptop', 3500.0, 5);
  print('${laptop.nazwa}: ${laptop.cena} zł');
  print('Dostępny: ${laptop.czyDostepny()}');
}
// Oczekiwane wyjście:
// Laptop: 3500.0 zł
// Dostępny: true
```

Kolejny przykład modeluje komponent UI — pola mogą mieć wartości domyślne i typy nullable:

```dart
// Klasa modelująca prosty przycisk w interfejsie użytkownika
class Przycisk {
  String etykieta;
  bool wlaczony;
  String? ikona; // pole nullable — ikona jest opcjonalna

  Przycisk(this.etykieta, {this.wlaczony = true, this.ikona});

  String opis() {
    var stan = wlaczony ? 'aktywny' : 'nieaktywny';
    var ikonaOpis = ikona != null ? ' [$ikona]' : '';
    return 'Przycisk "$etykieta"$ikonaOpis — $stan';
  }
}

void main() {
  var zapisz = Przycisk('Zapisz', ikona: 'dysk');
  var anuluj = Przycisk('Anuluj', wlaczony: false);
  print(zapisz.opis());
  print(anuluj.opis());
}
// Oczekiwane wyjście:
// Przycisk "Zapisz" [dysk] — aktywny
// Przycisk "Anuluj" — nieaktywny
```

---

## Pola instancji

Pola instancji przechowują stan każdego obiektu osobno. Mogą być modyfikowalne (`var`/typ), niemodyfikowalne po inicjalizacji (`final`) lub inicjalizowane leniwie (`late`).

```dart
// Rekord przetwarzania danych z polami o różnej modyfikowalności
class RekordPomiaru {
  final String czujnik;     // final — ustawiane raz, potem niezmienne
  double wartosc;           // modyfikowalne
  late final DateTime znacznikCzasu; // late final — inicjalizowane w ciele konstruktora

  RekordPomiaru(this.czujnik, this.wartosc) {
    // late final pozwala odłożyć inicjalizację do ciała konstruktora
    znacznikCzasu = DateTime(2024, 1, 1, 12, 0);
  }

  void skoryguj(double delta) {
    wartosc += delta; // dozwolone — pole nie jest final
    // czujnik = 'inny'; // BŁĄD gdyby odkomentować — final nie można zmienić
  }
}

void main() {
  var pomiar = RekordPomiaru('temp-01', 21.5);
  pomiar.skoryguj(1.5);
  print('${pomiar.czujnik}: ${pomiar.wartosc}°C @ ${pomiar.znacznikCzasu}');
}
// Oczekiwane wyjście:
// temp-01: 23.0°C @ 2024-01-01 12:00:00.000
```

Pola można też inicjalizować bezpośrednio przy deklaracji — przydatne dla wartości domyślnych niezależnych od argumentów konstruktora:

```dart
// Koszyk zakupowy z polem inicjalizowanym przy deklaracji
class Koszyk {
  // Inicjalizacja przy deklaracji — każdy koszyk startuje z pustą listą
  final List<String> pozycje = [];
  int _licznikDodan = 0; // pole prywatne — podkreślenie oznacza widoczność w bibliotece

  void dodaj(String pozycja) {
    pozycje.add(pozycja);
    _licznikDodan++;
  }

  int get liczbaOperacji => _licznikDodan;
}

void main() {
  var koszyk = Koszyk();
  koszyk.dodaj('Jabłka');
  koszyk.dodaj('Chleb');
  print('Pozycje: ${koszyk.pozycje}');
  print('Liczba operacji dodania: ${koszyk.liczbaOperacji}');
}
// Oczekiwane wyjście:
// Pozycje: [Jabłka, Chleb]
// Liczba operacji dodania: 2
```

---

## Konstruktor domyślny i inicjalizacja pól

Konstruktor to specjalna metoda o nazwie identycznej z klasą, tworząca i inicjalizująca obiekt. Jeśli nie zdefiniujesz żadnego konstruktora, Dart udostępnia niejawny konstruktor bezargumentowy.

### Składnia skrócona i lista inicjalizatorów

Zapis `this.pole` w liście parametrów automatycznie przypisuje argument do pola. Lista inicjalizatorów (po dwukropku) pozwala nadać wartości polom `final` przed uruchomieniem ciała konstruktora.

```dart
// Konstruktor z listą inicjalizatorów — oblicza pole final przed ciałem
class Prostokat {
  final double szerokosc;
  final double wysokosc;
  final double pole; // musi być zainicjalizowane w liście inicjalizatorów

  // Lista inicjalizatorów po ':' — wykonuje się przed ciałem konstruktora
  Prostokat(this.szerokosc, this.wysokosc) : pole = szerokosc * wysokosc {
    // Ciało konstruktora — tu można wykonać walidację lub efekty uboczne
    assert(szerokosc > 0 && wysokosc > 0, 'Wymiary muszą być dodatnie');
  }

  @override
  String toString() => 'Prostokat(${szerokosc}x$wysokosc, pole: $pole)';
}

void main() {
  var r = Prostokat(4.0, 3.0);
  print(r);
}
// Oczekiwane wyjście:
// Prostokat(4.0x3.0, pole: 12.0)
```

### Anti-pattern: próba przypisania do pola `final` w ciele konstruktora

Pola `final` muszą zostać zainicjalizowane w liście parametrów albo w liście inicjalizatorów — nie w ciele konstruktora. To częsty błąd początkujących.

```dart
// ❌ BŁĄD KOMPILACJI — nie kompiluje się
class BledneKonto {
  final double saldo;

  BledneKonto(double poczatkowe) {
    // saldo jest final — nie można przypisać w ciele konstruktora
    saldo = poczatkowe; // Błąd: 'saldo' can't be used as a setter because
                        // it's final.  Try finding a different setter, or
                        // making 'saldo' non-final.
  }
}
```

```dart
// ✅ POPRAWNIE — inicjalizacja w liście inicjalizatorów
class Konto {
  final double saldo;

  // Przypisanie w liście inicjalizatorów (przed ciałem) — dozwolone dla final
  Konto(double poczatkowe) : saldo = poczatkowe;

  // Albo jeszcze prościej, składnią skróconą:
  // Konto(this.saldo);
}

void main() {
  var konto = Konto(100.0);
  print('Saldo: ${konto.saldo}');
}
// Oczekiwane wyjście:
// Saldo: 100.0
```

---

## Konstruktory nazwane

Klasa może mieć wiele konstruktorów dzięki konstruktorom nazwanym o postaci `Klasa.nazwa(...)`. Pozwalają one na alternatywne sposoby tworzenia obiektu.

```dart
// Konstruktory nazwane dla różnych sposobów utworzenia koloru (komponent UI)
class Kolor {
  final int r;
  final int g;
  final int b;

  // Konstruktor główny (nienazwany)
  Kolor(this.r, this.g, this.b);

  // Konstruktor nazwany — kolor czarny
  Kolor.czarny() : r = 0, g = 0, b = 0;

  // Konstruktor nazwany — odcień szarości z jednej wartości
  Kolor.szarosc(int poziom) : r = poziom, g = poziom, b = poziom;

  @override
  String toString() => 'rgb($r, $g, $b)';
}

void main() {
  var czerwony = Kolor(255, 0, 0);
  var czarny = Kolor.czarny();
  var szary = Kolor.szarosc(128);
  print(czerwony);
  print(czarny);
  print(szary);
}
// Oczekiwane wyjście:
// rgb(255, 0, 0)
// rgb(0, 0, 0)
// rgb(128, 128, 128)
```

Konstruktory nazwane są szczególnie przydatne przy parsowaniu danych z różnych źródeł:

```dart
// Konstruktory nazwane do parsowania danych (przetwarzanie danych)
class Wspolrzedne {
  final double lat;
  final double lng;

  Wspolrzedne(this.lat, this.lng);

  // Parsowanie z formatu "lat,lng"
  Wspolrzedne.zTekstu(String tekst)
      : lat = double.parse(tekst.split(',')[0]),
        lng = double.parse(tekst.split(',')[1]);

  // Punkt zerowy (Zatoka Gwinejska)
  Wspolrzedne.zero()
      : lat = 0.0,
        lng = 0.0;

  @override
  String toString() => '($lat, $lng)';
}

void main() {
  var p1 = Wspolrzedne(52.23, 21.01);
  var p2 = Wspolrzedne.zTekstu('50.06,19.94');
  var p3 = Wspolrzedne.zero();
  print(p1);
  print(p2);
  print(p3);
}
// Oczekiwane wyjście:
// (52.23, 21.01)
// (50.06, 19.94)
// (0.0, 0.0)
```

---

## Konstruktory przekierowujące (redirecting)

Konstruktor przekierowujący deleguje pracę do innego konstruktora tej samej klasy. Ma puste ciało, a po dwukropku wywołuje `this(...)` lub `this.nazwa(...)`. Redukuje to duplikację logiki inicjalizacji.

```dart
// Konstruktory przekierowujące — jeden konstruktor deleguje do drugiego
class Zamowienie {
  final String id;
  final int liczbaSztuk;
  final bool ekspresowe;

  // Konstruktor główny — zawiera całą logikę
  Zamowienie(this.id, this.liczbaSztuk, this.ekspresowe);

  // Przekierowanie do konstruktora głównego z domyślnym liczbaSztuk = 1
  Zamowienie.pojedyncze(String id) : this(id, 1, false);

  // Przekierowanie z ustawieniem dostawy ekspresowej
  Zamowienie.ekspres(String id, int liczba) : this(id, liczba, true);

  @override
  String toString() =>
      'Zamowienie($id, sztuk: $liczbaSztuk, ekspres: $ekspresowe)';
}

void main() {
  print(Zamowienie('Z-1', 3, false));
  print(Zamowienie.pojedyncze('Z-2'));
  print(Zamowienie.ekspres('Z-3', 5));
}
// Oczekiwane wyjście:
// Zamowienie(Z-1, sztuk: 3, ekspres: false)
// Zamowienie(Z-2, sztuk: 1, ekspres: false)
// Zamowienie(Z-3, sztuk: 5, ekspres: true)
```

### Anti-pattern: konstruktor przekierowujący z ciałem lub inicjalizacją pól

Konstruktor przekierowujący nie może mieć ciała ani bezpośrednio inicjalizować pól — cała inicjalizacja musi odbyć się w konstruktorze docelowym.

```dart
// ❌ BŁĄD KOMPILACJI — konstruktor przekierowujący nie może inicjalizować pól
class BledneZamowienie {
  final String id;
  final int liczbaSztuk;

  BledneZamowienie(this.id, this.liczbaSztuk);

  // Nie można łączyć przekierowania (this(...)) z inicjalizacją pola
  BledneZamowienie.pojedyncze(String id)
      : liczbaSztuk = 1,        // Błąd: The redirecting constructor can't have
        this(id, 1);            // a field initializer.
}
```

```dart
// ✅ POPRAWNIE — cała inicjalizacja w konstruktorze docelowym
class PoprawneZamowienie {
  final String id;
  final int liczbaSztuk;

  PoprawneZamowienie(this.id, this.liczbaSztuk);

  // Przekierowanie przekazuje wartości — bez własnej inicjalizacji pól
  PoprawneZamowienie.pojedyncze(String id) : this(id, 1);
}

void main() {
  var z = PoprawneZamowienie.pojedyncze('Z-9');
  print('${z.id}: ${z.liczbaSztuk} szt.');
}
// Oczekiwane wyjście:
// Z-9: 1 szt.
```

---

## Konstruktory stałe (const)

Konstruktor `const` tworzy obiekty niezmienne (immutable) w czasie kompilacji. Wszystkie pola takiej klasy muszą być `final`. Dwie stałe instancje o tych samych wartościach są tym samym obiektem (kanonizacja).

```dart
// Konstruktor const dla niezmiennej wartości (idealne dla wartości UI/konfiguracji)
class Wymiar {
  final double szerokosc;
  final double wysokosc;

  // const wymaga, aby wszystkie pola były final
  const Wymiar(this.szerokosc, this.wysokosc);

  // Konstruktor const nazwany
  const Wymiar.kwadrat(double bok)
      : szerokosc = bok,
        wysokosc = bok;

  @override
  String toString() => '${szerokosc}x$wysokosc';
}

void main() {
  // Obiekty const są kanonizowane — identyczne wartości = ten sam obiekt
  const a = Wymiar(100, 50);
  const b = Wymiar(100, 50);
  print(identical(a, b)); // true — to ten sam obiekt w pamięci

  const kwadrat = Wymiar.kwadrat(64);
  print(kwadrat);
}
// Oczekiwane wyjście:
// true
// 64.0x64.0
```

Porównanie: obiekt utworzony z `const` vs bez `const`:

```dart
// Różnica między instancją const a zwykłą instancją tej samej klasy
class Punkt {
  final int x;
  final int y;
  const Punkt(this.x, this.y);
}

void main() {
  const p1 = Punkt(1, 2);
  const p2 = Punkt(1, 2);
  var p3 = Punkt(1, 2); // bez const — nowa instancja za każdym razem

  print(identical(p1, p2)); // true — const kanonizowane
  print(identical(p1, p3)); // false — p3 to osobny obiekt na stercie
  print(p1 == p3);          // false — domyślne == to identyczność referencji
}
// Oczekiwane wyjście:
// true
// false
// false
```

### Anti-pattern: `const` z polem niebędącym `final`

Klasa z konstruktorem `const` nie może mieć modyfikowalnych pól.

```dart
// ❌ BŁĄD KOMPILACJI — pole nie jest final w klasie z konstruktorem const
class BlednyPunkt {
  int x; // powinno być final
  final int y;

  const BlednyPunkt(this.x, this.y);
  // Błąd: Can't define a const constructor for a class with non-final fields.
}
```

```dart
// ✅ POPRAWNIE — wszystkie pola final
class DobryPunkt {
  final int x;
  final int y;
  const DobryPunkt(this.x, this.y);
}

void main() {
  const p = DobryPunkt(3, 4);
  print('(${p.x}, ${p.y})');
}
// Oczekiwane wyjście:
// (3, 4)
```

---

## Konstruktory fabryczne (factory)

Konstruktor `factory` nie musi tworzyć nowej instancji — może zwrócić istniejący obiekt (np. z cache), instancję podklasy lub obiekt utworzony po nietrywialnej logice. Wewnątrz `factory` używa się `return`.

```dart
// Factory z cache — te same identyfikatory zwracają tę samą instancję (przetwarzanie danych)
class Uzytkownik {
  final String id;
  final String nazwa;

  // Prywatny cache wspólny dla wszystkich instancji
  static final Map<String, Uzytkownik> _cache = {};

  // Prywatny konstruktor generatywny — nazwa z podkreśleniem
  Uzytkownik._(this.id, this.nazwa);

  // Factory decyduje: zwróć z cache lub utwórz nowy
  factory Uzytkownik(String id, String nazwa) {
    return _cache.putIfAbsent(id, () => Uzytkownik._(id, nazwa));
  }

  @override
  String toString() => 'Uzytkownik($id, $nazwa)';
}

void main() {
  var u1 = Uzytkownik('u1', 'Anna');
  var u2 = Uzytkownik('u1', 'Anna (duplikat)');
  print(identical(u1, u2)); // true — factory zwrócił obiekt z cache
  print(u2.nazwa);          // Anna — bo zwrócono obiekt z pierwszego wywołania
}
// Oczekiwane wyjście:
// true
// Anna
```

Częstym zastosowaniem factory jest parsowanie danych JSON na obiekt domenowy:

```dart
// Factory do budowy obiektu z mapy (typowy wzorzec fromJson)
class Artykul {
  final String tytul;
  final int liczbaSlow;

  Artykul(this.tytul, this.liczbaSlow);

  // Factory wykonuje logikę parsowania i walidacji przed utworzeniem obiektu
  factory Artykul.zMapy(Map<String, dynamic> dane) {
    var tytul = dane['tytul'] as String? ?? 'Bez tytułu';
    var tresc = dane['tresc'] as String? ?? '';
    var liczba = tresc.isEmpty ? 0 : tresc.split(' ').length;
    return Artykul(tytul, liczba);
  }

  @override
  String toString() => '"$tytul" ($liczbaSlow słów)';
}

void main() {
  var a = Artykul.zMapy({'tytul': 'Dart 3', 'tresc': 'Nowe funkcje języka Dart'});
  print(a);
}
// Oczekiwane wyjście:
// "Dart 3" (4 słów)
```

### Anti-pattern: użycie `this` w konstruktorze `factory`

W konstruktorze `factory` obiekt jeszcze nie istnieje — nie ma dostępu do `this`. Trzeba jawnie zwrócić instancję.

```dart
// ❌ BŁĄD KOMPILACJI — factory nie ma dostępu do this
class BlednaKonfiguracja {
  final int limit;
  BlednaKonfiguracja._(this.limit);

  factory BlednaKonfiguracja(int limit) {
    this.limit = limit; // Błąd: Invalid reference to 'this' expression.
                        // W factory nie istnieje jeszcze instancja obiektu.
  }
}
```

```dart
// ✅ POPRAWNIE — factory jawnie zwraca instancję
class Konfiguracja {
  final int limit;
  Konfiguracja._(this.limit);

  factory Konfiguracja(int limit) {
    // Walidacja i normalizacja, potem zwrot nowej instancji
    var bezpieczny = limit < 0 ? 0 : limit;
    return Konfiguracja._(bezpieczny);
  }
}

void main() {
  var k = Konfiguracja(-5);
  print('Limit: ${k.limit}');
}
// Oczekiwane wyjście:
// Limit: 0
```

---

## Metody

Metody definiują zachowanie obiektu i mają dostęp do jego pól przez `this` (zwykle niejawnie). Metody instancji działają na konkretnym obiekcie.

```dart
// Metody operujące na stanie obiektu (koszyk e-commerce)
class KoszykSklepu {
  final Map<String, double> _pozycje = {}; // nazwa -> cena

  // Metoda modyfikująca stan
  void dodajPozycje(String nazwa, double cena) {
    _pozycje[nazwa] = cena;
  }

  // Metoda odczytująca stan i zwracająca wynik
  double sumaCalkowita() {
    return _pozycje.values.fold(0.0, (suma, cena) => suma + cena);
  }

  // Metoda z parametrem — zastosowanie rabatu
  double sumaZRabatem(double procent) {
    return sumaCalkowita() * (1 - procent / 100);
  }
}

void main() {
  var koszyk = KoszykSklepu();
  koszyk.dodajPozycje('Laptop', 3500);
  koszyk.dodajPozycje('Mysz', 100);
  print('Suma: ${koszyk.sumaCalkowita()}');
  print('Po rabacie 10%: ${koszyk.sumaZRabatem(10)}');
}
// Oczekiwane wyjście:
// Suma: 3600.0
// Po rabacie 10%: 3240.0
```

Metody można też przeciążać operacje przez nadpisanie operatorów oraz metody z `Object`, np. `toString`:

```dart
// Nadpisanie toString i operatora == (przetwarzanie danych)
class Wektor {
  final double x;
  final double y;
  const Wektor(this.x, this.y);

  // Operator dodawania jako metoda
  Wektor operator +(Wektor inny) => Wektor(x + inny.x, y + inny.y);

  @override
  String toString() => 'Wektor($x, $y)';
}

void main() {
  var suma = const Wektor(1, 2) + const Wektor(3, 4);
  print(suma);
}
// Oczekiwane wyjście:
// Wektor(4.0, 6.0)
```

---

## Gettery i settery

Gettery i settery to specjalne metody dające dostęp do wartości tak, jakby były polami. Getter deklaruje się słowem `get`, setter słowem `set`. Umożliwiają walidację, wartości obliczane i enkapsulację.

```dart
// Getter obliczany i setter z walidacją (encja e-commerce)
class ProduktZCena {
  final String nazwa;
  double _cenaNetto; // pole prywatne

  ProduktZCena(this.nazwa, this._cenaNetto);

  // Getter obliczany — cena brutto z VAT 23%
  double get cenaBrutto => _cenaNetto * 1.23;

  // Getter zwracający pole prywatne
  double get cenaNetto => _cenaNetto;

  // Setter z walidacją — chroni przed ujemną ceną
  set cenaNetto(double wartosc) {
    if (wartosc < 0) {
      throw ArgumentError('Cena nie może być ujemna: $wartosc');
    }
    _cenaNetto = wartosc;
  }
}

void main() {
  var p = ProduktZCena('Klawiatura', 200);
  print('Netto: ${p.cenaNetto}, brutto: ${p.cenaBrutto}');

  p.cenaNetto = 250; // wywołuje setter — używamy jak zwykłego pola
  print('Nowe netto: ${p.cenaNetto}, brutto: ${p.cenaBrutto}');
}
// Oczekiwane wyjście:
// Netto: 200.0, brutto: 246.0
// Nowe netto: 250.0, brutto: 307.5
```

Gettery są przydatne do wystawiania stanu tylko do odczytu:

```dart
// Getter tylko do odczytu — enkapsulacja stanu wewnętrznego (komponent UI)
class LicznikKlikniec {
  int _liczba = 0;

  // Brak settera — z zewnątrz nie można ustawić dowolnej wartości
  int get liczba => _liczba;

  // Getter obliczany na podstawie stanu
  bool get przekroczonoLimit => _liczba > 10;

  void kliknij() => _liczba++;
}

void main() {
  var licznik = LicznikKlikniec();
  for (var i = 0; i < 12; i++) {
    licznik.kliknij();
  }
  print('Kliknięcia: ${licznik.liczba}');
  print('Przekroczono limit: ${licznik.przekroczonoLimit}');
}
// Oczekiwane wyjście:
// Kliknięcia: 12
// Przekroczono limit: true
```

### Anti-pattern: setter zmieniający pole `final`

Nie można napisać settera dla pola `final` — po inicjalizacji jest niezmienne.

```dart
// ❌ BŁĄD KOMPILACJI — próba modyfikacji pola final w setterze
class BlednyRabat {
  final double procent;
  BlednyRabat(this.procent);

  set nowyProcent(double wartosc) {
    procent = wartosc; // Błąd: 'procent' can't be used as a setter
                       // because it's final.
  }
}
```

```dart
// ✅ POPRAWNIE — pole modyfikowalne z prywatnym backingiem i walidacją
class Rabat {
  double _procent;
  Rabat(this._procent);

  double get procent => _procent;

  set procent(double wartosc) {
    _procent = wartosc.clamp(0, 100); // ogranicz do zakresu 0–100
  }
}

void main() {
  var r = Rabat(10);
  r.procent = 150; // zostanie ograniczone do 100
  print('Procent: ${r.procent}');
}
// Oczekiwane wyjście:
// Procent: 100.0
```

---

## Składowe statyczne

Składowe statyczne (`static`) należą do klasy, a nie do konkretnej instancji. Statyczne pola są współdzielone przez wszystkie obiekty, a statyczne metody wywołuje się na klasie.

```dart
// Pola i metody statyczne — współdzielone przez wszystkie instancje
class Faktura {
  static int _licznik = 0;        // wspólny licznik dla wszystkich faktur
  static const double stawkaVat = 0.23; // stała statyczna

  final int numer;
  final double kwotaNetto;

  Faktura(this.kwotaNetto) : numer = ++_licznik;

  // Metoda statyczna — nie ma dostępu do pól instancji, tylko do statycznych
  static double obliczVat(double netto) => netto * stawkaVat;

  // Getter statyczny — ile faktur utworzono
  static int get liczbaFaktur => _licznik;

  double get kwotaBrutto => kwotaNetto + obliczVat(kwotaNetto);
}

void main() {
  var f1 = Faktura(1000);
  var f2 = Faktura(2000);
  print('Faktura #${f1.numer}: brutto ${f1.kwotaBrutto}');
  print('Faktura #${f2.numer}: brutto ${f2.kwotaBrutto}');
  print('Łącznie faktur: ${Faktura.liczbaFaktur}'); // dostęp przez nazwę klasy
  print('VAT od 500: ${Faktura.obliczVat(500)}');
}
// Oczekiwane wyjście:
// Faktura #1: brutto 1230.0
// Faktura #2: brutto 2460.0
// Łącznie faktur: 2
// VAT od 500: 115.0
```

Stałe statyczne są częstym sposobem grupowania powiązanych wartości konfiguracyjnych:

```dart
// Stałe statyczne jako zgrupowana konfiguracja (komponent UI)
class Motyw {
  static const String kolorPodstawowy = '#2196F3';
  static const String kolorAkcentu = '#FF4081';
  static const double promienNarozy = 8.0;

  // Prywatny konstruktor uniemożliwia tworzenie instancji klasy-narzędzia
  Motyw._();

  static String opisMotywu() =>
      'Podstawowy: $kolorPodstawowy, akcent: $kolorAkcentu';
}

void main() {
  print(Motyw.opisMotywu());
  print('Promień naroży: ${Motyw.promienNarozy}');
}
// Oczekiwane wyjście:
// Podstawowy: #2196F3, akcent: #FF4081
// Promień naroży: 8.0
```

### Anti-pattern: dostęp do składowej statycznej przez instancję lub do `this` w metodzie statycznej

Składowych statycznych nie wywołuje się przez instancję, a metody statyczne nie mają dostępu do `this` ani do pól instancji.

```dart
// ❌ BŁĄD KOMPILACJI — metoda statyczna nie ma dostępu do pól instancji
class BlednyLicznik {
  int wartosc = 0;

  static int podwoj() {
    return wartosc * 2; // Błąd: The instance member 'wartosc' can't be
                        // accessed in a static method.
  }
}
```

```dart
// ✅ POPRAWNIE — metoda statyczna przyjmuje dane jako parametr
class Kalkulator {
  // Metoda statyczna operuje wyłącznie na argumentach i składowych statycznych
  static int podwoj(int wartosc) => wartosc * 2;
}

void main() {
  // Wywołanie przez nazwę klasy — nie przez instancję
  print(Kalkulator.podwoj(21));
}
// Oczekiwane wyjście:
// 42
```

---

## Ćwiczenie 1

### Opis problemu

Zamodeluj klasę `Konto` reprezentującą konto bankowe. Klasa powinna mieć: prywatne pole `_saldo` (domyślnie 0), publiczny getter `saldo`, metodę `wplac(double kwota)` oraz metodę `wyplac(double kwota)`, która zwraca `true` gdy wypłata się powiedzie (dość środków) lub `false` w przeciwnym razie. Dodaj statyczne pole `stopaOprocentowania` (domyślnie `0.02`) oraz metodę statyczną `oblicznOdsetki(double saldo)` zwracającą odsetki. Wpłata i wypłata kwoty ujemnej powinny być ignorowane (brak zmiany salda).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `wplac(100)`, potem `saldo` | `100.0` |
| `wplac(100)`, `wyplac(150)` (zwrot), potem `saldo` | `false`, `100.0` |
| `Konto.oblicznOdsetki(1000)` | `20.0` |

### Wskazówki

1. Getter `saldo` powinien zwracać `_saldo`, ale nie powinno być publicznego settera
2. W `wyplac` najpierw sprawdź, czy kwota jest dodatnia i czy jest wystarczające saldo
3. Metoda statyczna korzysta ze statycznego pola `stopaOprocentowania`, nie z pól instancji

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Konto {
  static double stopaOprocentowania = 0.02;

  double _saldo = 0;

  double get saldo => _saldo;

  void wplac(double kwota) {
    if (kwota <= 0) return; // ignoruj kwoty niedodatnie
    _saldo += kwota;
  }

  bool wyplac(double kwota) {
    if (kwota <= 0) return false;
    if (kwota > _saldo) return false; // za mało środków
    _saldo -= kwota;
    return true;
  }

  static double oblicznOdsetki(double saldo) => saldo * stopaOprocentowania;
}

void main() {
  var konto = Konto();
  konto.wplac(100);
  print(konto.saldo); // 100.0

  print(konto.wyplac(150)); // false — za mało środków
  print(konto.saldo);       // 100.0 — bez zmiany

  print(Konto.oblicznOdsetki(1000)); // 20.0
}
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Zaprojektuj klasę `Temperatura` przechowującą temperaturę wewnętrznie w stopniach Celsjusza (`final double celsjusz`). Dodaj:
1. Główny konstruktor przyjmujący wartość w Celsjuszu
2. Konstruktor nazwany `Temperatura.zFahrenheita(double f)` przeliczający `(f - 32) * 5 / 9`
3. Konstruktor przekierowujący `Temperatura.zamarzanie()` ustawiający 0°C (przekieruj do konstruktora głównego)
4. Getter `fahrenheit` zwracający `celsjusz * 9 / 5 + 32`
5. Konstruktor `const`, tak aby dało się tworzyć stałe temperatury

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Temperatura(100).fahrenheit` | `212.0` |
| `Temperatura.zFahrenheita(32).celsjusz` | `0.0` |
| `Temperatura.zamarzanie().celsjusz` | `0.0` |

### Wskazówki

1. Aby konstruktor był `const`, wszystkie pola muszą być `final` — przelicznik w konstruktorze nazwanym umieść w liście inicjalizatorów
2. Konstruktor przekierowujący ma pustą treść i wywołuje `this(...)` po dwukropku
3. Getter `fahrenheit` to prosty getter obliczany ze składnią strzałkową

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Temperatura {
  final double celsjusz;

  // Konstruktor główny — const, bo pole jest final
  const Temperatura(this.celsjusz);

  // Konstruktor nazwany z przelicznikiem w liście inicjalizatorów
  const Temperatura.zFahrenheita(double f) : celsjusz = (f - 32) * 5 / 9;

  // Konstruktor przekierowujący do głównego
  const Temperatura.zamarzanie() : this(0);

  // Getter obliczany
  double get fahrenheit => celsjusz * 9 / 5 + 32;
}

void main() {
  print(Temperatura(100).fahrenheit);          // 212.0
  print(Temperatura.zFahrenheita(32).celsjusz); // 0.0
  print(Temperatura.zamarzanie().celsjusz);      // 0.0
}
```

</details>

---

## Ćwiczenie 3

### Opis problemu

Zaimplementuj klasę `RejestrProduktow` używającą konstruktora `factory` z cache. Klasa `Produkt` ma pola `final String kod` oraz `final String nazwa`. Konstruktor `factory Produkt(String kod, String nazwa)` powinien zwracać zawsze tę samą instancję dla danego `kod` (pierwsza rejestracja wygrywa). Dodaj statyczny getter `Produkt.liczbaZarejestrowanych` zwracający liczbę unikalnych produktów w cache.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `identical(Produkt('A', 'Alfa'), Produkt('A', 'Inna'))` | `true` |
| `Produkt('A', 'Alfa').nazwa` po `Produkt('A', 'Inna')` | `Alfa` |
| po utworzeniu `A`, `B`, `A` → `Produkt.liczbaZarejestrowanych` | `2` |

### Wskazówki

1. Użyj prywatnego konstruktora generatywnego `Produkt._(this.kod, this.nazwa)` i publicznego `factory`
2. Statyczna mapa `Map<String, Produkt>` posłuży jako cache — klucz to `kod`
3. `putIfAbsent` zwraca istniejącą wartość lub tworzy nową tylko gdy klucz nie istnieje

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Produkt {
  static final Map<String, Produkt> _cache = {};

  final String kod;
  final String nazwa;

  // Prywatny konstruktor generatywny
  Produkt._(this.kod, this.nazwa);

  // Factory z cache — pierwsza rejestracja danego kodu wygrywa
  factory Produkt(String kod, String nazwa) {
    return _cache.putIfAbsent(kod, () => Produkt._(kod, nazwa));
  }

  static int get liczbaZarejestrowanych => _cache.length;
}

void main() {
  var p1 = Produkt('A', 'Alfa');
  var p2 = Produkt('A', 'Inna');
  print(identical(p1, p2)); // true — ta sama instancja z cache
  print(p2.nazwa);          // Alfa — pierwsza rejestracja wygrała

  Produkt('B', 'Beta');
  Produkt('A', 'Kolejna');  // kod A już istnieje — brak nowej instancji
  print(Produkt.liczbaZarejestrowanych); // 2
}
```

</details>

---

**Następny moduł:** [Dziedziczenie](02-inheritance.md)
**Poprzedni moduł:** [Generatory](../04-functions/04-generators.md)
