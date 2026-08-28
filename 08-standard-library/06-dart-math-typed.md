---
id: "8.6"
title: "Biblioteka standardowa — dart:math i dart:typed_data"
difficulty: "intermediate"
section: "08-standard-library"
prerequisites:
  - "Zmienne, typy i operatory (dart:core)"
  - "Klasy i konstruktory"
  - "Kolekcje: List, Set, Map"
---

# 8.6 Biblioteka standardowa — dart:math i dart:typed_data

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Zmienne, typy i operatory](../01-basics/01-variables-types.md), [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Biblioteka standardowa — dart:core](../08-standard-library/01-dart-core.md)
- **Cele nauki:**
  1. Generować liczby losowe za pomocą `Random` oraz `Random.secure()`, w tym liczby całkowite i zmiennoprzecinkowe w zadanym zakresie
  2. Wykorzystywać funkcje i stałe matematyczne z `dart:math`: `min`, `max`, `pow`, `sqrt`, `sin`/`cos`/`tan`, `pi`, `e`
  3. Wykonywać obliczenia geometryczne przy użyciu klas `Point`, `Rectangle` i `MutableRectangle`
  4. Reprezentować i manipulować danymi binarnymi za pomocą typów `Uint8List`, `Int32List`, `Float64List` oraz `ByteBuffer` i `ByteData`
  5. Odczytywać i zapisywać ustrukturyzowane formaty binarne z uwzględnieniem kolejności bajtów (`Endian`)

---

## Czym są dart:math i dart:typed_data?

`dart:math` to niewielka biblioteka dostarczająca narzędzia matematyczne: generator liczb losowych (`Random`), funkcje matematyczne (`min`, `max`, `pow`, `sqrt`, funkcje trygonometryczne), stałe matematyczne (`pi`, `e`) oraz proste klasy geometryczne (`Point`, `Rectangle`, `MutableRectangle`).

`dart:typed_data` udostępnia **typowane listy** i widoki na pamięć bajtową, pozwalające pracować z danymi binarnymi w sposób efektywny i przewidywalny. Zamiast zwykłej `List<int>`, gdzie każdy element jest pełnoprawnym obiektem Dart, typy takie jak `Uint8List` czy `Int32List` przechowują liczby w zwartym, ciągłym buforze pamięci — dokładnie tak, jak robią to formaty plików, protokoły sieciowe czy API systemowe.

Aby korzystać z tych bibliotek, importujemy je na początku pliku:

```dart
import 'dart:math';
import 'dart:typed_data';
```

W tym module poznasz najpierw narzędzia matematyczne, a następnie techniki pracy z danymi binarnymi.

---

## dart:math — generator liczb losowych (Random)

Klasa `Random` generuje pseudolosowe liczby. Udostępnia trzy podstawowe metody:

- `nextInt(max)` — losowa liczba całkowita z zakresu `0` (włącznie) do `max` (wyłącznie),
- `nextDouble()` — losowa liczba zmiennoprzecinkowa z zakresu `[0.0, 1.0)`,
- `nextBool()` — losowa wartość logiczna.

Konstruktor `Random(seed)` przyjmuje opcjonalne **ziarno** (seed) — podanie tej samej wartości daje **powtarzalną** sekwencję, co jest bezcenne w testach i symulacjach.

```dart
import 'dart:math';

void main() {
  // Random z ustalonym ziarnem daje powtarzalną sekwencję
  final rng = Random(42);

  print(rng.nextInt(6) + 1); // rzut kostką: 1-6
  print(rng.nextInt(6) + 1);
  print(rng.nextDouble());   // liczba z zakresu [0.0, 1.0)
  print(rng.nextBool());     // true lub false
}
// Oczekiwane wyjście (dla ziarna 42, deterministyczne):
// 2
// 1
// 0.6041479725361217
// false
```

### Losowanie w dowolnym zakresie

Aby uzyskać liczbę całkowitą w zakresie `[min, max]` (obie granice włącznie), przesuwamy wynik `nextInt`. Dla liczb zmiennoprzecinkowych w zakresie `[min, max)` skalujemy `nextDouble()`.

Poniższy przykład pokazuje generowanie liczb losowych w zadanym przedziale — typowe zadanie w grach, symulacjach i testach.

```dart
import 'dart:math';

// Losowa liczba całkowita w zakresie [min, max] (obie granice włącznie)
int losujInt(Random rng, int min, int max) =>
    min + rng.nextInt(max - min + 1);

// Losowa liczba zmiennoprzecinkowa w zakresie [min, max)
double losujDouble(Random rng, double min, double max) =>
    min + rng.nextDouble() * (max - min);

void main() {
  final rng = Random(7); // stałe ziarno dla powtarzalności

  print(losujInt(rng, 10, 20));      // liczba z [10, 20]
  print(losujInt(rng, -5, 5));       // liczba z [-5, 5]
  print(losujDouble(rng, 0.0, 100.0)); // liczba z [0.0, 100.0)
}
// Oczekiwane wyjście (dla ziarna 7, deterministyczne):
// 15
// 4
// 73.14634084349075
```

### Random.secure() — losowość kryptograficzna

Do zastosowań związanych z bezpieczeństwem (tokeny, hasła, klucze, sole) używamy `Random.secure()`. Korzysta on z kryptograficznie bezpiecznego źródła entropii systemu operacyjnego. **Nie przyjmuje ziarna** — z założenia jego wyników nie da się przewidzieć ani odtworzyć.

```dart
import 'dart:math';

void main() {
  // Random.secure() używa bezpiecznego kryptograficznie źródła losowości
  final secure = Random.secure();

  // Generujemy 8 losowych bajtów (np. dla tokenu sesji)
  final bajty = List<int>.generate(8, (_) => secure.nextInt(256));
  print('Liczba bajtów: ${bajty.length}');
  print('Wszystkie w zakresie 0-255: '
      '${bajty.every((b) => b >= 0 && b < 256)}');
}
// Oczekiwane wyjście (wartości losowe, struktura stała):
// Liczba bajtów: 8
// Wszystkie w zakresie 0-255: true
```

> **Uwaga:** Nigdy nie używaj zwykłego `Random()` do generowania sekretów (tokenów, haseł jednorazowych). Zwykły generator jest przewidywalny — do bezpieczeństwa zawsze `Random.secure()`.

---

## dart:math — funkcje i stałe matematyczne

`dart:math` udostępnia funkcje najwyższego poziomu oraz stałe:

- `min(a, b)` i `max(a, b)` — mniejsza/większa z dwóch wartości (działają na `num`),
- `pow(x, exponent)` — potęgowanie (zwraca `num`),
- `sqrt(x)` — pierwiastek kwadratowy,
- `sin(x)`, `cos(x)`, `tan(x)` — funkcje trygonometryczne (argument w **radianach**),
- `pi` — stała π ≈ 3.14159,
- `e` — podstawa logarytmu naturalnego ≈ 2.71828.

```dart
import 'dart:math';

void main() {
  print(min(3, 8));       // mniejsza z dwóch liczb
  print(max(3.5, 2.1));   // większa z dwóch liczb

  print(pow(2, 10));      // 2 do potęgi 10
  print(sqrt(144));       // pierwiastek kwadratowy

  print(pi);              // stała π
  print(e);               // podstawa logarytmu naturalnego

  // Argumenty funkcji trygonometrycznych są w radianach.
  // sin(π/2) = 1, cos(π) = -1
  print(sin(pi / 2));
  print(cos(pi).round()); // round() usuwa drobny błąd zmiennoprzecinkowy
}
// Oczekiwane wyjście:
// 3
// 3.5
// 1024
// 12.0
// 3.141592653589793
// 2.718281828459045
// 1.0
// -1
```

> **Uwaga:** Funkcje trygonometryczne przyjmują argumenty w **radianach**, nie w stopniach. Aby przeliczyć stopnie na radiany: `radiany = stopnie * pi / 180`. Wyniki `sin`/`cos` bywają obarczone drobnym błędem zmiennoprzecinkowym (np. `cos(pi)` daje `-1.0`, ale `sin(pi)` daje wartość bliską zeru, nie dokładnie `0`).

---

## dart:math — geometria: Point, Rectangle, MutableRectangle

Klasa `Point<T extends num>` reprezentuje punkt o współrzędnych `x` i `y`. Udostępnia m.in.:

- `distanceTo(other)` — odległość euklidesową do innego punktu,
- `magnitude` — odległość od początku układu współrzędnych,
- operatory `+` i `-` — dodawanie i odejmowanie punktów (jak wektorów).

Klasa `Rectangle<T extends num>` reprezentuje **niezmienialny** prostokąt (pola `left`, `top`, `width`, `height`, wyliczane `right`, `bottom`). `MutableRectangle` to jego zmienialny wariant, którego pola można modyfikować po utworzeniu.

Poniższy przykład wykonuje obliczenia geometryczne na punktach: odległość i dodawanie wektorów.

```dart
import 'dart:math';

void main() {
  final a = Point<int>(0, 0);
  final b = Point<int>(3, 4);

  // distanceTo liczy odległość euklidesową: sqrt(3^2 + 4^2) = 5
  print(a.distanceTo(b));

  // magnitude to odległość punktu od początku układu (0, 0)
  print(b.magnitude);

  // Operator + dodaje współrzędne (jak wektory)
  final suma = a + b + Point<int>(1, 1);
  print('(${suma.x}, ${suma.y})');

  // Operator - odejmuje współrzędne
  final roznica = b - Point<int>(1, 1);
  print('(${roznica.x}, ${roznica.y})');
}
// Oczekiwane wyjście:
// 5.0
// 5.0
// (4, 5)
// (2, 3)
```

Kolejny przykład używa `Rectangle` do obliczeń na prostokątach: pole powierzchni, sprawdzenie zawierania punktu oraz część wspólna dwóch prostokątów (`intersection`). Pokazuje też `MutableRectangle`, którego wymiary zmieniamy po utworzeniu.

```dart
import 'dart:math';

void main() {
  // Rectangle(left, top, width, height) — niezmienialny prostokąt
  final r = Rectangle<int>(0, 0, 10, 5);

  print('Pole: ${r.width * r.height}'); // 10 * 5
  print('Prawy brzeg: ${r.right}, dolny brzeg: ${r.bottom}');

  // containsPoint sprawdza, czy punkt leży wewnątrz prostokąta
  print(r.containsPoint(Point<int>(3, 2))); // wewnątrz
  print(r.containsPoint(Point<int>(20, 2))); // poza

  // intersection zwraca część wspólną dwóch prostokątów (lub null)
  final inny = Rectangle<int>(5, 2, 10, 10);
  final wspolny = r.intersection(inny);
  print('Część wspólna: $wspolny');

  // MutableRectangle pozwala zmieniać pola po utworzeniu
  final m = MutableRectangle<int>(0, 0, 4, 4);
  m.width = 8;
  m.height = 2;
  print('Nowe pole: ${m.width * m.height}');
}
// Oczekiwane wyjście:
// Pole: 50
// Prawy brzeg: 10, dolny brzeg: 5
// true
// false
// Część wspólna: Rectangle (5, 2) 5 x 3
// Nowe pole: 16
```

---

## dart:typed_data — typowane listy (Uint8List, Int32List, Float64List)

Typowane listy przechowują liczby w zwartym buforze bajtów zamiast jako osobne obiekty Dart. Każdy typ ma określony rozmiar i zakres elementu:

| Typ | Bajtów/element | Zakres elementu |
|-----|----------------|-----------------|
| `Uint8List` | 1 | 0 – 255 (bez znaku) |
| `Int8List` | 1 | -128 – 127 (ze znakiem) |
| `Int32List` | 4 | -2³¹ – 2³¹-1 |
| `Uint32List` | 4 | 0 – 2³²-1 |
| `Float64List` | 8 | liczby zmiennoprzecinkowe podwójnej precyzji |

Kluczowa cecha: przy zapisie wartości spoza zakresu następuje **zawinięcie** (wrap-around) przez obcięcie do dozwolonej liczby bitów — bez wyjątku. Poniższy przykład ilustruje to zachowanie oraz podstawowe operacje na typowanych listach.

```dart
import 'dart:typed_data';

void main() {
  // Uint8List o długości 4, wypełniona zerami
  final bajty = Uint8List(4);
  bajty[0] = 10;
  bajty[1] = 255;
  bajty[2] = 256; // poza zakresem 0-255 -> zawinięcie do 0
  bajty[3] = -1;  // poza zakresem -> zawinięcie do 255
  print(bajty);

  // Tworzenie z istniejącej listy
  final zListy = Uint8List.fromList([1, 2, 3]);
  print('Suma: ${zListy.reduce((a, b) => a + b)}');

  // Float64List przechowuje liczby zmiennoprzecinkowe
  final wartosci = Float64List.fromList([1.5, 2.5, 3.0]);
  print('Średnia: ${wartosci.reduce((a, b) => a + b) / wartosci.length}');

  // Int32List obsługuje liczby ujemne i duże
  final liczby = Int32List.fromList([-100, 100000, 2147483647]);
  print(liczby);
}
// Oczekiwane wyjście:
// [10, 255, 0, 255]
// Suma: 6
// Średnia: 2.3333333333333335
// [-100, 100000, 2147483647]
```

---

## dart:typed_data — ByteBuffer i ByteData

Wszystkie typowane listy współdzielą wspólną reprezentację pamięci — **`ByteBuffer`**. Dostęp do niego daje właściwość `.buffer`. Kluczowa idea: jeden bufor pamięci może być oglądany przez **różne widoki** (views). Ten sam ciąg bajtów można interpretować raz jako `Uint8List`, a raz jako `Int32List`.

**`ByteData`** to widok, który pozwala odczytywać i zapisywać liczby **różnych typów w dowolnym miejscu** bufora, z jawną kontrolą **kolejności bajtów** (`Endian`). Udostępnia metody takie jak `getInt32`, `setFloat64`, `getUint16` itd. To podstawowe narzędzie do parsowania ustrukturyzowanych formatów binarnych.

```dart
import 'dart:typed_data';

void main() {
  // Tworzymy bufor na 8 bajtów przez ByteData
  final data = ByteData(8);

  // setInt32(offset, value, endian) zapisuje 4-bajtową liczbę pod offsetem
  data.setInt32(0, 1000, Endian.big);
  data.setInt16(4, -5, Endian.big);
  data.setUint8(6, 200);

  // Odczyt tymi samymi metodami get...
  print(data.getInt32(0, Endian.big));
  print(data.getInt16(4, Endian.big));
  print(data.getUint8(6));

  // Ten sam bufor można oglądać jako Uint8List (widok bajtowy)
  final bajty = data.buffer.asUint8List();
  print('Liczba bajtów w buforze: ${bajty.length}');
}
// Oczekiwane wyjście:
// 1000
// -5
// 200
// Liczba bajtów w buforze: 8
```

---

## dart:typed_data — kolejność bajtów (Endian)

**Endianness** określa kolejność, w jakiej bajty wielobajtowej liczby są zapisane w pamięci:

- `Endian.big` (big-endian) — najbardziej znaczący bajt jako pierwszy. Standard w większości protokołów sieciowych („network byte order").
- `Endian.little` (little-endian) — najmniej znaczący bajt jako pierwszy. Używany przez większość procesorów (x86, ARM).
- `Endian.host` — kolejność natywna dla maszyny, na której działa program.

Wybór złej kolejności bajtów przy parsowaniu danych to jeden z najczęstszych błędów przy pracy z formatami binarnymi. Poniższy przykład pokazuje, jak ta sama liczba wygląda inaczej w obu porządkach.

```dart
import 'dart:typed_data';

void main() {
  final data = ByteData(4);

  // Zapisujemy tę samą liczbę raz jako big-, raz jako little-endian
  data.setUint32(0, 0x01020304, Endian.big);
  print(data.buffer.asUint8List()); // [1, 2, 3, 4]

  data.setUint32(0, 0x01020304, Endian.little);
  print(data.buffer.asUint8List()); // [4, 3, 2, 1]

  // Odczyt z błędną kolejnością daje zupełnie inną wartość
  final poprawne = data.getUint32(0, Endian.little);
  final bledne = data.getUint32(0, Endian.big);
  print('Poprawnie (little): $poprawne');
  print('Błędnie (big): $bledne');
}
// Oczekiwane wyjście:
// [1, 2, 3, 4]
// [4, 3, 2, 1]
// Poprawnie (little): 16909060
// Błędnie (big): 67305985
```

---

## Przykład: zapis i odczyt ustrukturyzowanego formatu binarnego

Realistyczne zadanie: zapisać rekord o stałej strukturze do bajtów i odczytać go z powrotem. Zdefiniujmy prosty format „pomiaru czujnika":

| Offset | Rozmiar | Pole | Typ |
|--------|---------|------|-----|
| 0 | 2 | id czujnika | `Uint16` |
| 2 | 4 | znacznik czasu | `Uint32` |
| 6 | 8 | temperatura | `Float64` |

Poniższy przykład serializuje rekord do `Uint8List` (big-endian), a następnie parsuje go z powrotem — dokładnie tak działa zapis do pliku binarnego lub ramki protokołu sieciowego.

```dart
import 'dart:typed_data';

// Rekord pomiaru o stałym rozmiarze 14 bajtów
class Pomiar {
  final int idCzujnika;   // Uint16
  final int znacznikCzasu; // Uint32
  final double temperatura; // Float64

  Pomiar(this.idCzujnika, this.znacznikCzasu, this.temperatura);

  // Serializacja: zapisujemy pola pod ustalonymi offsetami
  Uint8List doBajtow() {
    final data = ByteData(14);
    data.setUint16(0, idCzujnika, Endian.big);
    data.setUint32(2, znacznikCzasu, Endian.big);
    data.setFloat64(6, temperatura, Endian.big);
    return data.buffer.asUint8List();
  }

  // Deserializacja: odczytujemy pola spod tych samych offsetów
  factory Pomiar.zBajtow(Uint8List bajty) {
    // asByteData tworzy widok ByteData na buforze listy bajtów
    final data = bajty.buffer.asByteData();
    return Pomiar(
      data.getUint16(0, Endian.big),
      data.getUint32(2, Endian.big),
      data.getFloat64(6, Endian.big),
    );
  }

  @override
  String toString() =>
      'Pomiar(id: $idCzujnika, czas: $znacznikCzasu, temp: $temperatura)';
}

void main() {
  final oryginal = Pomiar(42, 1700000000, 21.5);

  // Rekord -> bajty
  final bajty = oryginal.doBajtow();
  print('Rozmiar ramki: ${bajty.length} bajtów');

  // Bajty -> rekord (round-trip)
  final odczytany = Pomiar.zBajtow(bajty);
  print(odczytany);
}
// Oczekiwane wyjście:
// Rozmiar ramki: 14 bajtów
// Pomiar(id: 42, czas: 1700000000, temp: 21.5)
```

---

## Przykład: parsowanie bajt po bajcie prostego protokołu

Wiele protokołów sieciowych używa nagłówka o stałym układzie: typ komunikatu, długość payloadu, a po nich same dane. Poniższy przykład parsuje taką ramkę: pierwszy bajt to typ, kolejne dwa (big-endian) to długość, a reszta to payload. To typowy wzorzec przy odbieraniu danych z gniazda TCP.

```dart
import 'dart:typed_data';

void main() {
  // Ramka: [typ=1][długość=5 jako Uint16 BE][H][e][l][l][o]
  final ramka = Uint8List.fromList([
    1, // typ komunikatu
    0, 5, // długość payloadu = 5 (Uint16 big-endian)
    72, 101, 108, 108, 111, // "Hello" w ASCII
  ]);

  final data = ramka.buffer.asByteData();

  // Odczyt nagłówka spod stałych offsetów
  final typ = data.getUint8(0);
  final dlugosc = data.getUint16(1, Endian.big);
  print('Typ: $typ, długość payloadu: $dlugosc');

  // sublistView tworzy widok na fragment bufora bez kopiowania
  final payload = Uint8List.sublistView(ramka, 3, 3 + dlugosc);
  final tekst = String.fromCharCodes(payload);
  print('Payload: $tekst');
}
// Oczekiwane wyjście:
// Typ: 1, długość payloadu: 5
// Payload: Hello
```

---

## Ćwiczenie 1 (dart:math): Symulacja rzutów kostką

### Opis problemu

Napisz funkcję `symulujRzuty(int liczbaRzutow, int seed)`, która symuluje `liczbaRzutow` rzutów sześcienną kostką (wartości 1–6) używając `Random` z podanym ziarnem `seed`, a następnie zwraca `Map<int, int>` — histogram zliczający, ile razy wypadła każda wartość. Wykorzystaj stałe ziarno, aby wynik był powtarzalny.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `symulujRzuty(6, 1)` | mapa z 6 kluczami (1–6), suma wartości = `6` |
| `symulujRzuty(1000, 42)` | suma wszystkich wartości histogramu = `1000` |

### Wskazówki

1. Rzut kostką to `rng.nextInt(6) + 1` (przesunięcie o 1, bo `nextInt(6)` daje 0–5)
2. Zliczaj wystąpienia w mapie: `histogram[wartosc] = (histogram[wartosc] ?? 0) + 1`
3. Suma wszystkich wartości histogramu musi równać się liczbie rzutów

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:math';

Map<int, int> symulujRzuty(int liczbaRzutow, int seed) {
  final rng = Random(seed); // stałe ziarno -> powtarzalny wynik
  final histogram = <int, int>{};

  for (var i = 0; i < liczbaRzutow; i++) {
    final wartosc = rng.nextInt(6) + 1; // rzut kostką: 1-6
    histogram[wartosc] = (histogram[wartosc] ?? 0) + 1;
  }
  return histogram;
}

void main() {
  final wynik = symulujRzuty(1000, 42);

  // Wypisujemy histogram posortowany po wartości oczka
  final klucze = wynik.keys.toList()..sort();
  for (final k in klucze) {
    print('$k: ${wynik[k]}');
  }

  final suma = wynik.values.reduce((a, b) => a + b);
  print('Suma rzutów: $suma'); // musi być 1000
}
// Oczekiwane wyjście (histogram deterministyczny dla ziarna 42; suma stała):
// 1: 168
// 2: 176
// 3: 162
// 4: 160
// 5: 162
// 6: 172
// Suma rzutów: 1000
```

</details>

---

## Ćwiczenie 2 (dart:typed_data): Kodowanie i dekodowanie listy liczb

### Opis problemu

Zaimplementuj dwie funkcje:

- `Uint8List zakoduj(List<int> liczby)` — serializuje listę liczb 32-bitowych do bufora bajtów. Format: najpierw `Uint32` (big-endian) z liczbą elementów, a następnie każda liczba jako `Int32` (big-endian).
- `List<int> odkoduj(Uint8List bajty)` — odczytuje bufor z powrotem do listy liczb.

Funkcje muszą tworzyć poprawny round-trip: `odkoduj(zakoduj(xs)) == xs` dla dowolnej listy liczb mieszczących się w `Int32`.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `zakoduj([10, -20, 30])` | bufor o długości `16` bajtów (4 na licznik + 3×4) |
| `odkoduj(zakoduj([10, -20, 30]))` | `[10, -20, 30]` |

### Wskazówki

1. Rozmiar bufora to `4 + liczby.length * 4` bajtów
2. Zapisuj licznik pod offsetem 0: `data.setUint32(0, liczby.length, Endian.big)`
3. Kolejne liczby zapisuj pod offsetami `4 + i * 4` metodą `setInt32`
4. Przy odczycie najpierw odczytaj licznik, potem w pętli odczytaj tyle liczb metodą `getInt32`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:typed_data';

Uint8List zakoduj(List<int> liczby) {
  // 4 bajty na licznik + po 4 bajty na każdą liczbę
  final data = ByteData(4 + liczby.length * 4);

  // Nagłówek: liczba elementów
  data.setUint32(0, liczby.length, Endian.big);

  // Ciało: kolejne liczby jako Int32 pod rosnącymi offsetami
  for (var i = 0; i < liczby.length; i++) {
    data.setInt32(4 + i * 4, liczby[i], Endian.big);
  }
  return data.buffer.asUint8List();
}

List<int> odkoduj(Uint8List bajty) {
  final data = bajty.buffer.asByteData();

  // Odczyt licznika, potem tylu elementów, ile zapowiada nagłówek
  final liczbaElementow = data.getUint32(0, Endian.big);
  final wynik = <int>[];
  for (var i = 0; i < liczbaElementow; i++) {
    wynik.add(data.getInt32(4 + i * 4, Endian.big));
  }
  return wynik;
}

void main() {
  final oryginal = [10, -20, 30, 2147483647, -2147483648];

  final bajty = zakoduj(oryginal);
  print('Rozmiar bufora: ${bajty.length} bajtów');

  final odczytane = odkoduj(bajty);
  print(odczytane);

  // Weryfikacja round-trip
  final zgodne = odczytane.length == oryginal.length &&
      List.generate(oryginal.length, (i) => odczytane[i] == oryginal[i])
          .every((x) => x);
  print('Round-trip poprawny: $zgodne');
}
// Oczekiwane wyjście:
// Rozmiar bufora: 24 bajtów
// [10, -20, 30, 2147483647, -2147483648]
// Round-trip poprawny: true
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`Random`** generuje liczby pseudolosowe (`nextInt`, `nextDouble`, `nextBool`); podanie ziarna daje powtarzalną sekwencję — bezcenne w testach i symulacjach. Do sekretów zawsze używaj **`Random.secure()`**.
2. **Funkcje matematyczne** (`min`, `max`, `pow`, `sqrt`, `sin`/`cos`/`tan`) i stałe (`pi`, `e`) pokrywają typowe obliczenia; funkcje trygonometryczne przyjmują argumenty w **radianach**.
3. **`Point` i `Rectangle`** wspierają geometrię: odległości, dodawanie wektorów, pole, zawieranie punktu i część wspólną; `MutableRectangle` pozwala zmieniać wymiary po utworzeniu.
4. **Typowane listy** (`Uint8List`, `Int32List`, `Float64List`) przechowują liczby w zwartym buforze; zapis wartości spoza zakresu powoduje **zawinięcie** przez obcięcie bitów, a nie wyjątek.
5. **`ByteBuffer` i `ByteData`** dają precyzyjny dostęp do pamięci: różne widoki na ten sam bufor oraz odczyt/zapis liczb różnych typów pod dowolnymi offsetami — podstawa parsowania formatów binarnych.
6. **`Endian`** (big/little) decyduje o kolejności bajtów; zła kolejność to jeden z najczęstszych błędów przy pracy z formatami binarnymi i protokołami sieciowymi (network byte order to big-endian).

---

**Poprzedni moduł:** [Biblioteka standardowa — dart:convert](../08-standard-library/05-dart-convert.md)
**Następny moduł:** [Rozszerzenia (extension methods i extension types)](../09-advanced/01-extensions.md)
