---
id: "4.1"
title: "Deklaracje funkcji i parametry"
difficulty: "intermediate"
section: "04-functions"
prerequisites:
  - "Zmienne i typy danych"
  - "Operatory"
---

# 4.1 Deklaracje funkcji i parametry

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Zmienne i typy danych](../01-basics/01-variables-types.md), [Operatory](../01-basics/02-operators.md)
- **Cele nauki:**
  1. Deklarować funkcje z jawnym typem zwracanym i stosować składnię strzałkową (`=>`)
  2. Rozumieć różnicę między parametrami nazwanymi a pozycyjnymi i wiedzieć kiedy stosować każdy rodzaj
  3. Definiować parametry opcjonalne (pozycyjne i nazwane) z wartościami domyślnymi oraz parametry wymagane (`required`)
  4. Dobierać odpowiedni styl parametrów do kontekstu API

---

## Deklaracje funkcji

W Dart funkcje są pełnoprawnymi obiektami — można je przypisywać do zmiennych, przekazywać jako argumenty i zwracać z innych funkcji. Każda funkcja ma typ zwracany (jawny lub wnioskowany).

### Podstawowa deklaracja

Funkcje deklaruje się podając typ zwracany, nazwę, listę parametrów i ciało w nawiasach klamrowych.

```dart
// Jawny typ zwracany — int
int dodaj(int a, int b) {
  return a + b;
}

// Dart może wnioskować typ zwracany, ale jawna adnotacja jest zalecana w API publicznym
String powitanie(String imie) {
  return 'Cześć, $imie!';
}

void main() {
  print(dodaj(3, 5));         // wywołanie z argumentami pozycyjnymi
  print(powitanie('Anna'));
}
// Oczekiwane wyjście:
// 8
// Cześć, Anna!
```

Funkcje zwracające `void` nie zwracają wartości — służą do wykonywania efektów ubocznych:

```dart
// void oznacza brak wartości zwracanej
void wyswietlSeparator(int szerokosc) {
  print('-' * szerokosc);
}

// Funkcja bez jawnego typu — Dart wnioskuje typ zwracany
pomnoz(int x, int y) {
  return x * y; // wnioskowany typ zwracany: int
}

void main() {
  wyswietlSeparator(20);
  print(pomnoz(4, 7));
}
// Oczekiwane wyjście:
// --------------------
// 28
```

---

## Składnia strzałkowa (=>)

Gdy ciało funkcji składa się z pojedynczego wyrażenia, można użyć skróconej notacji strzałkowej `=>`. Jest to równoważne z `{ return wyrażenie; }`.

```dart
// Składnia strzałkowa — zwięzła forma dla pojedynczych wyrażeń
int kwadrat(int n) => n * n;

String formatujCene(double cena) => '${cena.toStringAsFixed(2)} zł';

bool jestParzysta(int liczba) => liczba % 2 == 0;

void main() {
  print(kwadrat(5));
  print(formatujCene(19.9));
  print(jestParzysta(7));
}
// Oczekiwane wyjście:
// 25
// 19.90 zł
// false
```

Składnia strzałkowa działa również z funkcjami `void` — wyrażenie jest wykonywane, ale wartość jest odrzucana:

```dart
// Strzałka z void — wyrażenie jest wykonywane jako efekt
void przywitaj(String imie) => print('Witaj, $imie!');

// Porównanie tradycyjnej i strzałkowej składni
int maxTraditional(int a, int b) {
  if (a > b) return a;
  return b;
}

// Operator warunkowy pozwala użyć strzałki nawet z logiką
int maxArrow(int a, int b) => a > b ? a : b;

void main() {
  przywitaj('Dart');
  print(maxTraditional(10, 7));
  print(maxArrow(3, 9));
}
// Oczekiwane wyjście:
// Witaj, Dart!
// 10
// 9
```

---

## Parametry nazwane

Parametry nazwane są otoczone nawiasami klamrowymi `{}`. Przy wywołaniu podaje się ich nazwę, co zwiększa czytelność kodu. Domyślnie parametry nazwane są opcjonalne (nullable lub z wartością domyślną), chyba że oznaczono je `required`.

### Opcjonalne parametry nazwane z wartościami domyślnymi

```dart
// Parametry nazwane w {} — opcjonalne, z wartościami domyślnymi
void konfigurujSerwer({String host = 'localhost', int port = 8080, bool ssl = false}) {
  var protokol = ssl ? 'https' : 'http';
  print('$protokol://$host:$port');
}

void main() {
  konfigurujSerwer();                          // wszystkie domyślne
  konfigurujSerwer(port: 3000);                // tylko port zmieniony
  konfigurujSerwer(host: '0.0.0.0', ssl: true); // host i ssl zmienione
}
// Oczekiwane wyjście:
// http://localhost:8080
// http://localhost:3000
// https://0.0.0.0:8080
```

Parametry nazwane bez wartości domyślnej muszą być typu nullable:

```dart
// Parametr nazwany nullable — bez wartości domyślnej przyjmuje null
String budujUrl({String? sciezka, String? query}) {
  var url = 'https://api.example.com';
  if (sciezka != null) url += '/$sciezka';
  if (query != null) url += '?q=$query';
  return url;
}

void main() {
  print(budujUrl());
  print(budujUrl(sciezka: 'users'));
  print(budujUrl(sciezka: 'search', query: 'dart'));
}
// Oczekiwane wyjście:
// https://api.example.com
// https://api.example.com/users
// https://api.example.com/search?q=dart
```

### Wymagane parametry nazwane (`required`)

Słowo kluczowe `required` wymusza podanie parametru nazwanego przy wywołaniu — bez niego kod się nie skompiluje.

```dart
// required wymusza podanie parametru — bez niego jest błąd kompilacji
class Uzytkownik {
  final String imie;
  final String email;
  final int? wiek;

  Uzytkownik({
    required this.imie,   // musi być podany
    required this.email,  // musi być podany
    this.wiek,            // opcjonalny — nullable bez required
  });

  @override
  String toString() => '$imie ($email)${wiek != null ? ", wiek: $wiek" : ""}';
}

void main() {
  var u1 = Uzytkownik(imie: 'Anna', email: 'anna@example.com');
  var u2 = Uzytkownik(imie: 'Jan', email: 'jan@example.com', wiek: 30);

  // Poniższe nie skompiluje się — brak required parametru:
  // var u3 = Uzytkownik(imie: 'Ewa'); // Błąd: The named parameter 'email' is required

  print(u1);
  print(u2);
}
// Oczekiwane wyjście:
// Anna (anna@example.com)
// Jan (jan@example.com), wiek: 30
```

Kolejny przykład demonstrujący `required` w zwykłej funkcji:

```dart
// required w funkcji — idealne gdy parametr jest niezbędny, ale nazwa zwiększa czytelność
double obliczRabat({
  required double cena,
  required double procentRabatu,
  bool zaokraglij = true,
}) {
  var rabat = cena * (procentRabatu / 100);
  return zaokraglij ? (rabat * 100).roundToDouble() / 100 : rabat;
}

void main() {
  print(obliczRabat(cena: 199.99, procentRabatu: 15));
  print(obliczRabat(cena: 50.0, procentRabatu: 10, zaokraglij: false));
}
// Oczekiwane wyjście:
// 30.0
// 5.0
```

---

## Opcjonalne parametry pozycyjne

Parametry pozycyjne otoczone nawiasami kwadratowymi `[]` stają się opcjonalne. Mogą mieć wartości domyślne; bez nich są `null` (muszą być nullable).

```dart
// Parametry w [] są opcjonalne i pozycyjne
String formatujNazwe(String imie, [String? drugieImie, String? nazwisko]) {
  var czesci = [imie];
  if (drugieImie != null) czesci.add(drugieImie);
  if (nazwisko != null) czesci.add(nazwisko);
  return czesci.join(' ');
}

void main() {
  print(formatujNazwe('Anna'));
  print(formatujNazwe('Anna', 'Maria'));
  print(formatujNazwe('Anna', 'Maria', 'Kowalska'));
}
// Oczekiwane wyjście:
// Anna
// Anna Maria
// Anna Maria Kowalska
```

Opcjonalne parametry pozycyjne z wartościami domyślnymi:

```dart
// Opcjonalne pozycyjne z wartościami domyślnymi
String powtorz(String tekst, [int razy = 1, String separator = ' ']) {
  return List.filled(razy, tekst).join(separator);
}

// Uwaga: opcjonalne pozycyjne muszą być na końcu listy parametrów
int suma(int a, int b, [int c = 0, int d = 0]) {
  return a + b + c + d;
}

void main() {
  print(powtorz('ha'));
  print(powtorz('ha', 3));
  print(powtorz('ha', 3, '-'));
  print(suma(1, 2));
  print(suma(1, 2, 3, 4));
}
// Oczekiwane wyjście:
// ha
// ha ha ha
// ha-ha-ha
// 3
// 10
```

---

## Wartości domyślne

Wartości domyślne muszą być stałymi czasu kompilacji. Mogą być stosowane zarówno z parametrami nazwanymi, jak i opcjonalnymi pozycyjnymi.

```dart
// Wartości domyślne muszą być stałymi czasu kompilacji (const)
void loguj(
  String wiadomosc, {
  String poziom = 'INFO',          // string literal — const
  DateTime? czas,                  // null jako domyślna wartość
  List<String> tagi = const [],    // const lista jako wartość domyślna
}) {
  var timestamp = czas ?? DateTime.now();
  print('[$poziom] $timestamp: $wiadomosc ${tagi.isNotEmpty ? tagi : ""}');
}

void main() {
  loguj('Start aplikacji');
  loguj('Połączenie nieudane', poziom: 'ERROR', tagi: ['network', 'critical']);
}
// Oczekiwane wyjście:
// [INFO] (bieżąca data): Start aplikacji []
// [ERROR] (bieżąca data): Połączenie nieudane [network, critical]
```

Wzorzec z nullable parametrem i obliczoną wartością domyślną w ciele funkcji:

```dart
// Gdy wartość domyślna nie jest const — używamy null + obliczenie w ciele
List<int> generujLiczby(int ile, {int? startOd}) {
  // Nie można napisać: startOd = DateTime.now().millisecond — to nie const
  // Zamiast tego: nullable parametr + domyślna logika w ciele
  var start = startOd ?? 0;
  return List.generate(ile, (i) => start + i);
}

void main() {
  print(generujLiczby(5));
  print(generujLiczby(3, startOd: 10));
}
// Oczekiwane wyjście:
// [0, 1, 2, 3, 4]
// [10, 11, 12]
```

---

## Porównanie parametrów nazwanych i pozycyjnych

Wybór między parametrami nazwanymi a pozycyjnymi zależy od czytelności, liczby parametrów i kontekstu API.

### Kiedy stosować parametry pozycyjne

Parametry pozycyjne sprawdzają się gdy:
- Funkcja ma 1–2 parametry o oczywistym znaczeniu
- Kolejność jest naturalna i intuicyjna (np. `x, y` w geometrii)
- To operacja matematyczna lub transformacja

```dart
// Pozycyjne — znaczenie parametrów jest oczywiste z kontekstu
double pole(double szerokosc, double wysokosc) => szerokosc * wysokosc;

// Pozycyjne — naturalna kolejność operandów
int potega(int baza, int wykladnik) {
  var wynik = 1;
  for (var i = 0; i < wykladnik; i++) {
    wynik *= baza;
  }
  return wynik;
}

// Opcjonalne pozycyjne — prosty interfejs z sensownym domyślnym
String obetnij(String tekst, [int maxDlugosc = 50, String sufiks = '...']) {
  if (tekst.length <= maxDlugosc) return tekst;
  return '${tekst.substring(0, maxDlugosc)}$sufiks';
}

void main() {
  print(pole(5.0, 3.0));
  print(potega(2, 10));
  print(obetnij('Bardzo długi tekst do przycięcia', 15));
}
// Oczekiwane wyjście:
// 15.0
// 1024
// Bardzo długi te...
```

### Kiedy stosować parametry nazwane

Parametry nazwane sprawdzają się gdy:
- Funkcja przyjmuje więcej niż 2–3 parametry
- Znaczenie parametrów nie jest oczywiste z samej wartości (np. `true` — co to oznacza?)
- To konfiguracja lub konstruktor z wieloma opcjami
- Kilka parametrów jest opcjonalnych

```dart
// Nazwane — wiele parametrów konfiguracyjnych, flagi bool
Widget stworzPrzycisk({
  required String tekst,
  required void Function() onTap,
  double szerokosc = 200,
  double wysokosc = 48,
  bool zaokraglony = true,
  bool wylaczony = false,
}) {
  // Przy wywołaniu jasno widać co oznacza każdy parametr
  return Widget(tekst);
}

class Widget {
  final String label;
  Widget(this.label);
}

void main() {
  // Nazwane parametry — czytelne wywołanie
  stworzPrzycisk(
    tekst: 'Zapisz',
    onTap: () => print('Zapisano!'),
    zaokraglony: true,  // jasne znaczenie — bez nazwy byłoby: true — co?
    wylaczony: false,
  );
  print('Przycisk utworzony');
}
// Oczekiwane wyjście:
// Przycisk utworzony
```

### Porównanie bezpośrednie

Poniższy przykład pokazuje tę samą operację zaimplementowaną dwoma sposobami:

```dart
// --- Podejście 1: parametry pozycyjne ---
// Problem: co oznacza 10? A true? Trzeba sprawdzić sygnaturę
String formatV1(String tekst, int maxLength, bool uppercase, String suffix) {
  var wynik = tekst.length > maxLength
      ? tekst.substring(0, maxLength) + suffix
      : tekst;
  return uppercase ? wynik.toUpperCase() : wynik;
}

// --- Podejście 2: parametry nazwane ---
// Zaleta: wywołanie jest samodokumentujące
String formatV2(
  String tekst, {
  int maxLength = 100,
  bool uppercase = false,
  String suffix = '...',
}) {
  var wynik = tekst.length > maxLength
      ? tekst.substring(0, maxLength) + suffix
      : tekst;
  return uppercase ? wynik.toUpperCase() : wynik;
}

void main() {
  // Pozycyjne — co oznacza 10, true, '...'? Nieczytelne bez kontekstu
  print(formatV1('Hello World', 5, true, '...'));

  // Nazwane — intencja jest jasna w miejscu wywołania
  print(formatV2('Hello World', maxLength: 5, uppercase: true));
}
// Oczekiwane wyjście:
// HELLO...
// HELLO...
```

### Tabela porównawcza

| Cecha | Pozycyjne | Nazwane |
|-------|-----------|---------|
| Składnia deklaracji | `int f(int a, int b)` | `int f({required int a, int b = 0})` |
| Składnia wywołania | `f(1, 2)` | `f(a: 1, b: 2)` |
| Kolejność przy wywołaniu | Musi być zachowana | Dowolna |
| Opcjonalność | `[int a = 0]` | Domyślnie opcjonalne, `required` wymusza |
| Czytelność przy >3 param. | Niska | Wysoka |
| Idealne zastosowanie | Matematyka, proste transformacje | Konfiguracja, konstruktory, API |

---

## Ćwiczenie 1

### Opis problemu

Napisz funkcję `tworzAdres` przyjmującą parametry nazwane: `ulica` (wymagany), `numer` (wymagany), `mieszkanie` (opcjonalny), `miasto` (domyślnie `'Warszawa'`), `kodPocztowy` (opcjonalny). Funkcja powinna zwrócić sformatowany adres. Jeśli podano `kodPocztowy`, dodaj go przed miastem. Jeśli podano `mieszkanie`, dopisz je po numerze z ukośnikiem.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `ulica: 'Marszałkowska', numer: 10` | `Marszałkowska 10, Warszawa` |
| `ulica: 'Długa', numer: 5, mieszkanie: '3A', miasto: 'Kraków', kodPocztowy: '31-000'` | `Długa 5/3A, 31-000 Kraków` |
| `ulica: 'Nowa', numer: 22, kodPocztowy: '00-001'` | `Nowa 22, 00-001 Warszawa` |

### Wskazówki

1. Użyj `required` dla parametrów, które zawsze muszą być podane
2. Parametry opcjonalne typu `String?` mają domyślnie wartość `null` — sprawdź je przed użyciem
3. Buduj string stopniowo, sprawdzając które opcjonalne parametry zostały podane

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String tworzAdres({
  required String ulica,
  required int numer,
  String? mieszkanie,
  String miasto = 'Warszawa',
  String? kodPocztowy,
}) {
  var adres = '$ulica $numer';
  if (mieszkanie != null) adres += '/$mieszkanie';
  adres += ', ';
  if (kodPocztowy != null) adres += '$kodPocztowy ';
  adres += miasto;
  return adres;
}

void main() {
  print(tworzAdres(ulica: 'Marszałkowska', numer: 10));
  print(tworzAdres(
    ulica: 'Długa',
    numer: 5,
    mieszkanie: '3A',
    miasto: 'Kraków',
    kodPocztowy: '31-000',
  ));
  print(tworzAdres(ulica: 'Nowa', numer: 22, kodPocztowy: '00-001'));
}
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Napisz trzy wersje funkcji `podsumowanie`, która przyjmuje listę liczb i zwraca string z podsumowaniem statystycznym (suma, średnia, min, max). Zaimplementuj:
1. `podsumowanieV1` — z opcjonalnymi parametrami pozycyjnymi kontrolującymi czy pokazywać min/max (domyślnie tak) i liczbę miejsc po przecinku (domyślnie 2)
2. `podsumowanieV2` — z parametrami nazwanymi o tej samej funkcjonalności
3. W `main()` wywołaj obie wersje z tymi samymi danymi i porównaj czytelność wywołań

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `[10, 20, 30, 40, 50]`, pokazMin/Max: `true`, miejsca: `1` | `Suma: 150.0, Średnia: 30.0, Min: 10.0, Max: 50.0` |
| `[3, 7, 11]`, pokazMin/Max: `false`, miejsca: `2` | `Suma: 21.00, Średnia: 7.00` |

### Wskazówki

1. Użyj `reduce` lub `fold` do obliczenia sumy, a `reduce` z porównaniem do znalezienia min/max
2. Metoda `toStringAsFixed(n)` formatuje liczbę do `n` miejsc po przecinku
3. Porównaj jak wygląda wywołanie z `true, 1` (pozycyjne) vs `pokazMinMax: true, miejscaDecymalne: 1` (nazwane)

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:math' as math;

// Wersja 1: opcjonalne parametry pozycyjne
String podsumowanieV1(List<double> liczby, [bool pokazMinMax = true, int miejsca = 2]) {
  var suma = liczby.fold(0.0, (a, b) => a + b);
  var srednia = suma / liczby.length;
  var wynik = 'Suma: ${suma.toStringAsFixed(miejsca)}, '
      'Średnia: ${srednia.toStringAsFixed(miejsca)}';
  if (pokazMinMax) {
    var min = liczby.reduce(math.min);
    var max = liczby.reduce(math.max);
    wynik += ', Min: ${min.toStringAsFixed(miejsca)}, Max: ${max.toStringAsFixed(miejsca)}';
  }
  return wynik;
}

// Wersja 2: parametry nazwane — bardziej czytelne wywołanie
String podsumowanieV2(List<double> liczby, {bool pokazMinMax = true, int miejscaDecymalne = 2}) {
  var suma = liczby.fold(0.0, (a, b) => a + b);
  var srednia = suma / liczby.length;
  var wynik = 'Suma: ${suma.toStringAsFixed(miejscaDecymalne)}, '
      'Średnia: ${srednia.toStringAsFixed(miejscaDecymalne)}';
  if (pokazMinMax) {
    var min = liczby.reduce(math.min);
    var max = liczby.reduce(math.max);
    wynik += ', Min: ${min.toStringAsFixed(miejscaDecymalne)}, '
        'Max: ${max.toStringAsFixed(miejscaDecymalne)}';
  }
  return wynik;
}

void main() {
  var dane = [10.0, 20.0, 30.0, 40.0, 50.0];

  // Pozycyjne — co oznacza true i 1? Bez kontekstu nieczytelne
  print(podsumowanieV1(dane, true, 1));

  // Nazwane — intencja jasna w miejscu wywołania
  print(podsumowanieV2(dane, pokazMinMax: true, miejscaDecymalne: 1));

  // Bez min/max
  print(podsumowanieV2([3.0, 7.0, 11.0], pokazMinMax: false));
}
```

</details>

---

## Ćwiczenie 3

### Opis problemu

Napisz funkcję `filtrujProdukty` przyjmującą listę produktów (jako `List<Map<String, dynamic>>`) oraz opcjonalne filtry jako parametry nazwane: `minCena`, `maxCena`, `kategoria`, `tylkoDostepne` (domyślnie `true`). Funkcja powinna zwrócić przefiltrowaną listę. Użyj składni strzałkowej tam, gdzie to możliwe dla pomocniczych predykatów.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| produkty: `[{'nazwa': 'Laptop', 'cena': 3500, 'kategoria': 'elektronika', 'dostepny': true}, {'nazwa': 'Mysz', 'cena': 50, 'kategoria': 'elektronika', 'dostepny': false}]`, `minCena: 100` | `[{nazwa: Laptop, cena: 3500, kategoria: elektronika, dostepny: true}]` |
| produkty: `[{'nazwa': 'Książka', 'cena': 30, 'kategoria': 'edukacja', 'dostepny': true}, {'nazwa': 'Kurs', 'cena': 200, 'kategoria': 'edukacja', 'dostepny': true}]`, `kategoria: 'edukacja', maxCena: 100` | `[{nazwa: Książka, cena: 30, kategoria: edukacja, dostepny: true}]` |

### Wskazówki

1. Użyj metody `where()` na liście do filtrowania z predykatem
2. Każdy filtr powinien być stosowany tylko gdy odpowiedni parametr nie jest `null`
3. Składnia strzałkowa nadaje się idealnie do prostych predykatów: `bool spelniaWarunek(Map p) => p['cena'] >= minCena`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
List<Map<String, dynamic>> filtrujProdukty(
  List<Map<String, dynamic>> produkty, {
  double? minCena,
  double? maxCena,
  String? kategoria,
  bool tylkoDostepne = true,
}) {
  return produkty.where((p) {
    if (tylkoDostepne && p['dostepny'] != true) return false;
    if (minCena != null && (p['cena'] as num) < minCena) return false;
    if (maxCena != null && (p['cena'] as num) > maxCena) return false;
    if (kategoria != null && p['kategoria'] != kategoria) return false;
    return true;
  }).toList();
}

void main() {
  var produkty = [
    {'nazwa': 'Laptop', 'cena': 3500, 'kategoria': 'elektronika', 'dostepny': true},
    {'nazwa': 'Mysz', 'cena': 50, 'kategoria': 'elektronika', 'dostepny': false},
    {'nazwa': 'Książka', 'cena': 30, 'kategoria': 'edukacja', 'dostepny': true},
    {'nazwa': 'Kurs', 'cena': 200, 'kategoria': 'edukacja', 'dostepny': true},
  ];

  print(filtrujProdukty(produkty, minCena: 100));
  print(filtrujProdukty(produkty, kategoria: 'edukacja', maxCena: 100));
}
```

</details>

---

**Następny moduł:** [Lambdy i typedef](02-lambdas-typedef.md)
**Poprzedni moduł:** [Assert](../03-control-flow/03-assert.md)
