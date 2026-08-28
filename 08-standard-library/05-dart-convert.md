---
id: "8.5"
title: "Biblioteka standardowa — dart:convert"
difficulty: "intermediate"
section: "08-standard-library"
prerequisites:
  - "Zmienne, typy i operatory (dart:core)"
  - "Klasy i konstruktory"
  - "Event loop i Futures (async/await)"
---

# 8.5 Biblioteka standardowa — dart:convert

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Zmienne, typy i operatory](../01-basics/01-variables-types.md), [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Event loop i Futures (async/await)](../07-concurrency/01-async-await.md)
- **Cele nauki:**
  1. Kodować i dekodować dane w formacie JSON za pomocą `jsonEncode`, `jsonDecode` oraz klasy `JsonCodec`, w tym typy proste, kolekcje i zagnieżdżone obiekty
  2. Stosować kodeki znakowe i binarne: `utf8`, `latin1`, `base64` oraz narzędzie `LineSplitter`
  3. Implementować wzorce `toJson`/`fromJson` do serializacji i deserializacji własnych klas oraz wykonywać round-trip (obiekt → JSON → obiekt) z weryfikacją równości
  4. Tworzyć własne kodeki i konwertery przez rozszerzanie `Codec` i `Converter`
  5. Bezpiecznie obsługiwać błędne dane wejściowe za pomocą `try/catch` i wyjątku `FormatException`

---

## Czym jest dart:convert?

Biblioteka `dart:convert` dostarcza zestaw **kodeków** (codecs) i **konwerterów** (converters) do zamiany danych między różnymi reprezentacjami: obiektami Dart a tekstem JSON, ciągami znaków a bajtami (UTF-8, Latin-1), danymi binarnymi a tekstem Base64 itd.

Centralnym pojęciem jest **`Codec<S, T>`** — obiekt, który potrafi kodować wartość typu `S` do typu `T` (przez `encoder`) i dekodować z powrotem (przez `decoder`). Każdy kodek składa się z dwóch **konwerterów** (`Converter<S, T>`). Biblioteka udostępnia gotowe instancje kodeków jako stałe globalne: `json`, `utf8`, `latin1`, `base64`, `ascii`.

Aby korzystać z biblioteki, importujemy ją na początku pliku:

```dart
import 'dart:convert';
```

W tym module poznasz zarówno wygodne funkcje najwyższego poziomu (`jsonEncode`, `jsonDecode`), jak i pełne API kodeków, które daje większą kontrolę i możliwość składania (`fuse`) oraz tworzenia własnych konwerterów.

---

## JSON — kodowanie i dekodowanie typów prostych i kolekcji

JSON (JavaScript Object Notation) to najpopularniejszy format wymiany danych. W Dart obsługują go:

- `jsonEncode(obiekt)` — zamienia obiekt Dart na `String` w formacie JSON,
- `jsonDecode(tekst)` — parsuje `String` JSON na obiekty Dart (`Map`, `List`, `String`, `num`, `bool`, `null`).

Funkcje te są cienkimi opakowaniami na globalną instancję `json` (typu `JsonCodec`). Poniższy przykład pokazuje kodowanie i dekodowanie typów prostych oraz kolekcji.

```dart
import 'dart:convert';

void main() {
  // jsonEncode zamienia wartości Dart na tekst JSON
  print(jsonEncode(42));              // liczba
  print(jsonEncode('cześć'));         // string (w cudzysłowach)
  print(jsonEncode(true));            // bool
  print(jsonEncode([1, 2, 3]));       // lista -> tablica JSON
  print(jsonEncode({'a': 1, 'b': 2})); // mapa -> obiekt JSON

  // jsonDecode parsuje tekst JSON na obiekty Dart
  final lista = jsonDecode('[10, 20, 30]');
  print(lista);                       // List<dynamic>
  print(lista[1]);                    // dostęp jak do zwykłej listy

  final mapa = jsonDecode('{"imie": "Ala", "wiek": 30}');
  print(mapa['imie']);                // dostęp po kluczu
}
// Oczekiwane wyjście:
// 42
// "cześć"
// true
// [1,2,3]
// {"a":1,"b":2}
// [10, 20, 30]
// 20
// Ala
```

Wartości zwracane przez `jsonDecode` mają typ `dynamic`. Aby bezpiecznie z nich korzystać, zwykle rzutujemy je na oczekiwany typ (`as Map<String, dynamic>`, `as List<dynamic>`). Ta sama globalna instancja `json` udostępnia metody `encode` i `decode` — poniżej ten sam efekt osiągnięty przez pełne API kodeka.

```dart
import 'dart:convert';

void main() {
  // json to globalna instancja JsonCodec; encode/decode działają jak jsonEncode/jsonDecode
  final tekst = json.encode({'produkty': ['chleb', 'mleko'], 'liczba': 2});
  print(tekst);

  final dane = json.decode(tekst) as Map<String, dynamic>;
  final produkty = dane['produkty'] as List<dynamic>;
  print(produkty.length);
}
// Oczekiwane wyjście:
// {"produkty":["chleb","mleko"],"liczba":2}
// 2
```

### Formatowanie z wcięciami (pretty print)

Klasa `JsonEncoder` z opcją `withIndent` produkuje czytelny, sformatowany JSON — przydatny w logach i plikach konfiguracyjnych.

```dart
import 'dart:convert';

void main() {
  // JsonEncoder.withIndent tworzy encoder dodający wcięcia dla czytelności
  const encoder = JsonEncoder.withIndent('  ');

  final dane = {
    'nazwa': 'projekt',
    'wersja': 3,
    'tagi': ['dart', 'json'],
  };

  print(encoder.convert(dane));
}
// Oczekiwane wyjście:
// {
//   "nazwa": "projekt",
//   "wersja": 3,
//   "tagi": [
//     "dart",
//     "json"
//   ]
// }
```

---

## Obiekty zagnieżdżone (2+ poziomy)

JSON naturalnie reprezentuje struktury zagnieżdżone jako mapy zawierające inne mapy i listy. Poniższy przykład dekoduje strukturę o **dwóch poziomach zagnieżdżenia** (zamówienie → klient → adres) i sięga do głęboko położonych pól.

```dart
import 'dart:convert';

void main() {
  const surowyJson = '''
  {
    "numer": "Z-100",
    "klient": {
      "imie": "Ewa",
      "adres": {
        "miasto": "Kraków",
        "kod": "30-001"
      }
    },
    "pozycje": [
      {"produkt": "Klawiatura", "ilosc": 1},
      {"produkt": "Mysz", "ilosc": 2}
    ]
  }
  ''';

  final zamowienie = jsonDecode(surowyJson) as Map<String, dynamic>;

  // Poziom 1: pole na najwyższym poziomie
  print(zamowienie['numer']);

  // Poziom 2: zagnieżdżona mapa "klient"
  final klient = zamowienie['klient'] as Map<String, dynamic>;
  print(klient['imie']);

  // Poziom 3: mapa "adres" wewnątrz "klient"
  final adres = klient['adres'] as Map<String, dynamic>;
  print(adres['miasto']);

  // Lista map "pozycje" — sumujemy ilości
  final pozycje = zamowienie['pozycje'] as List<dynamic>;
  final sumaSztuk = pozycje
      .cast<Map<String, dynamic>>()
      .fold<int>(0, (suma, p) => suma + (p['ilosc'] as int));
  print('Łącznie sztuk: $sumaSztuk');
}
// Oczekiwane wyjście:
// Z-100
// Ewa
// Kraków
// Łącznie sztuk: 3
```

---

## Wzorzec toJson / fromJson dla własnych klas

`jsonEncode` nie wie, jak zamienić dowolny obiekt na JSON. Konwencja w Dart mówi, że klasa powinna udostępniać:

- metodę `Map<String, dynamic> toJson()` — `jsonEncode` **automatycznie** ją wywoła, jeśli napotka obiekt, którego nie potrafi zakodować bezpośrednio,
- konstruktor fabryczny `factory X.fromJson(Map<String, dynamic> json)` — do odtworzenia obiektu z mapy.

Poniższy przykład pokazuje klasę `Ksiazka` z metodą `toJson` oraz konstruktorem `fromJson`.

```dart
import 'dart:convert';

class Ksiazka {
  final String tytul;
  final String autor;
  final int rok;

  Ksiazka(this.tytul, this.autor, this.rok);

  // toJson zwraca mapę, którą jsonEncode potrafi zserializować
  Map<String, dynamic> toJson() => {
        'tytul': tytul,
        'autor': autor,
        'rok': rok,
      };

  // fromJson odtwarza obiekt z mapy uzyskanej z jsonDecode
  factory Ksiazka.fromJson(Map<String, dynamic> json) => Ksiazka(
        json['tytul'] as String,
        json['autor'] as String,
        json['rok'] as int,
      );

  @override
  String toString() => '$tytul ($autor, $rok)';
}

void main() {
  final ksiazka = Ksiazka('Diuna', 'Herbert', 1965);

  // jsonEncode automatycznie wywołuje toJson na obiekcie Ksiazka
  final tekst = jsonEncode(ksiazka);
  print(tekst);

  // jsonDecode zwraca mapę, którą przekazujemy do fromJson
  final mapa = jsonDecode(tekst) as Map<String, dynamic>;
  final odtworzona = Ksiazka.fromJson(mapa);
  print(odtworzona);
}
// Oczekiwane wyjście:
// {"tytul":"Diuna","autor":"Herbert","rok":1965}
// Diuna (Herbert, 1965)
```

---

## Round-trip: obiekt → JSON → obiekt z weryfikacją równości

**Round-trip** to sprawdzenie, że po serializacji do JSON i deserializacji z powrotem otrzymujemy obiekt **równoważny** oryginałowi. To fundamentalna właściwość poprawnej serializacji. Aby porównywać obiekty przez wartość, nadpisujemy operator `==` i getter `hashCode`.

Poniższy przykład używa klasy `Pracownik` z **3 polami**, w tym jednym **zagnieżdżonym obiektem** (`Dzial`). Weryfikujemy, że `original == odtworzony`.

```dart
import 'dart:convert';

// Zagnieżdżony obiekt używany jako pole klasy Pracownik
class Dzial {
  final String nazwa;
  final int pietro;

  Dzial(this.nazwa, this.pietro);

  Map<String, dynamic> toJson() => {'nazwa': nazwa, 'pietro': pietro};

  factory Dzial.fromJson(Map<String, dynamic> json) =>
      Dzial(json['nazwa'] as String, json['pietro'] as int);

  // == i hashCode pozwalają porównywać działy przez wartość
  @override
  bool operator ==(Object other) =>
      other is Dzial && other.nazwa == nazwa && other.pietro == pietro;

  @override
  int get hashCode => Object.hash(nazwa, pietro);
}

class Pracownik {
  final String imie;      // pole 1
  final int pensja;       // pole 2
  final Dzial dzial;      // pole 3 — zagnieżdżony obiekt

  Pracownik(this.imie, this.pensja, this.dzial);

  Map<String, dynamic> toJson() => {
        'imie': imie,
        'pensja': pensja,
        'dzial': dzial.toJson(), // rekurencyjna serializacja zagnieżdżonego obiektu
      };

  factory Pracownik.fromJson(Map<String, dynamic> json) => Pracownik(
        json['imie'] as String,
        json['pensja'] as int,
        // rekurencyjna deserializacja zagnieżdżonej mapy
        Dzial.fromJson(json['dzial'] as Map<String, dynamic>),
      );

  @override
  bool operator ==(Object other) =>
      other is Pracownik &&
      other.imie == imie &&
      other.pensja == pensja &&
      other.dzial == dzial;

  @override
  int get hashCode => Object.hash(imie, pensja, dzial);
}

void main() {
  final original = Pracownik('Marek', 8000, Dzial('IT', 4));

  // Krok 1: obiekt -> JSON string
  final tekst = jsonEncode(original);
  print(tekst);

  // Krok 2: JSON string -> mapa -> obiekt
  final odtworzony =
      Pracownik.fromJson(jsonDecode(tekst) as Map<String, dynamic>);

  // Krok 3: weryfikacja równości round-trip
  print('Round-trip poprawny: ${original == odtworzony}');
}
// Oczekiwane wyjście:
// {"imie":"Marek","pensja":8000,"dzial":{"nazwa":"IT","pietro":4}}
// Round-trip poprawny: true
```

---

## Strumieniowe parsowanie JSON — JsonDecoder z danymi porcjowanymi

Gdy JSON przychodzi w **porcjach** (np. z sieci lub dużego pliku), zamiast składać całość w pamięci możemy parsować go strumieniowo. `JsonDecoder` udostępnia metodę `startChunkedConversion`, która zwraca `sink` przyjmujący kolejne fragmenty tekstu i produkujący zdekodowany obiekt dopiero po otrzymaniu kompletnego JSON-a.

Poniższy przykład dzieli tekst JSON na fragmenty i podaje je do konwersji porcjowej, symulując przychodzące dane.

```dart
import 'dart:convert';

void main() {
  Object? wynik;

  // Sink docelowy odbiera gotowy, zdekodowany obiekt
  final wyjscie = ChunkedConversionSink<Object?>.withCallback((porcje) {
    wynik = porcje.single; // dekodowanie porcjowe zwraca listę z jednym wynikiem
  });

  // startChunkedConversion zwraca sink przyjmujący fragmenty tekstu
  final wejscie = const JsonDecoder().startChunkedConversion(wyjscie);

  // Symulujemy przychodzące fragmenty JSON-a
  wejscie.add('{"status":');
  wejscie.add('"ok","poz');
  wejscie.add('ycje":[1,2,3]}');
  wejscie.close(); // zamknięcie sygnalizuje koniec danych i uruchamia dekodowanie

  final mapa = wynik as Map<String, dynamic>;
  print(mapa['status']);
  print(mapa['pozycje']);
}
// Oczekiwane wyjście:
// ok
// [1, 2, 3]
```

Częstym wzorcem jest łączenie dekodera JSON z dekoderem UTF-8 przy strumieniowym czytaniu pliku lub odpowiedzi HTTP: `utf8.decoder.fuse(json.decoder)` tworzy jeden kodek zamieniający bajty bezpośrednio w obiekty Dart.

```dart
import 'dart:convert';

void main() {
  // fuse łączy dwa konwertery: bajty UTF-8 -> tekst -> obiekt JSON
  final konwerter = utf8.decoder.fuse(const JsonDecoder());

  // Bajty reprezentujące tekst JSON {"a":1}
  final bajty = utf8.encode('{"a":1}');

  final obiekt = konwerter.convert(bajty) as Map<String, dynamic>;
  print(obiekt['a']);
}
// Oczekiwane wyjście:
// 1
```

---

## UTF-8 — kodowanie znaków na bajty

Globalna instancja `utf8` (typu `Utf8Codec`) zamienia tekst na listę bajtów (`List<int>`) i z powrotem, zgodnie ze standardem UTF-8. Znaki spoza ASCII (np. polskie litery, emoji) zajmują wiele bajtów.

```dart
import 'dart:convert';

void main() {
  const tekst = 'Zażółć gęślą jaźń';

  // utf8.encode zamienia string na bajty (List<int>)
  final bajty = utf8.encode(tekst);
  print('Liczba bajtów: ${bajty.length}'); // więcej niż liczba znaków
  print('Liczba znaków: ${tekst.length}');

  // utf8.decode odtwarza string z bajtów
  final odtworzony = utf8.decode(bajty);
  print(odtworzony);
  print('Round-trip: ${odtworzony == tekst}');
}
// Oczekiwane wyjście:
// Liczba bajtów: 26
// Liczba znaków: 17
// Zażółć gęślą jaźń
// Round-trip: true
```

---

## Latin1 — kodowanie jednobajtowe (ISO-8859-1)

`Latin1Codec` (globalna instancja `latin1`) koduje znaki jako **pojedyncze bajty** w zakresie 0–255. Jest zwięzły dla tekstów zachodnioeuropejskich, ale **nie obsługuje** znaków spoza tego zakresu (np. emoji), co domyślnie kończy się wyjątkiem.

```dart
import 'dart:convert';

void main() {
  const tekst = 'Café'; // wszystkie znaki mieszczą się w Latin-1

  // latin1.encode koduje każdy znak jako jeden bajt (0-255)
  final bajty = latin1.encode(tekst);
  print(bajty); // 'é' ma kod 233 w Latin-1

  final odtworzony = latin1.decode(bajty);
  print(odtworzony);

  // Znak spoza zakresu Latin-1 powoduje błąd kodowania
  try {
    latin1.encode('emoji: 😀'); // poza zakresem 0-255
  } on ArgumentError catch (e) {
    print('Nie da się zakodować: ${e.runtimeType}');
  }
}
// Oczekiwane wyjście:
// [67, 97, 102, 233]
// Café
// Nie da się zakodować: ArgumentError
```

---

## Base64 — tekstowa reprezentacja danych binarnych

`Base64Codec` (globalna instancja `base64`) koduje **bajty** do bezpiecznego tekstowo formatu Base64 i z powrotem. Używa się go do osadzania danych binarnych (obrazów, kluczy) w tekście, np. w JSON lub URL-ach. Wariant `base64Url` używa alfabetu bezpiecznego dla adresów URL.

```dart
import 'dart:convert';

void main() {
  const wiadomosc = 'Dart 3.13';

  // Najpierw tekst -> bajty (UTF-8), potem bajty -> Base64
  final bajty = utf8.encode(wiadomosc);
  final zakodowane = base64.encode(bajty);
  print(zakodowane); // tekst Base64

  // Dekodowanie: Base64 -> bajty -> tekst
  final odkodowaneBajty = base64.decode(zakodowane);
  final odkodowanyTekst = utf8.decode(odkodowaneBajty);
  print(odkodowanyTekst);

  // base64Url używa alfabetu bezpiecznego dla URL (- i _ zamiast + i /)
  final urlowy = base64Url.encode(utf8.encode('dane?/+'));
  print(urlowy);
}
// Oczekiwane wyjście:
// RGFydCAzLjEz
// Dart 3.13
// ZGFuZT8vKw==
```

---

## LineSplitter — dzielenie tekstu na linie

`LineSplitter` to konwerter dzielący tekst na poszczególne linie, rozpoznając różne separatory (`\n`, `\r\n`, `\r`). Metoda `split` zwraca listę linii, a `LineSplitter` można też składać (`fuse`) z dekoderem UTF-8 do strumieniowego czytania plików linia po linii.

```dart
import 'dart:convert';

void main() {
  const tekst = 'pierwsza\ndruga\r\ntrzecia';

  // LineSplitter().convert dzieli tekst na listę linii, rozpoznając \n i \r\n
  final linie = const LineSplitter().convert(tekst);
  print(linie.length);
  for (final (index, linia) in linie.indexed) {
    print('$index: $linia');
  }
}
// Oczekiwane wyjście:
// 3
// 0: pierwsza
// 1: druga
// 2: trzecia
```

Poniżej `LineSplitter` przetwarza bajty (np. z pliku) najpierw zdekodowane z UTF-8 na tekst, a następnie podzielone na linie.

```dart
import 'dart:convert';

void main() {
  // Symulujemy bajty pliku tekstowego
  final bajty = utf8.encode('linia A\nlinia B\nlinia C');

  // Krok 1: bajty UTF-8 -> tekst
  final tekst = utf8.decode(bajty);

  // Krok 2: tekst -> podział na linie
  final linie = const LineSplitter().convert(tekst);

  print(linie);
}
// Oczekiwane wyjście:
// [linia A, linia B, linia C]
```

> **Uwaga:** Do strumieniowego czytania pliku linia po linii częściej używa się `LineSplitter` jako `StreamTransformer` na strumieniu bajtów, np. `plik.openRead().transform(utf8.decoder).transform(const LineSplitter())` — zobacz moduł [dart:io](../08-standard-library/04-dart-io.md).

---

## Własny Codec i Converter

Gdy potrzebujemy niestandardowej transformacji, tworzymy własny `Converter<S, T>` (nadpisując metodę `convert`) i opakowujemy parę konwerterów w `Codec<S, T>` (dostarczając gettery `encoder` i `decoder`). Dzięki temu nasza transformacja zyskuje pełne API kodeka: `encode`, `decode`, `fuse`, `inverted`.

Poniższy przykład implementuje prosty **szyfr Cezara** jako kodek: `encoder` przesuwa litery o stałą wartość, a `decoder` cofa to przesunięcie.

```dart
import 'dart:convert';

// Konwerter kodujący: przesuwa każdą literę o [przesuniecie]
class SzyfrCezaraEncoder extends Converter<String, String> {
  final int przesuniecie;
  const SzyfrCezaraEncoder(this.przesuniecie);

  @override
  String convert(String input) {
    final kody = input.codeUnits.map((k) {
      if (k >= 65 && k <= 90) return 65 + (k - 65 + przesuniecie) % 26; // A-Z
      if (k >= 97 && k <= 122) return 97 + (k - 97 + przesuniecie) % 26; // a-z
      return k; // pozostałe znaki bez zmian
    });
    return String.fromCharCodes(kody);
  }
}

// Konwerter dekodujący: przesuwa w przeciwną stronę (26 - przesuniecie)
class SzyfrCezaraDecoder extends Converter<String, String> {
  final int przesuniecie;
  const SzyfrCezaraDecoder(this.przesuniecie);

  @override
  String convert(String input) =>
      SzyfrCezaraEncoder(26 - (przesuniecie % 26)).convert(input);
}

// Codec łączy encoder i decoder w jeden obiekt
class SzyfrCezara extends Codec<String, String> {
  final int przesuniecie;
  const SzyfrCezara(this.przesuniecie);

  @override
  Converter<String, String> get encoder => SzyfrCezaraEncoder(przesuniecie);

  @override
  Converter<String, String> get decoder => SzyfrCezaraDecoder(przesuniecie);
}

void main() {
  const szyfr = SzyfrCezara(3);

  final zaszyfrowane = szyfr.encode('Dart');
  print(zaszyfrowane);

  final odszyfrowane = szyfr.decode(zaszyfrowane);
  print(odszyfrowane);
  print('Round-trip: ${odszyfrowane == 'Dart'}');
}
// Oczekiwane wyjście:
// Gduw
// Dart
// Round-trip: true
```

---

## Obsługa błędnych danych wejściowych — FormatException

Gdy `jsonDecode` otrzyma **niepoprawny** JSON, rzuca `FormatException`. Podobnie zachowują się inne dekodery (np. `base64.decode` przy błędnym wejściu). Nigdy nie zakładaj, że dane z zewnątrz są poprawne — zawsze opakuj dekodowanie w `try/catch`.

`FormatException` zawiera opis błędu (`message`), a często także sam błędny łańcuch (`source`) oraz pozycję błędu (`offset`), co ułatwia diagnozę.

```dart
import 'dart:convert';

void main() {
  // Przykład 1: niepoprawny JSON (brakuje wartości po dwukropku)
  const bledny = '{"imie": }';
  try {
    jsonDecode(bledny);
  } on FormatException catch (e) {
    // FormatException.offset wskazuje pozycję błędu w źródle
    print('Błąd JSON: ${e.message} (offset: ${e.offset})');
  }

  // Przykład 2: obcięty JSON (nieoczekiwany koniec danych)
  try {
    jsonDecode('[1, 2,');
  } on FormatException catch (e) {
    print('Błąd JSON: ${e.message}');
  }

  // Przykład 3: błędne dane Base64
  try {
    base64.decode('to-nie-jest-base64!');
  } on FormatException catch (e) {
    print('Błąd Base64: ${e.message}');
  }

  // Wzorzec bezpiecznego parsowania z wartością domyślną
  Map<String, dynamic>? bezpiecznyParse(String tekst) {
    try {
      return jsonDecode(tekst) as Map<String, dynamic>;
    } on FormatException {
      return null; // sygnalizujemy niepowodzenie zamiast propagować wyjątek
    }
  }

  print(bezpiecznyParse('{"ok": true}'));
  print(bezpiecznyParse('}}}nieprawidłowe'));
}
// Oczekiwane wyjście (dokładne komunikaty mogą się nieznacznie różnić):
// Błąd JSON: Unexpected character (offset: 9)
// Błąd JSON: Unexpected end of input
// Błąd Base64: Invalid character
// {ok: true}
// null
```

> **Uwaga:** `FormatException` dziedziczy po `Exception`, więc jest to błąd przewidywalny i obsługiwalny — w odróżnieniu od `Error`, który zwykle sygnalizuje błąd programisty. Zawsze łap `FormatException` przy przetwarzaniu danych z niezaufanych źródeł (sieć, pliki, wejście użytkownika).

---

## Ćwiczenie 1: Round-trip pojedynczej klasy

### Opis problemu

Zaimplementuj klasę `Punkt` reprezentującą punkt na płaszczyźnie z polami `x` (int) i `y` (int). Dodaj metodę `toJson()`, konstruktor `Punkt.fromJson(...)` oraz operator `==` i `hashCode`. Napisz funkcję `main`, która wykonuje round-trip: `Punkt` → JSON → `Punkt` i wypisuje wynik porównania.

**Poziom trudności:** basic

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Punkt(3, 4)` | `{"x":3,"y":4}` oraz `true` |
| `Punkt(0, -1)` | `{"x":0,"y":-1}` oraz `true` |

### Wskazówki

1. `toJson()` zwraca `{'x': x, 'y': y}`
2. `fromJson` odczytuje `json['x'] as int` i `json['y'] as int`
3. Nadpisz `==` i `hashCode`, aby porównanie działało przez wartość (użyj `Object.hash(x, y)`)

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:convert';

class Punkt {
  final int x;
  final int y;

  Punkt(this.x, this.y);

  Map<String, dynamic> toJson() => {'x': x, 'y': y};

  factory Punkt.fromJson(Map<String, dynamic> json) =>
      Punkt(json['x'] as int, json['y'] as int);

  @override
  bool operator ==(Object other) =>
      other is Punkt && other.x == x && other.y == y;

  @override
  int get hashCode => Object.hash(x, y);
}

void main() {
  final oryginal = Punkt(3, 4);
  final tekst = jsonEncode(oryginal);
  print(tekst);

  final odtworzony = Punkt.fromJson(jsonDecode(tekst) as Map<String, dynamic>);
  print(oryginal == odtworzony);
}
// Oczekiwane wyjście:
// {"x":3,"y":4}
// true
```

</details>

---

## Ćwiczenie 2: Serializacja hierarchii 3 klas

### Opis problemu

Zaimplementuj serializację JSON dla **hierarchii co najmniej 3 klas** z użyciem dziedziczenia i kompozycji, obsługując **pola nullable** oraz **kolekcje**. Zaprojektuj następujący model biblioteki:

- Abstrakcyjna klasa bazowa `Publikacja` z polami: `tytul` (String) i `rokWydania` (int). Definiuje `toJson()` dodające pole `typ` (dyskryminator) oraz abstrakcyjną metodę pozostawioną podklasom.
- Klasa `Ksiazka extends Publikacja` z dodatkowymi polami: `autor` (String) i `isbn` (String? — nullable, może być nieznany).
- Klasa `Czasopismo extends Publikacja` z dodatkowym polem `numer` (int).
- Klasa `Biblioteka` (kompozycja) zawierająca `nazwa` (String) i `List<Publikacja> publikacje` (kolekcja).

Zaimplementuj:

1. `toJson()` w każdej klasie (podklasy dołączają swoje pola do wspólnych z bazy, dodając dyskryminator `typ`),
2. fabrykę `Publikacja.fromJson(...)`, która na podstawie pola `typ` tworzy właściwą podklasę,
3. `Biblioteka.toJson()` serializujące listę publikacji oraz `Biblioteka.fromJson(...)` ją odtwarzające,
4. round-trip całej `Biblioteka` i weryfikację, że liczba publikacji i pola się zgadzają.

**Poziom trudności:** advanced

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Biblioteka('Miejska', [Ksiazka('Diuna', 1965, 'Herbert', null), Czasopismo('Nauka', 2024, 7)])` | JSON z listą 2 publikacji; `isbn` = `null`; round-trip: `2` publikacje |
| `Biblioteka('Pusta', [])` | `{"nazwa":"Pusta","publikacje":[]}`; round-trip: `0` publikacji |

### Wskazówki

1. Dodaj pole dyskryminatora `typ` w `toJson()`, np. `'typ': 'ksiazka'` / `'typ': 'czasopismo'`, aby `fromJson` wiedziało, którą podklasę utworzyć
2. Pole nullable `isbn` serializuj wprost — `jsonEncode` zapisze `null`; przy dekodowaniu użyj `json['isbn'] as String?`
3. `Biblioteka.toJson()` mapuje listę: `publikacje.map((p) => p.toJson()).toList()`
4. `Biblioteka.fromJson` rzutuje `json['publikacje'] as List<dynamic>` i mapuje każdy element przez `Publikacja.fromJson(e as Map<String, dynamic>)`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:convert';

abstract class Publikacja {
  final String tytul;
  final int rokWydania;

  Publikacja(this.tytul, this.rokWydania);

  // Bazowe pola + dyskryminator typu; podklasy rozszerzają tę mapę
  Map<String, dynamic> toJson();

  // Fabryka wybierająca podklasę na podstawie pola "typ"
  factory Publikacja.fromJson(Map<String, dynamic> json) {
    final typ = json['typ'] as String;
    return switch (typ) {
      'ksiazka' => Ksiazka.fromJson(json),
      'czasopismo' => Czasopismo.fromJson(json),
      _ => throw FormatException('Nieznany typ publikacji: $typ'),
    };
  }
}

class Ksiazka extends Publikacja {
  final String autor;
  final String? isbn; // pole nullable

  Ksiazka(super.tytul, super.rokWydania, this.autor, this.isbn);

  @override
  Map<String, dynamic> toJson() => {
        'typ': 'ksiazka',
        'tytul': tytul,
        'rokWydania': rokWydania,
        'autor': autor,
        'isbn': isbn, // null zostanie zapisane jako null
      };

  factory Ksiazka.fromJson(Map<String, dynamic> json) => Ksiazka(
        json['tytul'] as String,
        json['rokWydania'] as int,
        json['autor'] as String,
        json['isbn'] as String?, // odczyt pola nullable
      );
}

class Czasopismo extends Publikacja {
  final int numer;

  Czasopismo(super.tytul, super.rokWydania, this.numer);

  @override
  Map<String, dynamic> toJson() => {
        'typ': 'czasopismo',
        'tytul': tytul,
        'rokWydania': rokWydania,
        'numer': numer,
      };

  factory Czasopismo.fromJson(Map<String, dynamic> json) => Czasopismo(
        json['tytul'] as String,
        json['rokWydania'] as int,
        json['numer'] as int,
      );
}

class Biblioteka {
  final String nazwa;
  final List<Publikacja> publikacje; // kolekcja obiektów polimorficznych

  Biblioteka(this.nazwa, this.publikacje);

  Map<String, dynamic> toJson() => {
        'nazwa': nazwa,
        // serializacja kolekcji: każdą publikację zamieniamy na mapę
        'publikacje': publikacje.map((p) => p.toJson()).toList(),
      };

  factory Biblioteka.fromJson(Map<String, dynamic> json) => Biblioteka(
        json['nazwa'] as String,
        (json['publikacje'] as List<dynamic>)
            .map((e) => Publikacja.fromJson(e as Map<String, dynamic>))
            .toList(),
      );
}

void main() {
  final biblioteka = Biblioteka('Miejska', [
    Ksiazka('Diuna', 1965, 'Herbert', null), // isbn nieznany (null)
    Czasopismo('Nauka', 2024, 7),
  ]);

  // Round-trip: obiekt -> JSON -> obiekt
  final tekst = jsonEncode(biblioteka);
  print(tekst);

  final odtworzona =
      Biblioteka.fromJson(jsonDecode(tekst) as Map<String, dynamic>);

  print('Liczba publikacji: ${odtworzona.publikacje.length}');
  final pierwsza = odtworzona.publikacje.first as Ksiazka;
  print('Pierwsza to książka: ${pierwsza.tytul}, isbn=${pierwsza.isbn}');
}
// Oczekiwane wyjście:
// {"nazwa":"Miejska","publikacje":[{"typ":"ksiazka","tytul":"Diuna","rokWydania":1965,"autor":"Herbert","isbn":null},{"typ":"czasopismo","tytul":"Nauka","rokWydania":2024,"numer":7}]}
// Liczba publikacji: 2
// Pierwsza to książka: Diuna, isbn=null
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`jsonEncode` / `jsonDecode`** to najprostszy sposób pracy z JSON — kodują i dekodują typy proste, listy i mapy; `jsonDecode` zwraca wartości typu `dynamic`, które zwykle rzutujemy na oczekiwany typ.
2. **Wzorzec `toJson` / `fromJson`** to konwencja serializacji własnych klas — `jsonEncode` automatycznie wywołuje `toJson()`, a `fromJson` odtwarza obiekt z mapy; dla obiektów zagnieżdżonych serializacja jest rekurencyjna.
3. **Round-trip z weryfikacją równości** (nadpisanie `==` i `hashCode`) potwierdza poprawność serializacji: `original == fromJson(decode(encode(original)))`.
4. **Kodeki znakowe i binarne** — `utf8` (wielobajtowy, uniwersalny), `latin1` (jednobajtowy, ograniczony do 0–255), `base64` (bajty → bezpieczny tekst) — służą różnym zadaniom konwersji.
5. **`LineSplitter` i konwersja porcjowa** (`startChunkedConversion`, `fuse`) umożliwiają strumieniowe przetwarzanie dużych danych bez ładowania całości do pamięci.
6. **Własne kodeki** tworzymy rozszerzając `Converter` (metoda `convert`) i `Codec` (gettery `encoder`/`decoder`), zyskując pełne API: `encode`, `decode`, `fuse`, `inverted`.
7. **Zawsze obsługuj `FormatException`** przy dekodowaniu danych z niezaufanych źródeł — `try/catch` chroni aplikację przed awarią na błędnym wejściu.

---

**Poprzedni moduł:** [Biblioteka standardowa — dart:io](../08-standard-library/04-dart-io.md)
**Następny moduł:** [Biblioteka standardowa — dart:math i dart:typed_data](../08-standard-library/06-dart-math-typed.md)
