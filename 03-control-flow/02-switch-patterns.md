---
id: "3.2"
title: "Switch i pattern matching"
difficulty: "beginner"
section: "03-control-flow"
prerequisites:
  - "Zmienne i typy danych"
  - "Operatory"
  - "Instrukcje warunkowe i pętle"
---

# 3.2 Switch i pattern matching

## Informacje o module

- **Poziom trudności:** beginner
- **Wymagania wstępne:** [Zmienne i typy danych](../01-basics/01-variables-types.md), [Operatory](../01-basics/02-operators.md), [Instrukcje warunkowe i pętle](01-conditionals-loops.md)
- **Cele nauki:**
  1. Stosować tradycyjną instrukcję `switch/case` do obsługi wielu wariantów wartości
  2. Wykorzystywać wyrażenia `switch` z Dart 3 do zwięzłego dopasowania wzorców
  3. Stosować destrukturyzację, guard clauses i sprawdzanie wyczerpywalności z sealed classes

---

## Tradycyjny `switch/case`

Tradycyjna instrukcja `switch` porównuje wartość z serią stałych `case`. Jest alternatywą dla długich łańcuchów `if/else if` gdy porównujemy jedną zmienną z wieloma wartościami.

### Podstawowe użycie `switch`

Poniższy przykład demonstruje klasyczny `switch` z obsługą dnia tygodnia:

```dart
void main() {
  var dzien = 'wtorek';

  // Tradycyjny switch porównuje wartość ze stałymi case
  switch (dzien) {
    case 'poniedzialek':
      print('Początek tygodnia');
      break;
    case 'wtorek':
    case 'sroda':
    case 'czwartek':
      print('Środek tygodnia');
      break;
    case 'piatek':
      print('Prawie weekend!');
      break;
    case 'sobota':
    case 'niedziela':
      print('Weekend!');
      break;
    default:
      print('Nieznany dzień');
  }
}
// Oczekiwane wyjście:
// Środek tygodnia
```

### `switch` z typem enum

Instrukcja `switch` jest szczególnie przydatna z typami wyliczeniowymi — Dart ostrzega gdy nie obsłużymy wszystkich wartości enum:

```dart
enum StatusZamowienia { nowe, wRealizacji, wyslane, dostarczone, anulowane }

void main() {
  var status = StatusZamowienia.wyslane;

  // switch z enum — Dart sprawdza kompletność obsługi
  switch (status) {
    case StatusZamowienia.nowe:
      print('Zamówienie przyjęte');
      break;
    case StatusZamowienia.wRealizacji:
      print('Pakujemy zamówienie');
      break;
    case StatusZamowienia.wyslane:
      print('Przesyłka w drodze');
      break;
    case StatusZamowienia.dostarczone:
      print('Zamówienie dostarczone');
      break;
    case StatusZamowienia.anulowane:
      print('Zamówienie anulowane');
      break;
  }
}
// Oczekiwane wyjście:
// Przesyłka w drodze
```

### `switch` z wieloma wartościami w jednym `case`

Gdy kilka wartości prowadzi do tego samego wyniku, można je grupować:

```dart
void main() {
  var kodHTTP = 404;

  // Grupowanie wielu wartości w jednym case
  switch (kodHTTP) {
    case 200:
    case 201:
    case 204:
      print('Sukces');
      break;
    case 301:
    case 302:
      print('Przekierowanie');
      break;
    case 400:
    case 401:
    case 403:
    case 404:
      print('Błąd klienta');
      break;
    case 500:
    case 502:
    case 503:
      print('Błąd serwera');
      break;
    default:
      print('Nieznany kod: $kodHTTP');
  }
}
// Oczekiwane wyjście:
// Błąd klienta
```

---

## Wyrażenia `switch` w Dart 3

Dart 3 wprowadza wyrażenia `switch` — zwracają wartość i wspierają zaawansowane wzorce dopasowania. Są zwięzłe i eliminują potrzebę `break`.

### Podstawowe wyrażenie `switch`

Wyrażenie `switch` zwraca wartość — idealne do przypisywania wyników:

```dart
void main() {
  var ocena = 4;

  // Wyrażenie switch — zwraca wartość, nie wymaga break
  var opis = switch (ocena) {
    5 => 'celujący',
    4 => 'bardzo dobry',
    3 => 'dostateczny',
    2 => 'dopuszczający',
    1 => 'niedostateczny',
    _ => 'nieznana ocena', // _ to wzorzec wildcard (domyślny)
  };

  print('Ocena $ocena: $opis');
}
// Oczekiwane wyjście:
// Ocena 4: bardzo dobry
```

### Wyrażenie `switch` z typami

Wyrażenia `switch` mogą dopasowywać typy i jednocześnie wiązać zmienne:

```dart
void main() {
  Object wartosc = 3.14;

  // Dopasowanie typu — Dart automatycznie rzutuje zmienną
  var opis = switch (wartosc) {
    int n => 'Liczba całkowita: $n',
    double d => 'Liczba zmiennoprzecinkowa: ${d.toStringAsFixed(2)}',
    String s => 'Tekst o długości ${s.length}',
    bool b => 'Wartość logiczna: $b',
    _ => 'Inny typ: ${wartosc.runtimeType}',
  };

  print(opis);
}
// Oczekiwane wyjście:
// Liczba zmiennoprzecinkowa: 3.14
```

---

## Destrukturyzacja wzorców (Destructuring Patterns)

Dart 3 pozwala na rozbieranie struktur danych (rekordów, list, map) bezpośrednio we wzorcach `switch`.

### Destrukturyzacja rekordów

Poniższy przykład demonstruje destrukturyzację rekordu (krotek) w wyrażeniu `switch`:

```dart
void main() {
  // Rekord (krotka) opisujący punkt na płaszczyźnie
  var punkt = (x: 3, y: 0);

  // Destrukturyzacja rekordu z nazwanymi polami
  var pozycja = switch (punkt) {
    (x: 0, y: 0) => 'Początek układu',
    (x: var px, y: 0) => 'Na osi X: x=$px',
    (x: 0, y: var py) => 'Na osi Y: y=$py',
    (x: var px, y: var py) => 'Punkt ($px, $py)',
  };

  print(pozycja);
}
// Oczekiwane wyjście:
// Na osi X: x=3
```

### Destrukturyzacja list

Wzorce listowe pozwalają dopasowywać i rozbierać listy na elementy:

```dart
void main() {
  var komendy = ['move', '10', '20'];

  // Destrukturyzacja listy — dopasowanie do wzorca [element, ...]
  var wynik = switch (komendy) {
    ['quit'] => 'Wyjście z programu',
    ['move', var x, var y] => 'Ruch do ($x, $y)',
    ['print', ...var reszta] => 'Drukowanie: ${reszta.join(" ")}',
    [] => 'Pusta komenda',
    _ => 'Nieznana komenda: ${komendy.first}',
  };

  print(wynik);
}
// Oczekiwane wyjście:
// Ruch do (10, 20)
```

### Destrukturyzacja obiektów

Wzorce obiektowe pozwalają dopasowywać pola klasy:

```dart
class Prostokat {
  final double szerokosc;
  final double wysokosc;

  Prostokat(this.szerokosc, this.wysokosc);
}

void main() {
  var figura = Prostokat(10, 5);

  // Destrukturyzacja obiektu — dopasowanie pól klasy
  var opis = switch (figura) {
    Prostokat(szerokosc: var s, wysokosc: var w) when s == w =>
      'Kwadrat o boku $s',
    Prostokat(szerokosc: var s, wysokosc: var w) =>
      'Prostokąt ${s}x$w (pole: ${s * w})',
  };

  print(opis);
}
// Oczekiwane wyjście:
// Prostokąt 10.0x5.0 (pole: 50.0)
```

---

## Guard clauses (`when`)

Słowo kluczowe `when` dodaje dodatkowy warunek logiczny do wzorca. Wzorzec pasuje tylko wtedy, gdy zarówno struktura, jak i warunek `when` są spełnione.

### Podstawowe użycie `when`

Poniższy przykład wykorzystuje `when` do klasyfikacji temperatur z dodatkowymi warunkami:

```dart
void main() {
  var temp = (wartosc: 38.5, jednostka: 'C');

  // when dodaje warunek logiczny do wzorca
  var diagnoza = switch (temp) {
    (wartosc: var t, jednostka: 'C') when t > 40.0 =>
      'Gorączka krytyczna: ${t}°C',
    (wartosc: var t, jednostka: 'C') when t > 37.0 =>
      'Podwyższona temperatura: ${t}°C',
    (wartosc: var t, jednostka: 'C') when t >= 36.0 =>
      'Temperatura prawidłowa: ${t}°C',
    (wartosc: var t, jednostka: 'C') =>
      'Hipotermia: ${t}°C',
    (wartosc: var t, jednostka: var j) =>
      'Nieobsługiwana jednostka: $j',
  };

  print(diagnoza);
}
// Oczekiwane wyjście:
// Podwyższona temperatura: 38.5°C
```

### `when` z wzorcami logicznymi

Guard clauses można łączyć ze wzorcami logicznymi (`&&`, `||`) dla złożonych warunków:

```dart
void main() {
  var zamowienia = [
    (kwota: 150.0, premium: true),
    (kwota: 50.0, premium: false),
    (kwota: 250.0, premium: false),
    (kwota: 30.0, premium: true),
  ];

  // when z wieloma warunkami — logika rabatowa
  for (var z in zamowienia) {
    var rabat = switch (z) {
      (kwota: var k, premium: true) when k > 100 => '20% rabatu',
      (kwota: var k, premium: true) => '10% rabatu (premium)',
      (kwota: var k, premium: false) when k > 200 => '15% rabatu',
      (kwota: var k, premium: false) when k > 100 => '5% rabatu',
      _ => 'brak rabatu',
    };
    print('Kwota: ${z.kwota} zł, premium: ${z.premium} → $rabat');
  }
}
// Oczekiwane wyjście:
// Kwota: 150.0 zł, premium: true → 20% rabatu
// Kwota: 50.0 zł, premium: false → brak rabatu
// Kwota: 250.0 zł, premium: false → 15% rabatu
// Kwota: 30.0 zł, premium: true → 10% rabatu (premium)
```

---

## Wyczerpywalność z sealed classes

Klasy `sealed` w Dart 3 gwarantują, że kompilator sprawdzi czy `switch` obsługuje wszystkie możliwe podtypy. Jeśli pominiesz przypadek, dostaniesz błąd kompilacji.

### Definicja hierarchii sealed

Poniższy przykład definiuje zamkniętą hierarchię kształtów:

```dart
// sealed class ogranicza podtypy do tego samego pliku
sealed class Ksztalt {}

class Kolo extends Ksztalt {
  final double promien;
  Kolo(this.promien);
}

class Prostokat2 extends Ksztalt {
  final double a;
  final double b;
  Prostokat2(this.a, this.b);
}

class Trojkat extends Ksztalt {
  final double podstawa;
  final double wysokosc;
  Trojkat(this.podstawa, this.wysokosc);
}

// Funkcja obliczająca pole — kompilator sprawdza wyczerpywalność
double obliczPole(Ksztalt ksztalt) {
  // Nie potrzeba 'default' — sealed gwarantuje kompletność
  return switch (ksztalt) {
    Kolo(promien: var r) => 3.14159 * r * r,
    Prostokat2(a: var a, b: var b) => a * b,
    Trojkat(podstawa: var p, wysokosc: var h) => 0.5 * p * h,
  };
}

void main() {
  var figury = <Ksztalt>[
    Kolo(5),
    Prostokat2(4, 6),
    Trojkat(3, 8),
  ];

  for (var f in figury) {
    print('${f.runtimeType}: pole = ${obliczPole(f).toStringAsFixed(2)}');
  }
}
// Oczekiwane wyjście:
// Kolo: pole = 78.54
// Prostokat2: pole = 24.00
// Trojkat: pole = 12.00
```

### Sealed class z polimorficzną logiką

Ten przykład pokazuje jak sealed classes wymuszają obsługę wszystkich wariantów w logice biznesowej:

```dart
// Zamknięta hierarchia wyników operacji sieciowej
sealed class Wynik<T> {}

class Sukces<T> extends Wynik<T> {
  final T dane;
  Sukces(this.dane);
}

class Blad<T> extends Wynik<T> {
  final String komunikat;
  final int kodBledu;
  Blad(this.komunikat, this.kodBledu);
}

class Ladowanie<T> extends Wynik<T> {
  final double postep; // 0.0 – 1.0
  Ladowanie(this.postep);
}

// Kompilator wymusza obsługę Sukces, Blad i Ladowanie
String opisWyniku(Wynik<String> wynik) {
  return switch (wynik) {
    Sukces(dane: var d) => '✓ Dane: $d',
    Blad(komunikat: var msg, kodBledu: var kod) => '✗ Błąd $kod: $msg',
    Ladowanie(postep: var p) => '⏳ Ładowanie: ${(p * 100).toInt()}%',
  };
}

void main() {
  var wyniki = <Wynik<String>>[
    Sukces('Dane załadowane'),
    Blad('Timeout', 408),
    Ladowanie(0.75),
  ];

  for (var w in wyniki) {
    print(opisWyniku(w));
  }
}
// Oczekiwane wyjście:
// ✓ Dane: Dane załadowane
// ✗ Błąd 408: Timeout
// ⏳ Ładowanie: 75%
```

---

## Wzorce logiczne i relacyjne

Dart 3 wspiera wzorce logiczne (`&&`, `||`) oraz relacyjne (`<`, `>`, `<=`, `>=`, `==`) w wyrażeniach `switch`.

### Wzorce relacyjne

Wzorce relacyjne pozwalają porównywać wartość z progami bezpośrednio we wzorcu:

```dart
void main() {
  var temperatura = 22;

  // Wzorce relacyjne — porównanie z wartościami progowymi
  var strefa = switch (temperatura) {
    < 0 => 'mróz',
    >= 0 && < 10 => 'zimno',
    >= 10 && < 20 => 'chłodno',
    >= 20 && < 30 => 'ciepło',
    >= 30 => 'gorąco',
    _ => 'nieznana', // nieosiągalne, ale wymagane dla int
  };

  print('$temperatura°C → $strefa');
}
// Oczekiwane wyjście:
// 22°C → ciepło
```

### Wzorce logiczne (`||`)

Wzorzec `||` dopasowuje gdy którykolwiek z alternatywnych wzorców pasuje:

```dart
void main() {
  var znaki = ['+', '-', '*', '/', '%', '^'];

  for (var z in znaki) {
    var kategoria = switch (z) {
      '+' || '-' => 'addytywny',
      '*' || '/' || '%' => 'multiplikatywny',
      _ => 'inny',
    };
    print("'$z' → $kategoria");
  }
}
// Oczekiwane wyjście:
// '+' → addytywny
// '-' → addytywny
// '*' → multiplikatywny
// '/' → multiplikatywny
// '%' → multiplikatywny
// '^' → inny
```

---

## Ćwiczenie 1: Tradycyjny `switch`

### Opis problemu

Napisz funkcję `dniWMiesiacu(int miesiac, {int rok = 2024})` która zwraca liczbę dni w podanym miesiącu. Użyj tradycyjnej instrukcji `switch`. Uwzględnij lata przestępne dla lutego (rok podzielny przez 4, ale nie przez 100, chyba że przez 400).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `miesiac = 2, rok = 2024` | `Luty 2024: 29 dni` |
| `miesiac = 11, rok = 2023` | `Listopad 2023: 30 dni` |

### Wskazówki

1. Miesiące 30-dniowe: 4, 6, 9, 11; 31-dniowe: 1, 3, 5, 7, 8, 10, 12
2. Rok przestępny: `(rok % 4 == 0 && rok % 100 != 0) || rok % 400 == 0`
3. Pogrupuj case'y z tą samą liczbą dni

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
int dniWMiesiacu(int miesiac, {int rok = 2024}) {
  switch (miesiac) {
    case 1:
    case 3:
    case 5:
    case 7:
    case 8:
    case 10:
    case 12:
      return 31;
    case 4:
    case 6:
    case 9:
    case 11:
      return 30;
    case 2:
      var przestepny =
          (rok % 4 == 0 && rok % 100 != 0) || rok % 400 == 0;
      return przestepny ? 29 : 28;
    default:
      return -1;
  }
}

void main() {
  print('Luty 2024: ${dniWMiesiacu(2, rok: 2024)} dni');
  print('Listopad 2023: ${dniWMiesiacu(11, rok: 2023)} dni');
}
```

</details>

---

## Ćwiczenie 2: Wyrażenie `switch` z Dart 3

### Opis problemu

Napisz funkcję `opisFigury(Object figura)` która przyjmuje rekord opisujący figurę geometryczną i zwraca jej opis z obliczonym polem. Użyj wyrażenia `switch` z destrukturyzacją. Obsłuż: koło `(typ: 'kolo', r: double)`, prostokąt `(typ: 'prostokat', a: double, b: double)`, trójkąt `(typ: 'trojkat', podstawa: double, h: double)`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `(typ: 'kolo', r: 5.0)` | `Koło o promieniu 5.0, pole: 78.54` |
| `(typ: 'prostokat', a: 3.0, b: 4.0)` | `Prostokąt 3.0x4.0, pole: 12.00` |

### Wskazówki

1. Użyj wyrażenia `switch` z wzorcami rekordowymi
2. Do obliczenia pola koła użyj `3.14159 * r * r`
3. Wzorzec `_` obsłuży nieznane typy

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String opisFigury(Object figura) {
  // Wyrażenie switch z destrukturyzacją rekordów i dopasowaniem typu pól
  return switch (figura) {
    (typ: 'kolo', r: double r) =>
      'Koło o promieniu $r, pole: ${(3.14159 * r * r).toStringAsFixed(2)}',
    (typ: 'prostokat', a: double a, b: double b) =>
      'Prostokąt ${a}x$b, pole: ${(a * b).toStringAsFixed(2)}',
    (typ: 'trojkat', podstawa: double p, h: double h) =>
      'Trójkąt (podstawa: $p, h: $h), pole: ${(0.5 * p * h).toStringAsFixed(2)}',
    _ => 'Nieznana figura',
  };
}

void main() {
  var figury = <Object>[
    (typ: 'kolo', r: 5.0),
    (typ: 'prostokat', a: 3.0, b: 4.0),
    (typ: 'trojkat', podstawa: 6.0, h: 4.0),
  ];

  for (var f in figury) {
    print(opisFigury(f));
  }
}
```

</details>

---

## Ćwiczenie 3: Guard clauses

### Opis problemu

Napisz funkcję `klasyfikujProdukt((String nazwa, double cena, int sztuk) produkt)` która klasyfikuje produkt na podstawie ceny i dostępności. Użyj wyrażenia `switch` z `when`. Kategorie: "premium drogi" (cena > 100 i sztuk < 5), "popularny" (sztuk > 50), "standardowy" (reszta).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `('Laptop', 3500.0, 3)` | `Laptop: premium drogi` |
| `('Długopis', 2.50, 200)` | `Długopis: popularny` |

### Wskazówki

1. Użyj destrukturyzacji rekordu pozycyjnego: `(var nazwa, var cena, var sztuk)`
2. Kolejność wzorców ma znaczenie — bardziej specyficzne warunki `when` powinny być wyżej
3. Ostatni wzorzec bez `when` służy jako domyślny

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String klasyfikujProdukt((String, double, int) produkt) {
  return switch (produkt) {
    (var nazwa, var cena, var sztuk) when cena > 100 && sztuk < 5 =>
      '$nazwa: premium drogi',
    (var nazwa, _, var sztuk) when sztuk > 50 =>
      '$nazwa: popularny',
    (var nazwa, _, _) =>
      '$nazwa: standardowy',
  };
}

void main() {
  var produkty = [
    ('Laptop', 3500.0, 3),
    ('Długopis', 2.50, 200),
    ('Książka', 45.0, 30),
  ];

  for (var p in produkty) {
    print(klasyfikujProdukt(p));
  }
}
```

</details>

---

## Ćwiczenie 4: Sealed classes i wyczerpywalność

### Opis problemu

Zdefiniuj hierarchię `sealed class Polecenie` z podklasami: `Dodaj(String element)`, `Usun(int indeks)`, `Wyczysc()`, `Zamien(int indeks, String nowyElement)`. Napisz funkcję `wykonaj` która za pomocą wyrażenia `switch` obsługuje każde polecenie i zwraca opis wykonanej akcji. Kompilator musi sprawdzać wyczerpywalność.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Dodaj('mleko')` | `Dodano: mleko` |
| `Usun(2)` | `Usunięto element na pozycji 2` |

### Wskazówki

1. `sealed class` musi być w tym samym pliku co jej podklasy
2. Nie dodawaj `default` / `_` — chcesz aby kompilator sprawdzał wyczerpywalność
3. Destrukturyzuj pola podklas bezpośrednio we wzorcach

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
sealed class Polecenie {}

class Dodaj extends Polecenie {
  final String element;
  Dodaj(this.element);
}

class Usun extends Polecenie {
  final int indeks;
  Usun(this.indeks);
}

class Wyczysc extends Polecenie {}

class Zamien extends Polecenie {
  final int indeks;
  final String nowyElement;
  Zamien(this.indeks, this.nowyElement);
}

String wykonaj(Polecenie polecenie) {
  return switch (polecenie) {
    Dodaj(element: var e) => 'Dodano: $e',
    Usun(indeks: var i) => 'Usunięto element na pozycji $i',
    Wyczysc() => 'Wyczyszczono listę',
    Zamien(indeks: var i, nowyElement: var n) =>
      'Zamieniono element na pozycji $i na: $n',
  };
}

void main() {
  var polecenia = <Polecenie>[
    Dodaj('mleko'),
    Dodaj('chleb'),
    Usun(0),
    Zamien(0, 'masło'),
    Wyczysc(),
  ];

  for (var p in polecenia) {
    print(wykonaj(p));
  }
}
```

</details>

---

## Ćwiczenie 5: Destrukturyzacja i wzorce logiczne

### Opis problemu

Napisz funkcję `analizujOdpowiedz(({int kod, String tresc, bool cache}) odpowiedz)` która analizuje odpowiedź HTTP. Użyj wyrażenia `switch` z destrukturyzacją i wzorcami relacyjnymi/logicznymi. Kategorie: sukces z cache (kod 200-299 i cache=true), sukces bez cache, przekierowanie (300-399), błąd klienta (400-499), błąd serwera (500+).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `(kod: 200, tresc: 'OK', cache: true)` | `Sukces (z cache): OK` |
| `(kod: 404, tresc: 'Not Found', cache: false)` | `Błąd klienta 404: Not Found` |

### Wskazówki

1. Destrukturyzuj nazwane pola rekordu: `(kod: var k, tresc: var t, cache: var c)`
2. Użyj wzorców relacyjnych z `when` dla zakresów kodów
3. Sprawdź pole `cache` w guard clause

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String analizujOdpowiedz(({int kod, String tresc, bool cache}) odpowiedz) {
  return switch (odpowiedz) {
    (kod: var k, tresc: var t, cache: true) when k >= 200 && k < 300 =>
      'Sukces (z cache): $t',
    (kod: var k, tresc: var t, cache: false) when k >= 200 && k < 300 =>
      'Sukces (bez cache): $t',
    (kod: var k, tresc: var t, cache: _) when k >= 300 && k < 400 =>
      'Przekierowanie $k: $t',
    (kod: var k, tresc: var t, cache: _) when k >= 400 && k < 500 =>
      'Błąd klienta $k: $t',
    (kod: var k, tresc: var t, cache: _) =>
      'Błąd serwera $k: $t',
  };
}

void main() {
  var odpowiedzi = [
    (kod: 200, tresc: 'OK', cache: true),
    (kod: 201, tresc: 'Created', cache: false),
    (kod: 301, tresc: 'Moved', cache: false),
    (kod: 404, tresc: 'Not Found', cache: false),
    (kod: 500, tresc: 'Internal Error', cache: false),
  ];

  for (var o in odpowiedzi) {
    print(analizujOdpowiedz(o));
  }
}
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. Tradycyjny `switch/case` wymaga `break` i operuje na stałych wartościach — używaj go do prostego dopasowania enum i stałych
2. Wyrażenia `switch` w Dart 3 zwracają wartość, nie wymagają `break` i wspierają zaawansowane wzorce
3. Destrukturyzacja pozwala rozbierać rekordy, listy i obiekty bezpośrednio we wzorcach `switch`
4. Guard clauses (`when`) dodają warunki logiczne do wzorców — porządek wzorców ma znaczenie
5. Sealed classes gwarantują wyczerpywalność — kompilator wymusza obsługę wszystkich podtypów
6. Wzorce logiczne (`||`, `&&`) i relacyjne (`<`, `>`, `>=`) umożliwiają zwięzłe wyrażanie zakresów i alternatyw

---

**Następny moduł:** [Assert](03-assert.md)
**Poprzedni moduł:** [Instrukcje warunkowe i pętle](01-conditionals-loops.md)
