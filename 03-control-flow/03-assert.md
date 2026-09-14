---
id: "3.3"
title: "Assert — asercje debugowe"
difficulty: "beginner"
section: "03-control-flow"
prerequisites:
  - "Zmienne i typy danych"
  - "Instrukcje warunkowe i pętle"
---

# 3.3 Assert — asercje debugowe

## Informacje o module

- **Poziom trudności:** beginner
- **Wymagania wstępne:** [Zmienne i typy danych](../01-basics/01-variables-types.md), [Instrukcje warunkowe i pętle](01-conditionals-loops.md)
- **Cele nauki:**
  1. Rozumieć rolę instrukcji `assert` w weryfikacji założeń w kodzie
  2. Wiedzieć kiedy asercje są aktywne (tryb debug) a kiedy wyłączone (tryb produkcyjny)
  3. Stosować `assert` z komunikatami do wczesnego wykrywania błędów logicznych

---

## Czym jest `assert`?

Instrukcja `assert` sprawdza warunek logiczny w czasie wykonania. Jeśli warunek jest `false`, program zostaje przerwany z błędem `AssertionError`. Asercje służą do weryfikacji wewnętrznych założeń programisty — nie do walidacji danych użytkownika.

### Podstawowa składnia `assert`

Poniższy przykład demonstruje użycie `assert` do sprawdzenia warunku wejściowego funkcji:

```dart
double obliczSrednia(List<double> oceny) {
  // assert sprawdza warunek — jeśli false, rzuca AssertionError
  assert(oceny.isNotEmpty, 'Lista ocen nie może być pusta');

  var suma = oceny.fold(0.0, (a, b) => a + b);
  return suma / oceny.length;
}

void main() {
  var mojeOceny = [4.5, 3.0, 5.0, 4.0];
  print('Średnia: ${obliczSrednia(mojeOceny).toStringAsFixed(2)}');

  // Poniższe wywołanie spowoduje AssertionError w trybie debug:
  // obliczSrednia([]);
}
// Oczekiwane wyjście:
// Średnia: 4.13
```

### `assert` z komunikatem

Drugi argument `assert` to opcjonalny komunikat wyświetlany gdy asercja nie przechodzi:

```dart
void ustawWiek(int wiek) {
  // Komunikat pomaga zrozumieć co poszło nie tak
  assert(wiek >= 0, 'Wiek nie może być ujemny, otrzymano: $wiek');
  assert(wiek <= 150, 'Wiek $wiek przekracza realistyczny zakres');
  print('Ustawiono wiek: $wiek');
}

void main() {
  ustawWiek(25);  // OK
  ustawWiek(0);   // OK — noworodek

  // W trybie debug poniższe rzuci:
  // AssertionError: 'Wiek nie może być ujemny, otrzymano: -5'
  // ustawWiek(-5);
}
// Oczekiwane wyjście:
// Ustawiono wiek: 25
// Ustawiono wiek: 0
```

---

## Tryb debug vs tryb produkcyjny

Kluczowa cecha `assert`: asercje są aktywne **tylko wtedy, gdy zostaną jawnie włączone**. W standardowym uruchomieniu z linii poleceń (`dart run` bez dodatkowych flag) asercje są **domyślnie wyłączone** — trzeba dodać flagę `--enable-asserts`. Flutter jest tu wyjątkiem: `flutter run` w trybie debug włącza asercje automatycznie. W skompilowanym kodzie produkcyjnym (`dart compile exe`, `flutter run --release`) asercji nie da się włączyć w ogóle — są całkowicie ignorowane i nie wpływają na wydajność ani zachowanie aplikacji.

### Kiedy asercje są aktywne?

| Środowisko | Assert aktywny? | Uwagi |
|-----------|-----------------|-------|
| `dart run` | ✗ Nie | Domyślnie wyłączone przy zwykłym uruchomieniu |
| `dart run --enable-asserts` | ✓ Tak | Jawne włączenie |
| `flutter run` (debug) | ✓ Tak | Domyślny tryb Flutter debug |
| `dart compile exe` | ✗ Nie | Kompilacja produkcyjna — nie da się włączyć |
| `flutter run --release` | ✗ Nie | Flutter release mode |
| `dart run --no-enable-asserts` | ✗ Nie | Jawne wyłączenie (i tak już domyślne) |

> **Uwaga:** aby samodzielnie zobaczyć `AssertionError` z przykładów w tym module, uruchamiaj je jako `dart run --enable-asserts plik.dart` — samo `dart run plik.dart` nie aktywuje asercji.

### Demonstracja zachowania w różnych trybach

Poniższy przykład pokazuje, że `assert` nie zatrzymuje programu w trybie produkcyjnym:

```dart
void przetworz(int wartosc) {
  // W trybie debug: rzuca AssertionError gdy wartosc < 0
  // W trybie produkcyjnym: ta linia jest całkowicie pomijana
  assert(wartosc >= 0, 'Wartość musi być nieujemna: $wartosc');

  print('Przetwarzam: $wartosc');
}

void main() {
  przetworz(10);  // OK w obu trybach
  przetworz(-1);  // Debug: AssertionError; Produkcja: wypisze "Przetwarzam: -1"
}
// Oczekiwane wyjście (tryb debug — program przerywa się przed -1):
// Przetwarzam: 10
//
// Oczekiwane wyjście (tryb produkcyjny — assert pomijany):
// Przetwarzam: 10
// Przetwarzam: -1
```

### Konsekwencje dla projektowania kodu

Ponieważ asercje są wyłączone w produkcji, **nie używaj ich** do walidacji danych zewnętrznych. Do tego służą zwykłe instrukcje warunkowe i wyjątki:

```dart
// ✗ ŹLE — assert nie chroni w produkcji
void zleUstawHaslo(String haslo) {
  assert(haslo.length >= 8, 'Hasło za krótkie');
  // W produkcji to nic nie sprawdza!
}

// ✓ DOBRZE — walidacja z wyjątkiem działa zawsze
void dobreUstawHaslo(String haslo) {
  if (haslo.length < 8) {
    throw ArgumentError('Hasło musi mieć min. 8 znaków, ma: ${haslo.length}');
  }
  print('Hasło ustawione');
}

void main() {
  // Rzuci wyjątek niezależnie od trybu uruchomienia
  try {
    dobreUstawHaslo('abc');
  } on ArgumentError catch (e) {
    print('Błąd: ${e.message}');
  }

  dobreUstawHaslo('bezpieczne_haslo123');
}
// Oczekiwane wyjście:
// Błąd: Hasło musi mieć min. 8 znaków, ma: 3
// Hasło ustawione
```

---

## Typowe zastosowania `assert`

### Weryfikacja niezmienników klasy

Asercje doskonale nadają się do sprawdzania wewnętrznych niezmienników (invariants) — warunków, które powinny być zawsze prawdziwe:

```dart
class Ulamek {
  final int licznik;
  final int mianownik;

  Ulamek(this.licznik, this.mianownik) {
    // Niezmiennik: mianownik nie może być zero
    assert(mianownik != 0, 'Mianownik nie może być zero');
    // Niezmiennik: ułamek jest w postaci skróconej (opcjonalnie)
    assert(_nwd(licznik.abs(), mianownik.abs()) == 1,
        'Ułamek $licznik/$mianownik nie jest skrócony');
  }

  static int _nwd(int a, int b) => b == 0 ? a : _nwd(b, a % b);

  @override
  String toString() => '$licznik/$mianownik';
}

void main() {
  var polowa = Ulamek(1, 2);
  var trzyPierwsze = Ulamek(3, 1);
  print('$polowa, $trzyPierwsze');

  // W trybie debug poniższe rzuci AssertionError:
  // var bledny = Ulamek(2, 4); // nie jest skrócony
  // var zero = Ulamek(1, 0);   // mianownik zero
}
// Oczekiwane wyjście:
// 1/2, 3/1
```

### Weryfikacja stanu po operacji

Asercje mogą sprawdzać warunki końcowe (postconditions) — czy wynik operacji spełnia oczekiwania:

```dart
List<int> sortujRosnaco(List<int> dane) {
  var posortowane = List<int>.from(dane)..sort();

  // Postcondition: wynik jest posortowany rosnąco
  assert(() {
    for (var i = 0; i < posortowane.length - 1; i++) {
      if (posortowane[i] > posortowane[i + 1]) return false;
    }
    return true;
  }(), 'Lista nie jest posortowana po operacji sort');

  // Postcondition: nie straciliśmy elementów
  assert(posortowane.length == dane.length,
      'Zmieniono liczbę elementów: ${dane.length} → ${posortowane.length}');

  return posortowane;
}

void main() {
  var dane = [5, 2, 8, 1, 9, 3];
  var wynik = sortujRosnaco(dane);
  print('Dane: $dane');
  print('Posortowane: $wynik');
}
// Oczekiwane wyjście:
// Dane: [5, 2, 8, 1, 9, 3]
// Posortowane: [1, 2, 3, 5, 8, 9]
```

### `assert` z wyrażeniem lambda

Gdy warunek jest złożony, można użyć wyrażenia lambda (IIFE) w `assert`. Pozwala to na wieloliniową logikę sprawdzenia:

```dart
void main() {
  var macierz = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9],
  ];

  // assert z IIFE — złożona walidacja w jednym assert
  assert(() {
    // Sprawdzenie: macierz jest kwadratowa
    var wiersze = macierz.length;
    for (var wiersz in macierz) {
      if (wiersz.length != wiersze) return false;
    }
    return true;
  }(), 'Macierz nie jest kwadratowa');

  print('Macierz ${macierz.length}x${macierz[0].length} jest poprawna');
}
// Oczekiwane wyjście:
// Macierz 3x3 jest poprawna
```

---

## `assert` vs inne mechanizmy walidacji

| Mechanizm | Aktywny w produkcji | Zastosowanie |
|-----------|-------------------|--------------|
| `assert` | Nie (tylko debug) | Wewnętrzne niezmienniki, warunki wstępne/końcowe |
| `if` + `throw` | Tak | Walidacja danych zewnętrznych, argumentów API |
| `RangeError.checkValidIndex` | Tak | Sprawdzanie indeksów |
| `ArgumentError.checkNotNull` | Tak | Sprawdzanie null |

Poniższy przykład ilustruje kiedy użyć `assert` a kiedy wyjątku:

```dart
class Konto {
  double _saldo;

  Konto(this._saldo) {
    // assert — wewnętrzny niezmiennik (programista nie powinien tworzyć
    // konta z ujemnym saldem, ale walidacja zewnętrzna jest gdzie indziej)
    assert(_saldo >= 0, 'Początkowe saldo nie może być ujemne');
  }

  void wplac(double kwota) {
    // throw — walidacja danych wejściowych (użytkownik może podać błędną kwotę)
    if (kwota <= 0) {
      throw ArgumentError('Kwota wpłaty musi być dodatnia: $kwota');
    }
    _saldo += kwota;

    // assert — postcondition (saldo po wpłacie nie powinno spaść)
    assert(_saldo >= kwota, 'Błąd logiczny: saldo mniejsze niż wpłata');
  }

  double get saldo => _saldo;
}

void main() {
  var konto = Konto(100.0);
  konto.wplac(50.0);
  print('Saldo: ${konto.saldo} zł');

  try {
    konto.wplac(-10.0);
  } on ArgumentError catch (e) {
    print('Błąd: ${e.message}');
  }
}
// Oczekiwane wyjście:
// Saldo: 150.0 zł
// Błąd: Kwota wpłaty musi być dodatnia: -10.0
```

---

## Ćwiczenie 1: Podstawowe assert

### Opis problemu

Napisz klasę `Zakres` z polami `min` i `max` (oba `int`). W konstruktorze użyj `assert` aby zapewnić, że `min <= max`. Dodaj metodę `zawiera(int wartosc)` która sprawdza czy wartość mieści się w zakresie. Użyj `assert` do weryfikacji postcondition w metodzie `przesun(int delta)` (przesuwa zakres o delta, assert sprawdza że nowy min <= nowy max).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Zakres(1, 10).zawiera(5)` | `true` |
| `Zakres(1, 10).zawiera(11)` | `false` |

### Wskazówki

1. Asercja w konstruktorze: `assert(min <= max, 'min musi być <= max')`
2. Metoda `zawiera`: zwykłe porównanie, nie potrzebuje assert
3. W `przesun`: po operacji dodaj assert sprawdzający niezmiennik

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Zakres {
  int min;
  int max;

  Zakres(this.min, this.max) {
    assert(min <= max, 'min ($min) musi być <= max ($max)');
  }

  bool zawiera(int wartosc) {
    return wartosc >= min && wartosc <= max;
  }

  void przesun(int delta) {
    min += delta;
    max += delta;
    // Postcondition: przesunięcie nie zmienia relacji min <= max
    assert(min <= max, 'Błąd po przesunięciu: min=$min > max=$max');
  }

  @override
  String toString() => '[$min, $max]';
}

void main() {
  var zakres = Zakres(1, 10);
  print('Zakres: $zakres');
  print('Zawiera 5: ${zakres.zawiera(5)}');
  print('Zawiera 11: ${zakres.zawiera(11)}');

  zakres.przesun(5);
  print('Po przesunięciu +5: $zakres');
  print('Zawiera 8: ${zakres.zawiera(8)}');
}
```

</details>

---

## Ćwiczenie 2: Assert vs walidacja

### Opis problemu

Napisz klasę `Macierz2x2` reprezentującą macierz 2×2. Konstruktor przyjmuje 4 wartości `double`. Dodaj metodę `wyznacznik()` oraz metodę `odwrotna()` która zwraca macierz odwrotną. W `odwrotna()`:
- Użyj `assert` do weryfikacji postcondition (iloczyn macierzy i jej odwrotnej ≈ macierz jednostkowa)
- Użyj `throw` gdy wyznacznik == 0 (macierz nieosobliwa — to walidacja, nie niezmiennik)

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Macierz2x2(1, 2, 3, 4).wyznacznik()` | `-2.0` |
| `Macierz2x2(1, 0, 0, 1).odwrotna()` | `[1.0, -0.0; -0.0, 1.0]` (macierz jednostkowa; `-0.0` to artefakt arytmetyki zmiennoprzecinkowej przy dzieleniu `-0/1`) |

### Wskazówki

1. Wyznacznik macierzy [[a,b],[c,d]]: `a*d - b*c`
2. Macierz odwrotna: `(1/det) * [[d,-b],[-c,a]]`
3. Assert postcondition: po obliczeniu odwrotnej, sprawdź że `A * A^-1 ≈ I` (z tolerancją epsilon)

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Macierz2x2 {
  final double a, b, c, d;

  Macierz2x2(this.a, this.b, this.c, this.d);

  double wyznacznik() => a * d - b * c;

  Macierz2x2 odwrotna() {
    var det = wyznacznik();
    // Walidacja — wyznacznik 0 oznacza macierz osobliwą (nie do odwrócenia)
    if (det == 0) {
      throw StateError('Macierz osobliwa — nie można odwrócić (det=0)');
    }

    var inv = Macierz2x2(d / det, -b / det, -c / det, a / det);

    // Postcondition: A * A^-1 ≈ I (macierz jednostkowa)
    assert(() {
      var m = pomnoz(this, inv);
      const eps = 1e-10;
      return (m.a - 1).abs() < eps &&
          m.b.abs() < eps &&
          m.c.abs() < eps &&
          (m.d - 1).abs() < eps;
    }(), 'Postcondition failed: A * A^-1 != I');

    return inv;
  }

  static Macierz2x2 pomnoz(Macierz2x2 x, Macierz2x2 y) {
    return Macierz2x2(
      x.a * y.a + x.b * y.c,
      x.a * y.b + x.b * y.d,
      x.c * y.a + x.d * y.c,
      x.c * y.b + x.d * y.d,
    );
  }

  @override
  String toString() =>
      '[${a.toStringAsFixed(1)}, ${b.toStringAsFixed(1)}; '
      '${c.toStringAsFixed(1)}, ${d.toStringAsFixed(1)}]';
}

void main() {
  var m = Macierz2x2(1, 2, 3, 4);
  print('Macierz: $m');
  print('Wyznacznik: ${m.wyznacznik()}');
  print('Odwrotna: ${m.odwrotna()}');

  var jednostkowa = Macierz2x2(1, 0, 0, 1);
  print('Odwrotna jednostkowej: ${jednostkowa.odwrotna()}');

  try {
    var osobliwa = Macierz2x2(1, 2, 2, 4); // det = 0
    osobliwa.odwrotna();
  } on StateError catch (e) {
    print('Błąd: ${e.message}');
  }
}
```

</details>

---

**Następny moduł:** [Deklaracje funkcji i parametry](../04-functions/01-declarations-params.md)
**Poprzedni moduł:** [Switch i pattern matching](02-switch-patterns.md)
