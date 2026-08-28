---
id: "8.1"
title: "Biblioteka dart:core"
difficulty: "intermediate"
section: "08-standard-library"
prerequisites:
  - "Zmienne i typy danych"
  - "Klasy i konstruktory"
  - "Klasy generyczne"
---

# 8.1 Biblioteka dart:core

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Zmienne i typy danych](../01-basics/01-variables-types.md), [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Klasy generyczne](../06-generics/01-generic-classes.md)
- **Cele nauki:**
  1. Sprawnie manipulować napisami (`String`): interpolacja, napisy wieloliniowe i surowe, metody przetwarzania oraz wyrażenia regularne (`RegExp`)
  2. Wykorzystywać typy liczbowe (`num`, `int`, `double`) oraz reprezentacje czasu (`DateTime`, `Duration`) do parsowania, obliczeń i porównań
  3. Dobierać i stosować kolekcje `List`, `Set` i `Map`, rozumiejąc złożoność czasową ich podstawowych operacji
  4. Korzystać z abstrakcji `Iterable`, `Iterator`, `Comparable`, `Pattern` oraz klasy `Uri` w typowych zadaniach

---

## String — napisy

Klasa `String` reprezentuje niezmienną (immutable) sekwencję jednostek kodowych UTF-16. Każda operacja „modyfikująca" napis w rzeczywistości zwraca nowy obiekt `String`.

### Interpolacja napisów

Interpolacja pozwala wstawić wartość wyrażenia bezpośrednio do napisu za pomocą `$nazwa` (dla prostej zmiennej) lub `${wyrażenie}` (dla dowolnego wyrażenia).

```dart
// Interpolacja prostych zmiennych i złożonych wyrażeń
void main() {
  var imie = 'Anna';
  var wiek = 30;
  // $zmienna wstawia wartość, ${...} pozwala na dowolne wyrażenie
  print('Cześć, $imie!');
  print('Za rok będziesz mieć ${wiek + 1} lat.');
  print('Imię wielkimi literami: ${imie.toUpperCase()}');
}
// Oczekiwane wyjście:
// Cześć, Anna!
// Za rok będziesz mieć 31 lat.
// Imię wielkimi literami: ANNA
```

### Napisy wieloliniowe i surowe

Potrójny cudzysłów (`'''` lub `"""`) tworzy napis wieloliniowy, zachowując znaki końca linii. Prefiks `r` tworzy napis surowy (raw), w którym znaki ucieczki (`\n`, `\t`) nie są interpretowane.

```dart
// Napis wieloliniowy oraz napis surowy (raw string)
void main() {
  // Napis wieloliniowy zachowuje podziały wierszy
  var wiersz = '''
Pierwsza linia
Druga linia
Trzecia linia''';
  print(wiersz);

  // Napis surowy — r'...' traktuje \n dosłownie, nie jako nową linię
  var sciezka = r'C:\Users\Anna\dokumenty';
  print(sciezka);

  // Bez prefiksu r sekwencja \n oznaczałaby nową linię
  print('Linia1\nLinia2');
}
// Oczekiwane wyjście:
// Pierwsza linia
// Druga linia
// Trzecia linia
// C:\Users\Anna\dokumenty
// Linia1
// Linia2
```

### Metody przetwarzania napisów

`String` udostępnia bogaty zestaw metod: `split` (podział na listę), `trim` (usuwanie białych znaków), `contains` (sprawdzenie zawierania), `replaceAll` (zamiana wszystkich wystąpień) i wiele innych.

```dart
// Popularne metody: split, trim, contains, replaceAll, substring, indexOf
void main() {
  var dane = '  jabłko,gruszka,śliwka  ';

  // trim usuwa białe znaki z początku i końca
  var oczyszczone = dane.trim();
  print('Po trim: "$oczyszczone"');

  // split dzieli napis wg separatora i zwraca List<String>
  var owoce = oczyszczone.split(',');
  print('Lista: $owoce (${owoce.length} elementy)');

  // contains sprawdza obecność podłańcucha
  print('Zawiera "gruszka": ${oczyszczone.contains('gruszka')}');

  // replaceAll zamienia wszystkie wystąpienia
  print('Zamiana: ${oczyszczone.replaceAll(',', ' | ')}');

  // startsWith, endsWith, toUpperCase, substring
  print('Wielkie litery: ${oczyszczone.toUpperCase()}');
  print('Pierwsze 5 znaków: ${oczyszczone.substring(0, 5)}');
}
// Oczekiwane wyjście:
// Po trim: "jabłko,gruszka,śliwka"
// Lista: [jabłko, gruszka, śliwka] (3 elementy)
// Zawiera "gruszka": true
// Zamiana: jabłko | gruszka | śliwka
// Wielkie litery: JABŁKO,GRUSZKA,ŚLIWKA
// Pierwsze 5 znaków: jabłk
```

### Wyrażenia regularne (RegExp)

Klasa `RegExp` implementuje interfejs `Pattern` i pozwala dopasowywać wzorce, wyodrębniać grupy oraz dokonywać zamian. Wzorce najlepiej zapisywać jako napisy surowe (`r'...'`), aby uniknąć podwójnego ekranowania.

```dart
// RegExp: dopasowanie, grupy przechwytujące i zamiany
void main() {
  // Wzorzec daty w formacie RRRR-MM-DD z grupami przechwytującymi
  var wzorzec = RegExp(r'(\d{4})-(\d{2})-(\d{2})');
  var tekst = 'Spotkanie 2024-03-15, deadline 2024-04-01.';

  // firstMatch zwraca pierwsze dopasowanie lub null
  var pierwsze = wzorzec.firstMatch(tekst);
  if (pierwsze != null) {
    // group(0) to całe dopasowanie, group(1..n) to grupy w nawiasach
    print('Cała data: ${pierwsze.group(0)}');
    print('Rok: ${pierwsze.group(1)}, miesiąc: ${pierwsze.group(2)}');
  }

  // allMatches zwraca wszystkie dopasowania jako Iterable<RegExpMatch>
  var wszystkie = wzorzec.allMatches(tekst);
  print('Znalezione daty: ${wszystkie.map((m) => m.group(0)).toList()}');

  // hasMatch zwraca true, jeśli wzorzec pasuje gdziekolwiek
  print('Zawiera datę: ${wzorzec.hasMatch(tekst)}');

  // replaceAllMapped (metoda String) pozwala przekształcić każde dopasowanie
  var przeformatowane = tekst.replaceAllMapped(
    wzorzec,
    (m) => '${m.group(3)}.${m.group(2)}.${m.group(1)}', // DD.MM.RRRR
  );
  print('Przeformatowane: $przeformatowane');
}
// Oczekiwane wyjście:
// Cała data: 2024-03-15
// Rok: 2024, miesiąc: 03
// Znalezione daty: [2024-03-15, 2024-04-01]
// Zawiera datę: true
// Przeformatowane: Spotkanie 15.03.2024, deadline 01.04.2024.
```

---

## num, int i double — typy liczbowe

`num` to wspólny nadtyp dla `int` (liczby całkowite) i `double` (liczby zmiennoprzecinkowe). Wszystkie trzy typy udostępniają metody parsowania, zaokrąglania i porównywania.

```dart
// Parsowanie, arytmetyka, zaokrąglanie i porównania liczb
void main() {
  // Parsowanie z napisu — parse rzuca wyjątek, tryParse zwraca null przy błędzie
  int calkowita = int.parse('42');
  double zmiennoprzecinkowa = double.parse('3.14');
  int? niepewna = int.tryParse('nie-liczba'); // zwraca null zamiast wyjątku
  print('Sparsowane: $calkowita, $zmiennoprzecinkowa, $niepewna');

  // Metody zaokrąglania na double
  var wartosc = 3.678;
  print('round: ${wartosc.round()}');     // najbliższa całkowita
  print('floor: ${wartosc.floor()}');     // w dół
  print('ceil: ${wartosc.ceil()}');       // w górę
  print('truncate: ${wartosc.truncate()}'); // odcięcie części ułamkowej
  print('toStringAsFixed: ${wartosc.toStringAsFixed(2)}'); // 2 miejsca po przecinku

  // Metody int: dzielenie całkowite, reszta, parzystość
  print('7 ~/ 2 = ${7 ~/ 2}, 7 % 2 = ${7 % 2}');
  print('Czy 10 parzyste: ${10.isEven}');

  // Porównania: compareTo, clamp, abs
  print('compareTo: ${5.compareTo(8)}'); // -1 (5 < 8)
  print('clamp: ${15.clamp(0, 10)}');    // ogranicza do zakresu -> 10
  print('abs: ${(-7).abs()}');
}
// Oczekiwane wyjście:
// Sparsowane: 42, 3.14, null
// round: 4
// floor: 3
// ceil: 4
// truncate: 3
// toStringAsFixed: 3.68
// 7 ~/ 2 = 3, 7 % 2 = 1
// Czy 10 parzyste: true
// compareTo: -1
// clamp: 10
// abs: 7
```

---

## DateTime i Duration — czas

`DateTime` reprezentuje moment w czasie, a `Duration` odcinek czasu. Razem umożliwiają tworzenie dat, parsowanie z napisów, obliczanie różnic i arytmetykę czasową.

```dart
// Tworzenie, parsowanie, różnice i arytmetyka czasu
void main() {
  // Tworzenie konkretnej daty (rok, miesiąc, dzień, godzina, minuta)
  var start = DateTime(2024, 1, 1, 9, 0);
  print('Data startowa: $start');

  // Parsowanie z napisu ISO 8601
  var koniec = DateTime.parse('2024-03-15 17:30:00');
  print('Data końcowa: $koniec');

  // Różnica między datami zwraca obiekt Duration
  Duration roznica = koniec.difference(start);
  print('Różnica w dniach: ${roznica.inDays}');
  print('Różnica w godzinach: ${roznica.inHours}');

  // Arytmetyka: dodawanie i odejmowanie czasu przez add/subtract
  var zaTydzien = start.add(const Duration(days: 7));
  var dzienWczesniej = start.subtract(const Duration(days: 1));
  print('Za tydzień: $zaTydzien');
  print('Dzień wcześniej: $dzienWczesniej');

  // Porównania dat
  print('koniec po start: ${koniec.isAfter(start)}');

  // Konstruowanie Duration z różnych jednostek
  var czasTrwania = const Duration(hours: 1, minutes: 30);
  print('Czas trwania w minutach: ${czasTrwania.inMinutes}');
}
// Oczekiwane wyjście:
// Data startowa: 2024-01-01 09:00:00.000
// Data końcowa: 2024-03-15 17:30:00.000
// Różnica w dniach: 74
// Różnica w godzinach: 1784
// Za tydzień: 2024-01-08 09:00:00.000
// Dzień wcześniej: 2023-12-31 09:00:00.000
// koniec po start: true
// Czas trwania w minutach: 90
```

---

## Uri — adresy i identyfikatory zasobów

Klasa `Uri` służy do parsowania, konstruowania i kodowania adresów URI/URL. Udostępnia dostęp do poszczególnych komponentów: schematu, hosta, ścieżki i parametrów zapytania.

```dart
// Parsowanie i konstruowanie URI oraz kodowanie komponentów
void main() {
  // Parsowanie istniejącego adresu
  var uri = Uri.parse('https://przyklad.pl:8080/szukaj?q=dart&limit=10#sekcja');
  print('Schemat: ${uri.scheme}');
  print('Host: ${uri.host}, port: ${uri.port}');
  print('Ścieżka: ${uri.path}');
  print('Parametry: ${uri.queryParameters}');
  print('Fragment: ${uri.fragment}');

  // Budowanie URI z komponentów — queryParameters są kodowane automatycznie
  var zbudowane = Uri(
    scheme: 'https',
    host: 'api.przyklad.pl',
    path: '/uzytkownicy',
    queryParameters: {'imie': 'Jan Kowalski', 'aktywny': 'true'},
  );
  print('Zbudowany URI: $zbudowane');

  // Kodowanie pojedynczego komponentu (spacje -> %20)
  print('Zakodowane: ${Uri.encodeComponent('Dart & Flutter')}');
}
// Oczekiwane wyjście:
// Schemat: https
// Host: przyklad.pl, port: 8080
// Ścieżka: /szukaj
// Parametry: {q: dart, limit: 10}
// Fragment: sekcja
// Zbudowany URI: https://api.przyklad.pl/uzytkownicy?imie=Jan+Kowalski&aktywny=true
// Zakodowane: Dart%20%26%20Flutter
```

---

## Iterable i Iterator — sekwencje i iteracja

`Iterable<E>` to abstrakcja reprezentująca sekwencję elementów, którą można przeglądać. `Iterator<E>` to obiekt realizujący przechodzenie po elementach za pomocą `moveNext()` i `current`. Pętla `for-in` korzysta z iteratora niejawnie.

```dart
// Iterable udostępnia leniwe metody map/where/take; Iterator działa krok po kroku
void main() {
  // Iterable stworzone leniwie — operacje nie wykonują się aż do konsumpcji
  Iterable<int> liczby = Iterable.generate(5, (i) => i * i); // 0,1,4,9,16

  // Metody Iterable: where (filtr), map (transformacja), reduce (agregacja)
  var parzyste = liczby.where((n) => n.isEven);
  print('Kwadraty parzyste: ${parzyste.toList()}');
  print('Suma kwadratów: ${liczby.reduce((a, b) => a + b)}');
  print('Pierwsze 3: ${liczby.take(3).toList()}');

  // Bezpośrednie użycie Iterator — moveNext() przesuwa, current daje wartość
  var iterator = ['a', 'b', 'c'].iterator;
  while (iterator.moveNext()) {
    print('Element: ${iterator.current}');
  }
}
// Oczekiwane wyjście:
// Kwadraty parzyste: [0, 4, 16]
// Suma kwadratów: 30
// Pierwsze 3: [0, 1, 4]
// Element: a
// Element: b
// Element: c
```

---

## Comparable — porządkowanie obiektów

Interfejs `Comparable<T>` definiuje naturalny porządek elementów przez metodę `compareTo`. Klasy implementujące ten interfejs można sortować bezpośrednio metodą `List.sort`.

```dart
// Implementacja Comparable pozwala sortować obiekty własnego typu
class Pracownik implements Comparable<Pracownik> {
  final String imie;
  final int pensja;

  Pracownik(this.imie, this.pensja);

  // compareTo: ujemne gdy this < other, 0 gdy równe, dodatnie gdy this > other
  @override
  int compareTo(Pracownik inny) => pensja.compareTo(inny.pensja);

  @override
  String toString() => '$imie ($pensja zł)';
}

void main() {
  var pracownicy = [
    Pracownik('Anna', 8000),
    Pracownik('Bartek', 6000),
    Pracownik('Celina', 10000),
  ];

  // sort korzysta z compareTo — sortuje rosnąco wg pensji
  pracownicy.sort();
  print('Wg pensji rosnąco: $pracownicy');

  // Sortowanie malejące — odwrócenie porównania
  pracownicy.sort((a, b) => b.compareTo(a));
  print('Wg pensji malejąco: $pracownicy');
}
// Oczekiwane wyjście:
// Wg pensji rosnąco: [Bartek (6000 zł), Anna (8000 zł), Celina (10000 zł)]
// Wg pensji malejąco: [Celina (10000 zł), Anna (8000 zł), Bartek (6000 zł)]
```

---

## Pattern — wspólny interfejs wzorców

`Pattern` to interfejs implementowany zarówno przez `String`, jak i `RegExp`. Dzięki temu metody takie jak `split`, `contains`, `replaceAll` przyjmują oba typy wzorców w jednolity sposób.

```dart
// String i RegExp implementują Pattern — metody działają z oboma
void main() {
  var tekst = 'kot123pies456ryba';

  // Pattern jako String — prosty separator literowy
  Pattern separatorTekstowy = 'pies';
  print('Split po napisie: ${tekst.split(separatorTekstowy)}');

  // Pattern jako RegExp — separator to dowolna sekwencja cyfr
  Pattern separatorRegexp = RegExp(r'\d+');
  print('Split po cyfrach: ${tekst.split(separatorRegexp)}');

  // matchAsPrefix — sprawdza dopasowanie wzorca na początku napisu
  var dopasowanie = RegExp(r'[a-z]+').matchAsPrefix(tekst);
  print('Prefiks literowy: ${dopasowanie?.group(0)}');
}
// Oczekiwane wyjście:
// Split po napisie: [kot123, 456ryba]
// Split po cyfrach: [kot, pies, ryba]
// Prefiks literowy: kot
```

---

## Kolekcje: List, Set, Map

`dart:core` definiuje trzy podstawowe kolekcje. Poniższa tabela podsumowuje ich charakterystykę oraz **złożoność czasową** typowych operacji dla domyślnych implementacji (`List` — tablica dynamiczna, `Set`/`Map` — tablice haszujące zachowujące kolejność wstawiania).

| Kolekcja | Uporządkowanie | Unikalność | Wyszukiwanie | Wstawianie | Usuwanie | Iteracja |
|----------|----------------|-----------|--------------|------------|----------|----------|
| `List<E>` | kolejność indeksów | nie | O(1) po indeksie, O(n) po wartości | O(1) amortyzowane na końcu, O(n) w środku | O(n) | O(n) |
| `Set<E>` | kolejność wstawiania | tak | O(1) średnio | O(1) średnio | O(1) średnio | O(n) |
| `Map<K,V>` | kolejność wstawiania | klucze unikalne | O(1) średnio po kluczu | O(1) średnio | O(1) średnio | O(n) |

### List — lista uporządkowana

`List<E>` to uporządkowana kolekcja z dostępem po indeksie. Dostęp `list[i]` jest O(1), ale wyszukiwanie wartości i wstawianie w środku wymaga przesuwania elementów (O(n)).

```dart
// Konstruktory listy oraz kluczowe metody z ich złożonością
void main() {
  // Literał listy — najczęstszy sposób
  var owoce = <String>['jabłko', 'gruszka'];

  // List.filled — lista o zadanym rozmiarze wypełniona wartością
  var zera = List<int>.filled(3, 0);

  // List.generate — lista tworzona funkcją indeksującą
  var kwadraty = List<int>.generate(4, (i) => i * i);

  print('Owoce: $owoce, zera: $zera, kwadraty: $kwadraty');

  // add O(1) amortyzowane, insert O(n) (przesuwa elementy)
  owoce.add('śliwka');           // dodanie na końcu
  owoce.insert(0, 'banan');      // wstawienie na początku
  print('Po dodaniu: $owoce');

  // Dostęp po indeksie O(1), indexOf O(n)
  print('Element [1]: ${owoce[1]}');
  print('Indeks "śliwka": ${owoce.indexOf('śliwka')}');

  // removeAt O(n), sublist tworzy kopię fragmentu
  owoce.removeAt(0);
  print('Po usunięciu: $owoce');
}
// Oczekiwane wyjście:
// Owoce: [jabłko, gruszka], zera: [0, 0, 0], kwadraty: [0, 1, 4, 9]
// Po dodaniu: [banan, jabłko, gruszka, śliwka]
// Element [1]: jabłko
// Indeks "śliwka": 3
// Po usunięciu: [jabłko, gruszka, śliwka]
```

### Set — zbiór unikalnych elementów

`Set<E>` przechowuje unikalne elementy bez powtórzeń. Sprawdzanie przynależności (`contains`) oraz dodawanie są średnio O(1), co czyni zbiór idealnym do usuwania duplikatów i szybkich testów obecności.

```dart
// Konstruktory zbioru, operacje mnogościowe i deduplikacja (O(1) średnio)
void main() {
  // Literał zbioru oraz konstruktor Set.from
  var a = <int>{1, 2, 3, 3, 2}; // duplikaty są ignorowane
  var b = Set<int>.from([3, 4, 5]);
  print('Zbiór a: $a'); // {1, 2, 3}

  // contains O(1) średnio — szybki test przynależności
  print('a zawiera 2: ${a.contains(2)}');

  // Operacje mnogościowe: suma, przecięcie, różnica
  print('Suma: ${a.union(b)}');
  print('Przecięcie: ${a.intersection(b)}');
  print('Różnica: ${a.difference(b)}');

  // Deduplikacja listy przez konwersję do Set i z powrotem
  var zDuplikatami = [1, 1, 2, 3, 3, 3];
  print('Bez duplikatów: ${zDuplikatami.toSet().toList()}');
}
// Oczekiwane wyjście:
// Zbiór a: {1, 2, 3}
// a zawiera 2: true
// Suma: {1, 2, 3, 4, 5}
// Przecięcie: {3}
// Różnica: {1, 2}
// Bez duplikatów: [1, 2, 3]
```

### Map — mapa klucz-wartość

`Map<K,V>` odwzorowuje unikalne klucze na wartości. Odczyt i zapis po kluczu są średnio O(1). Domyślna implementacja zachowuje kolejność wstawiania kluczy.

```dart
// Konstruktory mapy oraz kluczowe metody (dostęp po kluczu O(1) średnio)
void main() {
  // Literał mapy oraz Map.fromEntries
  var ceny = <String, double>{'jabłko': 3.5, 'gruszka': 4.0};

  // Odczyt po kluczu O(1); nieistniejący klucz zwraca null
  print('Cena jabłka: ${ceny['jabłko']}');
  print('Cena banana: ${ceny['banan']}'); // null

  // Zapis/aktualizacja po kluczu O(1)
  ceny['śliwka'] = 5.0;
  ceny['jabłko'] = 3.8; // nadpisanie istniejącej wartości

  // putIfAbsent dodaje tylko gdy klucz nie istnieje
  ceny.putIfAbsent('gruszka', () => 99.0); // nie zmieni — klucz istnieje

  // containsKey O(1), keys/values zwracają Iterable
  print('Zawiera "śliwka": ${ceny.containsKey('śliwka')}');
  print('Klucze: ${ceny.keys.toList()}');
  print('Wartości: ${ceny.values.toList()}');

  // update z ifAbsent, forEach do iteracji
  ceny.update('banan', (v) => v + 1, ifAbsent: () => 2.0);
  print('Mapa końcowa: $ceny');
}
// Oczekiwane wyjście:
// Cena jabłka: 3.5
// Cena banana: null
// Zawiera "śliwka": true
// Klucze: [jabłko, gruszka, śliwka]
// Wartości: [3.8, 4.0, 5.0]
// Mapa końcowa: {jabłko: 3.8, gruszka: 4.0, śliwka: 5.0, banan: 2.0}
```

---

## Ćwiczenie 1

### Opis problemu

Napisz funkcję `policzSlowa(String tekst)`, która zwraca `Map<String, int>` zliczającą wystąpienia każdego słowa (bez rozróżniania wielkości liter). Słowa są oddzielone dowolną liczbą białych znaków i znaków interpunkcyjnych. Interpunkcję (`.,!?;:`) należy usunąć, a puste elementy pominąć. Wynik uporządkuj malejąco według liczby wystąpień — użyj listy par posortowanej i zbuduj z niej mapę.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `'Kot i pies. Kot!'` | `{kot: 2, i: 1, pies: 1}` |
| `'Ala ma kota, kota ma Ala'` | `{ala: 2, ma: 2, kota: 2}` |

### Wskazówki

1. Zamień tekst na małe litery (`toLowerCase`), a następnie użyj `RegExp(r'[a-ząćęłńóśźż]+')` z `allMatches`, aby wyodrębnić same słowa
2. Zliczaj w `Map<String,int>` za pomocą `update(słowo, (v) => v + 1, ifAbsent: () => 1)`
3. Aby posortować, pobierz `map.entries.toList()`, wywołaj `sort` porównując wartości malejąco, a potem `Map.fromEntries`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
Map<String, int> policzSlowa(String tekst) {
  var liczniki = <String, int>{};

  // Wyodrębnij słowa (litery, także polskie) ignorując interpunkcję i wielkość liter
  var slowa = RegExp(r'[a-ząćęłńóśźż]+').allMatches(tekst.toLowerCase());
  for (var m in slowa) {
    var slowo = m.group(0)!;
    liczniki.update(slowo, (v) => v + 1, ifAbsent: () => 1);
  }

  // Posortuj pary malejąco wg liczby wystąpień
  var pary = liczniki.entries.toList()
    ..sort((a, b) => b.value.compareTo(a.value));

  return Map.fromEntries(pary);
}

void main() {
  print(policzSlowa('Kot i pies. Kot!'));
  print(policzSlowa('Ala ma kota, kota ma Ala'));
}
// Oczekiwane wyjście:
// {kot: 2, i: 1, pies: 1}
// {ala: 2, ma: 2, kota: 2}
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Zaimplementuj funkcję `dniDoWydarzenia(String dataISO)`, która przyjmuje datę w formacie ISO (`RRRR-MM-DD`) i zwraca liczbę pełnych dni od daty odniesienia `2024-01-01` (godzina 00:00) do podanej daty. Jeśli data jest wcześniejsza niż odniesienie, wynik ma być ujemny. Jeśli napis nie jest poprawną datą, funkcja ma zwrócić `null` (skorzystaj z bezpiecznego parsowania).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `'2024-01-11'` | `10` |
| `'2023-12-31'` | `-1` |
| `'nieprawidłowa'` | `null` |

### Wskazówki

1. Użyj `DateTime.tryParse`, które zwraca `null` zamiast rzucać wyjątek przy błędnym napisie
2. Zdefiniuj datę odniesienia jako `DateTime(2024, 1, 1)`
3. Różnicę policz metodą `difference`, a następnie odczytaj pole `inDays` z obiektu `Duration`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
int? dniDoWydarzenia(String dataISO) {
  // tryParse zwraca null przy niepoprawnym formacie — bezpieczniejsze niż parse
  var data = DateTime.tryParse(dataISO);
  if (data == null) return null;

  var odniesienie = DateTime(2024, 1, 1);
  // difference zwraca Duration; inDays daje liczbę pełnych dni (ze znakiem)
  return data.difference(odniesienie).inDays;
}

void main() {
  print(dniDoWydarzenia('2024-01-11')); // 10
  print(dniDoWydarzenia('2023-12-31')); // -1
  print(dniDoWydarzenia('nieprawidłowa')); // null
}
// Oczekiwane wyjście:
// 10
// -1
// null
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. `String` jest niezmienny — metody takie jak `replaceAll`, `trim` czy `toUpperCase` zawsze zwracają nowy napis. Interpolacja (`$`, `${}`), napisy wieloliniowe (`'''`) i surowe (`r'...'`) upraszczają budowanie tekstu.
2. `RegExp` implementuje `Pattern`, dlatego działa wszędzie tam, gdzie metody `String` przyjmują wzorzec; grupy przechwytujące (`group(n)`) i `replaceAllMapped` umożliwiają zaawansowane przekształcenia.
3. Typy liczbowe oferują bezpieczne parsowanie (`tryParse` zamiast `parse`), zaokrąglanie (`round`, `floor`, `ceil`, `truncate`) i porównania (`compareTo`, `clamp`).
4. `DateTime` i `Duration` współpracują: `difference` zwraca `Duration`, a `add`/`subtract` przesuwają moment w czasie; do bezpiecznego parsowania służy `DateTime.tryParse`.
5. Dobór kolekcji zależy od potrzeb: `List` dla uporządkowanego dostępu po indeksie (O(1) po indeksie), `Set` dla unikalności i szybkiego `contains` (O(1) średnio), `Map` dla odwzorowań klucz-wartość (O(1) średnio po kluczu).
6. Interfejsy `Iterable`, `Iterator` i `Comparable` to fundamenty przetwarzania sekwencji i porządkowania — implementacja `Comparable.compareTo` pozwala sortować obiekty własnych typów.

---

**Następny moduł:** [Biblioteka dart:collection](02-dart-collection.md)
**Poprzedni moduł:** [Obsługa błędów w kodzie asynchronicznym](../07-concurrency/04-error-handling-async.md)
