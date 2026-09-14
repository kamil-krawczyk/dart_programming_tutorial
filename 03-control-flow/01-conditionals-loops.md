---
id: "3.1"
title: "Instrukcje warunkowe i pętle"
difficulty: "beginner"
section: "03-control-flow"
prerequisites:
  - "Zmienne i typy danych"
  - "Operatory"
---

# 3.1 Instrukcje warunkowe i pętle

## Informacje o module

- **Poziom trudności:** beginner
- **Wymagania wstępne:** [Zmienne i typy danych](../01-basics/01-variables-types.md), [Operatory](../01-basics/02-operators.md)
- **Cele nauki:**
  1. Stosować instrukcje warunkowe `if`, `else if` i `else` do sterowania przebiegiem programu
  2. Wykorzystywać pętle `for`, `for-in`, `while` i `do-while` do iteracji
  3. Kontrolować przebieg pętli za pomocą `break` i `continue`

---

## Instrukcja `if` / `else`

Instrukcja `if` wykonuje blok kodu gdy warunek jest prawdziwy. Opcjonalnie można dodać `else if` dla kolejnych warunków oraz `else` jako domyślną ścieżkę.

### Podstawowe użycie `if` / `else`

Poniższy przykład demonstruje prostą logikę warunkową z oceną temperatury:

```dart
void main() {
  var temperatura = 25;

  // Podstawowa instrukcja if/else
  if (temperatura > 30) {
    print('Gorąco!');
  } else if (temperatura > 20) {
    print('Przyjemnie');
  } else if (temperatura > 10) {
    print('Chłodno');
  } else {
    print('Zimno!');
  }
}
// Oczekiwane wyjście:
// Przyjemnie
```

### `if` z wyrażeniami logicznymi

Dart pozwala łączyć warunki za pomocą operatorów logicznych `&&` (AND) i `||` (OR):

```dart
void main() {
  // Łączenie warunków operatorami logicznymi
  sprawdzWstep(20, true);

  // Warunek z operatorem || (OR)
  var dzien = 'sobota';
  if (dzien == 'sobota' || dzien == 'niedziela') {
    print('Weekend!');
  } else {
    print('Dzień roboczy');
  }
}

void sprawdzWstep(int wiek, bool maLegitymacje) {
  if (wiek >= 18 && maLegitymacje) {
    print('Wstęp dozwolony');
  } else if (wiek >= 18 && !maLegitymacje) {
    print('Potrzebna legitymacja');
  } else {
    print('Wstęp zabroniony — wymagany wiek 18+');
  }
}
// Oczekiwane wyjście:
// Wstęp dozwolony
// Weekend!
```

### `if-case` (Dart 3)

Dart 3 wprowadza składnię `if-case`, która pozwala na dopasowanie wzorców bezpośrednio w instrukcji `if`:

```dart
void main() {
  var punkt = (3, 7);

  // if-case z destrukturyzacją rekordu
  if (punkt case (int x, int y) when x > 0 && y > 0) {
    print('Punkt ($x, $y) jest w pierwszej ćwiartce');
  } else {
    print('Punkt nie jest w pierwszej ćwiartce');
  }
}
// Oczekiwane wyjście:
// Punkt (3, 7) jest w pierwszej ćwiartce
```

---

## Pętla `for`

Klasyczna pętla `for` składa się z inicjalizatora, warunku i inkrementacji. Jest idealna gdy znamy liczbę iteracji z góry.

### Klasyczna pętla `for`

Poniższy przykład generuje tabliczkę mnożenia z użyciem pętli `for`:

```dart
void main() {
  // Klasyczna pętla for — znamy liczbę iteracji
  for (var i = 1; i <= 5; i++) {
    print('$i x 3 = ${i * 3}');
  }
}
// Oczekiwane wyjście:
// 1 x 3 = 3
// 2 x 3 = 6
// 3 x 3 = 9
// 4 x 3 = 12
// 5 x 3 = 15
```

### Zagnieżdżone pętle `for`

Pętle `for` można zagnieżdżać — przydatne przy pracy z macierzami lub generowaniu kombinacji:

```dart
void main() {
  // Zagnieżdżone pętle — generowanie wzoru
  for (var wiersz = 1; wiersz <= 4; wiersz++) {
    var linia = '';
    for (var kol = 1; kol <= wiersz; kol++) {
      linia += '* ';
    }
    print(linia.trim());
  }
}
// Oczekiwane wyjście:
// *
// * *
// * * *
// * * * *
```

---

## Pętla `for-in`

Pętla `for-in` iteruje po elementach dowolnego obiektu `Iterable` (listy, zbiory, mapy itp.) bez potrzeby zarządzania indeksem.

### Iteracja po liście

Pętla `for-in` jest czytelniejsza niż klasyczny `for` gdy nie potrzebujemy indeksu:

```dart
void main() {
  var jezyki = ['Dart', 'Python', 'Rust', 'Go'];

  // for-in iteruje bezpośrednio po elementach
  for (var jezyk in jezyki) {
    print('Lubię $jezyk');
  }
}
// Oczekiwane wyjście:
// Lubię Dart
// Lubię Python
// Lubię Rust
// Lubię Go
```

### Iteracja po mapie

Do iteracji po mapie używamy `.entries`, `.keys` lub `.values`:

```dart
void main() {
  var ceny = {'chleb': 4.50, 'mleko': 3.20, 'ser': 12.00};

  // Iteracja po wpisach mapy — dostęp do klucza i wartości
  for (var entry in ceny.entries) {
    print('${entry.key}: ${entry.value.toStringAsFixed(2)} zł');
  }

  // Suma wartości za pomocą for-in
  var suma = 0.0;
  for (var cena in ceny.values) {
    suma += cena;
  }
  print('Razem: ${suma.toStringAsFixed(2)} zł');
}
// Oczekiwane wyjście:
// chleb: 4.50 zł
// mleko: 3.20 zł
// ser: 12.00 zł
// Razem: 19.70 zł
```

---

## Pętla `while`

Pętla `while` sprawdza warunek przed każdą iteracją. Wykonuje się dopóki warunek jest prawdziwy.

### Podstawowe użycie `while`

Poniższy przykład symuluje odliczanie za pomocą pętli `while`:

```dart
void main() {
  var odliczanie = 5;

  // while sprawdza warunek PRZED wykonaniem ciała pętli
  while (odliczanie > 0) {
    print('$odliczanie...');
    odliczanie--;
  }
  print('Start!');
}
// Oczekiwane wyjście:
// 5...
// 4...
// 3...
// 2...
// 1...
// Start!
```

### `while` z warunkiem złożonym

Pętla `while` jest szczególnie przydatna, gdy liczba iteracji nie jest znana z góry:

```dart
void main() {
  // Symulacja dzielenia przez 2 aż do osiągnięcia wartości < 1
  var wartosc = 100.0;
  var krokow = 0;

  while (wartosc >= 1.0) {
    wartosc /= 2;
    krokow++;
  }

  print('Po $krokow krokach wartość wynosi: ${wartosc.toStringAsFixed(4)}');
}
// Oczekiwane wyjście:
// Po 7 krokach wartość wynosi: 0.7813
```

---

## Pętla `do-while`

Pętla `do-while` sprawdza warunek po wykonaniu ciała — gwarantuje co najmniej jedno wykonanie bloku kodu.

### Podstawowe użycie `do-while`

Poniższy przykład demonstruje walidację danych, która musi wykonać się co najmniej raz:

```dart
void main() {
  var proba = 0;
  var wynik = 0;

  // do-while gwarantuje co najmniej jedno wykonanie
  do {
    proba++;
    wynik = proba * proba; // symulacja "próby"
  } while (wynik < 50);

  print('Sukces po $proba próbach (wynik: $wynik)');
}
// Oczekiwane wyjście:
// Sukces po 8 próbach (wynik: 64)
```

### Porównanie `while` vs `do-while`

Ten przykład ilustruje kluczową różnicę — `do-while` wykonuje się zawsze co najmniej raz:

```dart
void main() {
  var x = 10;

  // while: warunek fałszywy od początku — ciało NIE wykona się
  print('while:');
  while (x < 5) {
    print('  To się nie wypisze');
    x++;
  }
  print('  (pętla pominięta)');

  // do-while: ciało wykona się RAZ nawet gdy warunek jest fałszywy
  print('do-while:');
  do {
    print('  Wykonano raz (x=$x)');
    x++;
  } while (x < 5);
}
// Oczekiwane wyjście:
// while:
//   (pętla pominięta)
// do-while:
//   Wykonano raz (x=10)
```

---

## Instrukcja `break`

`break` natychmiast przerywa wykonywanie najbliższej otaczającej pętli.

### Przerwanie pętli przy znalezieniu elementu

Poniższy przykład używa `break` aby przerwać wyszukiwanie po znalezieniu elementu:

```dart
void main() {
  var dane = [4, 8, 15, 16, 23, 42];
  var szukana = 16;
  var znalezionoNaIndeksie = -1;

  // break przerywa pętlę natychmiast po spełnieniu warunku
  for (var i = 0; i < dane.length; i++) {
    if (dane[i] == szukana) {
      znalezionoNaIndeksie = i;
      break; // nie ma sensu kontynuować szukania
    }
  }

  print('Znaleziono $szukana na indeksie: $znalezionoNaIndeksie');
}
// Oczekiwane wyjście:
// Znaleziono 16 na indeksie: 3
```

### `break` w zagnieżdżonych pętlach z etykietami

Etykiety pozwalają przerwać pętlę zewnętrzną z wnętrza pętli wewnętrznej:

```dart
void main() {
  // Etykieta 'szukaj:' pozwala przerwać zewnętrzną pętlę
  szukaj:
  for (var i = 0; i < 5; i++) {
    for (var j = 0; j < 5; j++) {
      if (i * j > 6) {
        print('Przerwano przy i=$i, j=$j (iloczyn=${i * j})');
        break szukaj; // przerywa ZEWNĘTRZNĄ pętlę
      }
    }
  }
  print('Koniec');
}
// Oczekiwane wyjście:
// Przerwano przy i=2, j=4 (iloczyn=8)
// Koniec
```

---

## Instrukcja `continue`

`continue` pomija resztę bieżącej iteracji i przechodzi do następnej.

### Pomijanie wybranych elementów

Poniższy przykład używa `continue` aby pominąć liczby parzyste:

```dart
void main() {
  // continue pomija bieżącą iterację i przechodzi do następnej
  for (var i = 1; i <= 10; i++) {
    if (i % 2 == 0) {
      continue; // pomija liczby parzyste
    }
    print(i);
  }
}
// Oczekiwane wyjście:
// 1
// 3
// 5
// 7
// 9
```

### `continue` z etykietą

Podobnie jak `break`, `continue` może używać etykiet w zagnieżdżonych pętlach:

```dart
void main() {
  var wynik = <String>[];

  // continue z etykietą — przeskakuje do następnej iteracji ZEWNĘTRZNEJ pętli
  zewnetrzna:
  for (var i = 1; i <= 3; i++) {
    for (var j = 1; j <= 3; j++) {
      if (j == 2) {
        continue zewnetrzna; // pomija resztę wewnętrznej pętli
      }
      wynik.add('($i,$j)');
    }
  }

  print(wynik); // tylko pary gdzie j=1, bo j=2 przerywa wewnętrzną pętlę
}
// Oczekiwane wyjście:
// [(1,1), (2,1), (3,1)]
```

---

## Ćwiczenie 1: Instrukcja `if`/`else`

### Opis problemu

Napisz funkcję `klasyfikujBMI(double bmi)` która zwraca kategorię na podstawie wartości BMI: poniżej 18.5 → "niedowaga", 18.5–24.9 → "norma", 25.0–29.9 → "nadwaga", 30.0 i więcej → "otyłość". Wypisz wynik dla podanych wartości.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `bmi = 22.5` | `BMI 22.5: norma` |
| `bmi = 31.2` | `BMI 31.2: otyłość` |

### Wskazówki

1. Użyj `if` / `else if` / `else` z progami liczbowymi
2. Zwróć uwagę na prawidłowe zakresy (włącznie/wyłącznie)

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String klasyfikujBMI(double bmi) {
  if (bmi < 18.5) {
    return 'niedowaga';
  } else if (bmi < 25.0) {
    return 'norma';
  } else if (bmi < 30.0) {
    return 'nadwaga';
  } else {
    return 'otyłość';
  }
}

void main() {
  var wartosci = [16.0, 22.5, 27.3, 31.2];
  for (var bmi in wartosci) {
    print('BMI $bmi: ${klasyfikujBMI(bmi)}');
  }
}
```

</details>

---

## Ćwiczenie 2: Pętla `for`

### Opis problemu

Napisz program, który wypisuje ciąg Fibonacciego do n-tego elementu (włącznie). Użyj klasycznej pętli `for`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `n = 8` | `0, 1, 1, 2, 3, 5, 8, 13` |
| `n = 5` | `0, 1, 1, 2, 3` |

### Wskazówki

1. Ciąg Fibonacciego: F(0)=0, F(1)=1, F(n)=F(n-1)+F(n-2)
2. Użyj dwóch zmiennych do przechowywania poprzednich wartości

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  var n = 8;
  var wynik = <int>[];

  var a = 0;
  var b = 1;

  for (var i = 0; i < n; i++) {
    wynik.add(a);
    var temp = a + b;
    a = b;
    b = temp;
  }

  print(wynik.join(', '));
}
```

</details>

---

## Ćwiczenie 3: Pętla `for-in`

### Opis problemu

Napisz funkcję `policzSamogloski(String tekst)` która zlicza samogłoski (a, e, i, o, u, ą, ę, ó) w podanym tekście (bez rozróżniania wielkości liter). Użyj pętli `for-in` do iteracji po znakach.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `'Dart jest super'` | `Samogłoski: 4` |
| `'Ąę'` | `Samogłoski: 2` |

### Wskazówki

1. Użyj `tekst.toLowerCase().split('')` aby uzyskać listę znaków
2. Stwórz zbiór (`Set`) zawierający wszystkie samogłoski do szybkiego sprawdzania

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
int policzSamogloski(String tekst) {
  const samogloski = {'a', 'e', 'i', 'o', 'u', 'ą', 'ę', 'ó'};
  var licznik = 0;

  for (var znak in tekst.toLowerCase().split('')) {
    if (samogloski.contains(znak)) {
      licznik++;
    }
  }
  return licznik;
}

void main() {
  var tekst = 'Dart jest super';
  print('Samogłoski: ${policzSamogloski(tekst)}');
}
```

</details>

---

## Ćwiczenie 4: Pętla `while`

### Opis problemu

Napisz program, który symuluje grę w zgadywanie liczb. Program "myśli" liczbę (ustaloną z góry), a gracz zgaduje kolejno od 1 w górę. Użyj pętli `while`. Program wypisuje liczbę prób potrzebnych do odgadnięcia.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| szukana: `7` | `Odgadnięto 7 po 7 próbach` |
| szukana: `3` | `Odgadnięto 3 po 3 próbach` |

### Wskazówki

1. Ustaw zmienną `zgadywana` na 0 i zwiększaj ją w każdej iteracji
2. Pętla kończy się gdy `zgadywana == szukana`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  var szukana = 7;
  var zgadywana = 0;
  var proby = 0;

  while (zgadywana != szukana) {
    zgadywana++;
    proby++;
  }

  print('Odgadnięto $szukana po $proby próbach');
}
```

</details>

---

## Ćwiczenie 5: `do-while`

### Opis problemu

Napisz program, który sumuje kolejne liczby naturalne (1, 2, 3, ...) dopóki suma nie przekroczy podanego limitu. Użyj pętli `do-while`. Wypisz ostatnią liczbę dodaną do sumy oraz wynik.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| limit: `20` | `Ostatnia dodana: 6, suma: 21` |
| limit: `10` | `Ostatnia dodana: 4, suma: 10` |

### Wskazówki

1. Użyj zmiennej `licznik` zaczynającej od 1 i zmiennej `suma`
2. Pamiętaj, że `do-while` sprawdza warunek PO dodaniu

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  var limit = 20;
  var suma = 0;
  var licznik = 0;

  do {
    licznik++;
    suma += licznik;
  } while (suma < limit);

  print('Ostatnia dodana: $licznik, suma: $suma');
}
```

</details>

---

## Ćwiczenie 6: `break` i `continue`

### Opis problemu

Napisz program, który z listy liczb wypisuje tylko liczby podzielne przez 3, pomijając liczby ujemne (użyj `continue`), i przerywa przetwarzanie gdy napotka wartość 0 (użyj `break`).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `[9, -3, 6, 12, -6, 0, 15, 21]` | `9\n6\n12` |
| `[3, 6, 0, 9]` | `3\n6` |

### Wskazówki

1. Najpierw sprawdź warunek `break` (wartość == 0)
2. Następnie sprawdź warunek `continue` (wartość < 0)
3. Na końcu sprawdź podzielność przez 3

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
void main() {
  var liczby = [9, -3, 6, 12, -6, 0, 15, 21];

  for (var n in liczby) {
    if (n == 0) break;        // przerwij na zerze
    if (n < 0) continue;      // pomiń ujemne
    if (n % 3 == 0) print(n); // wypisz podzielne przez 3
  }
}
```

</details>

---

**Następny moduł:** [Switch i pattern matching](02-switch-patterns.md)
**Poprzedni moduł:** [Null safety](../01-basics/03-null-safety.md)
