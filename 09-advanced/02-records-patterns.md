---
id: "9.2"
title: "Rekordy i pattern matching"
difficulty: "advanced"
section: "09-advanced"
prerequisites:
  - "Switch i pattern matching"
  - "Klasy i konstruktory"
  - "Zmienne i typy danych"
---

# 9.2 Rekordy i pattern matching

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Switch i pattern matching](../03-control-flow/02-switch-patterns.md), [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Zmienne i typy danych](../01-basics/01-variables-types.md)
- **Cele nauki:**
  1. Deklarować i używać rekordów z polami pozycyjnymi i nazwanymi oraz rozumieć ich semantykę równości
  2. Stosować destrukturyzację wzorców w deklaracjach zmiennych, przypisaniach oraz konstrukcji `if-case`
  3. Wykorzystywać wyrażenia `switch` ze sprawdzaniem wyczerpywalności, wzorce logiczne (`&&`, `||`) i relacyjne (`<`, `>`, `==`) do przetwarzania danych

---

## Deklaracja rekordów i pola nazwane

Rekord (ang. *record*) to lekki, niezmienny typ agregujący kilka wartości bez konieczności definiowania osobnej klasy. Rekord może mieć pola **pozycyjne** (dostępne przez `.$1`, `.$2`) oraz pola **nazwane** (dostępne przez nazwę). Rekordy są idealne do zwracania wielu wartości z funkcji.

### Rekordy pozycyjne i nazwane

Poniższy przykład demonstruje tworzenie rekordów oraz dostęp do pól pozycyjnych i nazwanych:

```dart
void main() {
  // Rekord pozycyjny — dostęp przez .$1, .$2, ...
  var wspolrzedne = (52.23, 21.01);
  print('Szerokość: ${wspolrzedne.$1}, Długość: ${wspolrzedne.$2}');

  // Rekord z polami nazwanymi — dostęp przez nazwę pola
  var uzytkownik = (imie: 'Anna', wiek: 30);
  print('${uzytkownik.imie} ma ${uzytkownik.wiek} lat');

  // Rekord mieszany: pola pozycyjne + nazwane
  var pomiar = (100, jednostka: 'cm', dokladny: true);
  print('${pomiar.$1} ${pomiar.jednostka} (dokładny: ${pomiar.dokladny})');
}
// Oczekiwane wyjście:
// Szerokość: 52.23, Długość: 21.01
// Anna ma 30 lat
// 100 cm (dokładny: true)
```

### Rekord jako typ zwracany funkcji

Rekordy pozwalają zwrócić wiele wartości bez tworzenia dedykowanej klasy. Adnotacja typu rekordu opisuje kolejność i nazwy pól:

```dart
// Typ zwracany to rekord z dwoma nazwanymi polami
({int min, int max}) zakres(List<int> liczby) {
  var najmniejsza = liczby.reduce((a, b) => a < b ? a : b);
  var najwieksza = liczby.reduce((a, b) => a > b ? a : b);
  return (min: najmniejsza, max: najwieksza); // konstrukcja rekordu
}

void main() {
  var wynik = zakres([4, 8, 15, 16, 23, 42]);
  print('Min: ${wynik.min}, Max: ${wynik.max}');
}
// Oczekiwane wyjście:
// Min: 4, Max: 42
```

---

## Semantyka równości rekordów

Rekordy mają **strukturalną** semantykę równości: dwa rekordy są równe (`==`), jeśli mają ten sam kształt (te same pola pozycyjne i nazwane) oraz odpowiadające sobie pola są równe. Dwie osobne klasy musiałyby ręcznie nadpisywać `==` i `hashCode`, natomiast rekordy robią to automatycznie.

### Porównywanie rekordów przez `==`

Poniższy przykład pokazuje, że rekordy o identycznej strukturze i wartościach są równe niezależnie od tego, gdzie zostały utworzone:

```dart
void main() {
  var a = (x: 1, y: 2);
  var b = (x: 1, y: 2);
  var c = (x: 1, y: 3);

  // Równość strukturalna — porównywane są wartości pól, nie tożsamość obiektu
  print(a == b); // true — te same pola i wartości
  print(a == c); // false — różni się pole y

  // hashCode również jest strukturalny — rekordy działają jako klucze mapy
  var mapa = {(x: 1, y: 2): 'punkt A'};
  print(mapa[(x: 1, y: 2)]); // odczyt przez równoważny rekord
}
// Oczekiwane wyjście:
// true
// false
// punkt A
```

---

## Destrukturyzacja w deklaracjach i przypisaniach

Destrukturyzacja pozwala rozbić rekord (lub inną strukturę) na osobne zmienne w jednym kroku. Można jej użyć w deklaracji zmiennej oraz w przypisaniu do istniejących zmiennych.

### Destrukturyzacja w deklaracji zmiennej

Poniższy przykład rozbiera rekord na osobne zmienne bezpośrednio w deklaracji:

```dart
({String miasto, int kodPocztowy}) adres() => (miasto: 'Kraków', kodPocztowy: 30001);

void main() {
  // Destrukturyzacja rekordu nazwanego w deklaracji — nazwy pól muszą się zgadzać
  var (miasto: m, kodPocztowy: k) = adres();
  print('$m $k');

  // Destrukturyzacja rekordu pozycyjnego
  var (dzien, miesiac, rok) = (14, 3, 2024);
  print('$dzien.$miesiac.$rok');
}
// Oczekiwane wyjście:
// Kraków 30001
// 14.3.2024
```

### Destrukturyzacja w przypisaniu (swap)

Wzorce można stosować także w przypisaniu do istniejących zmiennych — klasyczny przykład to zamiana wartości bez zmiennej tymczasowej:

```dart
void main() {
  var lewo = 'A';
  var prawo = 'B';

  // Przypisanie z destrukturyzacją — zamiana wartości w jednej linii
  (lewo, prawo) = (prawo, lewo);
  print('lewo=$lewo, prawo=$prawo');
}
// Oczekiwane wyjście:
// lewo=B, prawo=A
```

---

## Konstrukcja `if-case`

Instrukcja `if (wartosc case wzorzec)` sprawdza, czy wartość pasuje do wzorca, i jeśli tak — wiąże zmienne z dopasowanego wzorca. Jest to wygodny sposób na warunkowe rozbieranie pojedynczej struktury, bez pełnego `switch`.

### Dopasowanie i wiązanie zmiennych z `if-case`

Poniższy przykład używa `if-case` z guard clause `when` do warunkowego przetworzenia rekordu:

```dart
void main() {
  var odpowiedz = (status: 200, dane: 'Zawartość strony');

  // if-case: dopasowuje wzorzec i wiąże zmienną 'tresc' tylko gdy status == 200
  if (odpowiedz case (status: 200, dane: var tresc)) {
    print('OK: $tresc');
  }

  // if-case z guard 'when' — dodatkowy warunek logiczny
  var pomiar = (temperatura: 38.5);
  if (pomiar case (temperatura: var t) when t > 37.0) {
    print('Podwyższona temperatura: $t°C');
  } else {
    print('Temperatura prawidłowa');
  }
}
// Oczekiwane wyjście:
// OK: Zawartość strony
// Podwyższona temperatura: 38.5°C
```

---

## Wyrażenia `switch` ze sprawdzaniem wyczerpywalności

Wyrażenie `switch` zwraca wartość i — w połączeniu z klasami `sealed` — pozwala kompilatorowi sprawdzić, czy obsłużono wszystkie możliwe przypadki. Jeśli pominiesz podtyp, otrzymasz błąd kompilacji zamiast błędu w czasie działania.

### Wyczerpywalność z `sealed class`

Poniższy przykład definiuje zamkniętą hierarchię zdarzeń — kompilator wymusza obsługę każdego wariantu:

```dart
// sealed ogranicza podtypy do tego pliku i umożliwia sprawdzanie wyczerpywalności
sealed class Zdarzenie {}

class Klikniecie extends Zdarzenie {
  final int x;
  final int y;
  Klikniecie(this.x, this.y);
}

class Przewiniecie extends Zdarzenie {
  final double delta;
  Przewiniecie(this.delta);
}

class Klawisz extends Zdarzenie {
  final String znak;
  Klawisz(this.znak);
}

String opisz(Zdarzenie e) {
  // Brak 'default' — kompilator gwarantuje, że wszystkie podtypy są pokryte
  return switch (e) {
    Klikniecie(x: var x, y: var y) => 'Kliknięcie w ($x, $y)',
    Przewiniecie(delta: var d) => 'Przewinięcie o $d',
    Klawisz(znak: var z) => 'Wciśnięto klawisz: $z',
  };
}

void main() {
  var zdarzenia = <Zdarzenie>[Klikniecie(10, 20), Przewiniecie(1.5), Klawisz('A')];
  for (var e in zdarzenia) {
    print(opisz(e));
  }
}
// Oczekiwane wyjście:
// Kliknięcie w (10, 20)
// Przewinięcie o 1.5
// Wciśnięto klawisz: A
```

---

## Wzorce logiczne (`&&`, `||`)

Wzorce logiczne łączą kilka wzorców. Wzorzec `||` pasuje, gdy dopasuje się którakolwiek z alternatyw; wzorzec `&&` pasuje, gdy dopasują się oba pod-wzorce jednocześnie (przydatne do nakładania dodatkowych warunków na tę samą wartość).

### Alternatywa `||` i koniunkcja `&&`

Poniższy przykład klasyfikuje znak, używając wzorca `||` dla alternatyw i `&&` do połączenia wzorca relacyjnego z wiązaniem zmiennej:

```dart
String klasyfikujZnak(String znak) {
  return switch (znak) {
    // Wzorzec || — dopasowuje którąkolwiek z wymienionych wartości
    'a' || 'e' || 'i' || 'o' || 'u' || 'y' => 'samogłoska',
    ' ' || '\t' || '\n' => 'biały znak',
    _ => 'inny znak',
  };
}

int? sprawdzWiek(int wiek) {
  return switch (wiek) {
    // Wzorzec && — wartość musi być >= 0 ORAZ zostaje związana ze zmienną 'w'
    >= 0 && var w when w <= 120 => w,
    _ => null, // wartość spoza dozwolonego zakresu
  };
}

void main() {
  print(klasyfikujZnak('e'));
  print(klasyfikujZnak('!'));
  print(sprawdzWiek(30));
  print(sprawdzWiek(-5));
}
// Oczekiwane wyjście:
// samogłoska
// inny znak
// 30
// null
```

---

## Wzorce relacyjne (`<`, `>`, `==`)

Wzorce relacyjne porównują dopasowywaną wartość z wartością stałą za pomocą operatorów `<`, `>`, `<=`, `>=`, `==`, `!=`. Umożliwiają zwięzłe wyrażanie zakresów bezpośrednio we wzorcu, zwykle w połączeniu z operatorem `&&`.

### Klasyfikacja zakresów wartości

Poniższy przykład przypisuje ocenę do wyniku procentowego, używając wzorców relacyjnych i logicznych:

```dart
String ocena(int wynik) {
  return switch (wynik) {
    // Wzorzec relacyjny == dla konkretnej wartości
    == 100 => 'Wynik perfekcyjny',
    // Wzorce relacyjne łączone operatorem && wyrażają zakresy
    >= 90 && < 100 => 'Bardzo dobry',
    >= 75 && < 90 => 'Dobry',
    >= 50 && < 75 => 'Dostateczny',
    >= 0 && < 50 => 'Niedostateczny',
    _ => 'Wynik nieprawidłowy', // np. wartości ujemne
  };
}

void main() {
  for (var w in [100, 95, 80, 60, 30, -1]) {
    print('$w → ${ocena(w)}');
  }
}
// Oczekiwane wyjście:
// 100 → Wynik perfekcyjny
// 95 → Bardzo dobry
// 80 → Dobry
// 60 → Dostateczny
// 30 → Niedostateczny
// -1 → Wynik nieprawidłowy
```

---

## Ćwiczenie 1: Parsowanie danych strukturalnych

### Opis problemu

Napisz funkcję `parsujWspolrzedne(String tekst)` która parsuje tekst w formacie `"szerokosc,dlugosc"` (np. `"52.23,21.01"`) i zwraca rekord `({double lat, double lng})`. Jeśli tekst jest nieprawidłowy (zła liczba części lub niepoprawne liczby), zwróć `null`. Użyj destrukturyzacji wzorca listowego oraz `if-case`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `"52.23,21.01"` | `(lat: 52.23, lng: 21.01)` |
| `"52.23"` | `null` |

### Wskazówki

1. Podziel tekst przez `,` metodą `split(',')`, a następnie użyj wzorca listowego `[var a, var b]` do dopasowania dokładnie dwóch części
2. Do konwersji tekstu na liczbę użyj `double.tryParse`, które zwraca `null` przy błędzie
3. Użyj `if-case` z wzorcem `[var a, var b]` aby sprawdzić, czy lista ma dokładnie dwa elementy

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
({double lat, double lng})? parsujWspolrzedne(String tekst) {
  var czesci = tekst.split(',');

  // Wzorzec listowy [var a, var b] pasuje tylko gdy lista ma dokładnie 2 elementy
  if (czesci case [var a, var b]) {
    var lat = double.tryParse(a.trim());
    var lng = double.tryParse(b.trim());
    // && łączy dwa dopasowania: oba muszą być nie-null (typ double)
    if ((lat, lng) case (double la, double lo)) {
      return (lat: la, lng: lo);
    }
  }
  return null;
}

void main() {
  print(parsujWspolrzedne('52.23,21.01'));
  print(parsujWspolrzedne('52.23'));
  print(parsujWspolrzedne('abc,def'));
}
// Oczekiwane wyjście:
// (lat: 52.23, lng: 21.01)
// null
// null
```

</details>

---

## Ćwiczenie 2: Filtrowanie kolekcji według kształtu

### Opis problemu

Masz listę rekordów opisujących transakcje: `(typ: String, kwota: double)`, gdzie `typ` to `'wplata'` lub `'wyplata'`. Napisz funkcję `saldo(List<({String typ, double kwota})> transakcje)` która oblicza saldo: wpłaty dodają kwotę, wypłaty odejmują. Ignoruj transakcje o nieznanym typie oraz o kwocie mniejszej lub równej 0. Użyj wyrażenia `switch` z destrukturyzacją, wzorcami logicznymi i relacyjnymi.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `[(typ:'wplata', kwota:100), (typ:'wyplata', kwota:30)]` | `70.0` |
| `[(typ:'wplata', kwota:50), (typ:'nieznany', kwota:20)]` | `50.0` |

### Wskazówki

1. Iteruj po transakcjach i dla każdej użyj wyrażenia `switch`, które zwraca wartość dodawaną do salda
2. Wzorzec `(typ: 'wplata', kwota: var k) when k > 0` dopasowuje tylko poprawne wpłaty
3. Dla wypłaty zwróć wartość ujemną (`-k`); dla wszystkiego innego zwróć `0`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
double saldo(List<({String typ, double kwota})> transakcje) {
  var suma = 0.0;
  for (var t in transakcje) {
    // switch zwraca zmianę salda; guard 'when k > 0' odrzuca niepoprawne kwoty
    suma += switch (t) {
      (typ: 'wplata', kwota: var k) when k > 0 => k,
      (typ: 'wyplata', kwota: var k) when k > 0 => -k,
      _ => 0.0, // nieznany typ lub kwota <= 0
    };
  }
  return suma;
}

void main() {
  print(saldo([(typ: 'wplata', kwota: 100), (typ: 'wyplata', kwota: 30)]));
  print(saldo([(typ: 'wplata', kwota: 50), (typ: 'nieznany', kwota: 20)]));
  print(saldo([(typ: 'wplata', kwota: 200), (typ: 'wyplata', kwota: -5)]));
}
// Oczekiwane wyjście:
// 70.0
// 50.0
// 200.0
```

</details>

---

## Ćwiczenie 3: Klasyfikacja punktów według położenia

### Opis problemu

Napisz funkcję `opiszPunkt((int, int) punkt)` która przyjmuje rekord pozycyjny `(x, y)` i zwraca opis położenia punktu. Użyj wyrażenia `switch` z destrukturyzacją, wzorcami stałymi i relacyjnymi. Kategorie: `"początek układu"` (0, 0), `"oś X"` (y == 0), `"oś Y"` (x == 0), `"ćwiartka I"` (x > 0 i y > 0), `"inna ćwiartka"` (reszta).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `(0, 0)` | `początek układu` |
| `(5, 3)` | `ćwiartka I` |

### Wskazówki

1. Destrukturyzuj rekord pozycyjny wzorcem `(var x, var y)` lub dopasowuj stałe `(0, 0)`
2. Kolejność wzorców ma znaczenie — najpierw najbardziej specyficzne (początek układu), potem osie, na końcu ćwiartki
3. Użyj guard `when x > 0 && y > 0` do rozpoznania pierwszej ćwiartki

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String opiszPunkt((int, int) punkt) {
  return switch (punkt) {
    (0, 0) => 'początek układu',
    (_, 0) => 'oś X',
    (0, _) => 'oś Y',
    (var x, var y) when x > 0 && y > 0 => 'ćwiartka I',
    _ => 'inna ćwiartka',
  };
}

void main() {
  for (var p in [(0, 0), (5, 0), (0, 7), (5, 3), (-2, 4)]) {
    print('$p → ${opiszPunkt(p)}');
  }
}
// Oczekiwane wyjście:
// (0, 0) → początek układu
// (5, 0) → oś X
// (0, 7) → oś Y
// (5, 3) → ćwiartka I
// (-2, 4) → inna ćwiartka
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. Rekordy to lekkie, niezmienne agregaty wartości z polami pozycyjnymi (`.$1`) i nazwanymi — idealne do zwracania wielu wartości z funkcji bez tworzenia klasy
2. Rekordy mają strukturalną równość: dwa rekordy są równe, gdy mają ten sam kształt i równe pola; automatycznie działają jako klucze map
3. Destrukturyzacja rozbija struktury na zmienne w deklaracjach i przypisaniach — pozwala między innymi zamienić wartości zmiennych w jednej linii
4. Konstrukcja `if-case` warunkowo dopasowuje pojedynczą wartość do wzorca i wiąże zmienne, opcjonalnie z guard clause `when`
5. Wyrażenia `switch` w połączeniu z klasami `sealed` zapewniają sprawdzanie wyczerpywalności w czasie kompilacji — pominięty przypadek to błąd kompilacji, a nie błąd czasu wykonania
6. Wzorce logiczne (`||`, `&&`) i relacyjne (`<`, `>`, `==`) umożliwiają zwięzłe wyrażanie alternatyw, dodatkowych warunków i zakresów wartości

---

**Następny moduł:** [Adnotacje i widoczność](03-annotations-visibility.md)
**Poprzedni moduł:** [Rozszerzenia (extensions)](01-extensions.md)
