---
id: "8.2"
title: "Biblioteka standardowa — dart:collection"
difficulty: "intermediate"
section: "08-standard-library"
prerequisites:
  - "Biblioteka standardowa — dart:core"
  - "Klasy generyczne"
---

# 8.2 Biblioteka standardowa — dart:collection

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Biblioteka standardowa — dart:core](01-dart-core.md), [Klasy generyczne](../06-generics/01-generic-classes.md)
- **Cele nauki:**
  1. Poznać zaawansowane struktury danych z biblioteki `dart:collection`: `LinkedList`, `Queue`, `HashMap`, `LinkedHashMap`, `SplayTreeMap`, `HashSet`, `LinkedHashSet`, `SplayTreeSet` oraz `UnmodifiableListView`
  2. Rozumieć złożoność czasową (wstawianie, wyszukiwanie, usuwanie, iteracja) każdej kolekcji i konsekwencje wyboru implementacji
  3. Rozróżniać gwarancje uporządkowania (brak porządku, kolejność wstawiania, porządek sortowania) oraz właściwości unikalności i model klucz–wartość
  4. Dobierać optymalną kolekcję do konkretnego wzorca dostępu (częste wyszukiwania, uporządkowana iteracja, deduplikacja, przetwarzanie FIFO) i uzasadniać ten wybór

---

## Wprowadzenie

Biblioteka `dart:core` udostępnia trzy „domyślne" kolekcje: `List`, `Set` i `Map`. Za tymi interfejsami stoją konkretne implementacje z biblioteki **`dart:collection`**, a oprócz nich biblioteka ta dostarcza dodatkowe, wyspecjalizowane struktury danych. Wybór właściwej implementacji ma bezpośredni wpływ na wydajność i zachowanie programu: inaczej zachowuje się kolekcja oparta na tablicy haszującej (szybkie wyszukiwanie, brak porządku), a inaczej kolekcja oparta na drzewie zrównoważonym (elementy zawsze posortowane, ale wolniejsze operacje).

Aby korzystać z tych typów, dołącz bibliotekę:

```dart
// Import biblioteki z zaawansowanymi strukturami danych
import 'dart:collection';
```

W całym module złożoność czasową podajemy w notacji dużego O, gdzie `n` oznacza liczbę elementów w kolekcji. Zapis **O(1) amortyzowane** oznacza, że pojedyncza operacja jest zwykle stała, choć sporadycznie (np. przy powiększaniu tablicy haszującej) może być kosztowniejsza — średnio jednak pozostaje stała.

---

## `Queue` — kolejka dwustronna (FIFO/LIFO)

`Queue` to kolejka pozwalająca wydajnie dodawać i usuwać elementy z **obu końców**. W przeciwieństwie do `List`, usunięcie z początku (`removeFirst`) jest tanie — nie wymaga przesuwania pozostałych elementów. Domyślna implementacja `ListQueue` opiera się na buforze cyklicznym.

- **Wstawianie** (`add`/`addFirst`/`addLast`): O(1) amortyzowane
- **Wyszukiwanie** (dostęp do dowolnego elementu, `contains`): O(n)
- **Usuwanie** (`removeFirst`/`removeLast`): O(1); usuwanie po wartości: O(n)
- **Iteracja**: O(n)

Wyróżniającą cechą `Queue` jest tani dostęp do obu końców, co czyni ją idealną do przetwarzania FIFO (kolejka zadań) oraz do implementacji stosu.

```dart
import 'dart:collection';

void main() {
  // Kolejka FIFO — zadania przetwarzane w kolejności zgłoszenia
  final kolejka = Queue<String>();
  kolejka.addLast('zadanie A'); // dodaj na koniec
  kolejka.addLast('zadanie B');
  kolejka.addFirst('zadanie pilne'); // dodaj na początek — O(1)

  // removeFirst jest tanie (O(1)) — inaczej niż List.removeAt(0)
  while (kolejka.isNotEmpty) {
    print('Przetwarzam: ${kolejka.removeFirst()}');
  }
}
// Oczekiwane wyjście:
// Przetwarzam: zadanie pilne
// Przetwarzam: zadanie A
// Przetwarzam: zadanie B
```

---

## `LinkedList` — lista dwukierunkowa z elementami zawierającymi wskaźniki

`LinkedList` w Dart jest nietypowa: nie przechowuje dowolnych wartości, lecz obiekty dziedziczące po `LinkedListEntry`. Każdy element „wie", w jakiej liście się znajduje oraz kto jest jego poprzednikiem i następnikiem. Dzięki temu usunięcie elementu ze środka listy (`entry.unlink()`) jest operacją O(1) — nie trzeba wyszukiwać jego pozycji.

- **Wstawianie** (`add`/`addFirst`/`insertBefore`/`insertAfter`): O(1)
- **Wyszukiwanie** (znalezienie elementu po wartości): O(n)
- **Usuwanie** (`unlink` znanego elementu): O(1)
- **Iteracja**: O(n)

Cechą wyróżniającą jest usuwanie i przepinanie elementów w czasie stałym, o ile mamy referencję do samego elementu.

```dart
import 'dart:collection';

// Element LinkedList musi dziedziczyć po LinkedListEntry
final class Zadanie extends LinkedListEntry<Zadanie> {
  final String nazwa;
  Zadanie(this.nazwa);
  @override
  String toString() => nazwa;
}

void main() {
  final lista = LinkedList<Zadanie>();
  final a = Zadanie('A');
  final b = Zadanie('B');
  final c = Zadanie('C');
  lista.addAll([a, b, c]);

  // Usunięcie elementu ze środka w czasie O(1) — mamy referencję do 'b'
  b.unlink();

  print(lista.map((z) => z.nazwa).toList());
  // Każdy element zna swojego następnika:
  print('Po A następuje: ${a.next}');
}
// Oczekiwane wyjście:
// [A, C]
// Po A następuje: C
```

---

## Mapy: `HashMap`, `LinkedHashMap`, `SplayTreeMap`

Mapy przechowują pary **klucz–wartość** o unikalnych kluczach. Trzy implementacje różnią się przede wszystkim gwarancją uporządkowania kluczy oraz złożonością operacji.

### `HashMap` — brak gwarancji porządku, najszybsze operacje

`HashMap` opiera się na tablicy haszującej. Kolejność iteracji jest **nieokreślona** i może się zmieniać. W zamian oferuje najszybszy dostęp.

- **Wstawianie** (`[]=`): O(1) amortyzowane
- **Wyszukiwanie** (`[]`, `containsKey`): O(1) amortyzowane
- **Usuwanie** (`remove`): O(1) amortyzowane
- **Iteracja**: O(n), kolejność nieokreślona

```dart
import 'dart:collection';

void main() {
  // HashMap — najszybsze wyszukiwanie, ale kolejność nieprzewidywalna
  final oceny = HashMap<String, int>();
  oceny['matematyka'] = 5;
  oceny['fizyka'] = 4;
  oceny['chemia'] = 3;

  // Wyszukiwanie po kluczu w czasie O(1)
  print('Ocena z fizyki: ${oceny['fizyka']}');
  print('Zawiera chemię? ${oceny.containsKey('chemia')}');
  // Kolejność iteracji nie jest gwarantowana — nie polegaj na niej
}
// Oczekiwane wyjście:
// Ocena z fizyki: 4
// Zawiera chemię? true
```

### `LinkedHashMap` — zachowuje kolejność wstawiania

`LinkedHashMap` łączy tablicę haszującą z listą wiążącą wpisy w kolejności ich dodania. Iteracja zawsze przebiega w **kolejności wstawiania**. To domyślna implementacja zwracana przez literał mapy `{}` oraz konstruktor `Map()`.

- **Wstawianie**: O(1) amortyzowane
- **Wyszukiwanie**: O(1) amortyzowane
- **Usuwanie**: O(1) amortyzowane
- **Iteracja**: O(n), w kolejności wstawiania

```dart
import 'dart:collection';

void main() {
  // LinkedHashMap — iteracja zawsze w kolejności wstawiania
  final koszyk = LinkedHashMap<String, int>();
  koszyk['chleb'] = 2;
  koszyk['mleko'] = 1;
  koszyk['jajka'] = 12;

  // Klucze pojawią się dokładnie w kolejności dodawania
  for (final wpis in koszyk.entries) {
    print('${wpis.key}: ${wpis.value}');
  }
}
// Oczekiwane wyjście:
// chleb: 2
// mleko: 1
// jajka: 12
```

### `SplayTreeMap` — klucze zawsze posortowane

`SplayTreeMap` opiera się na drzewie splay (samoorganizującym się drzewie BST). Klucze są utrzymywane w **porządku sortowania**, dzięki czemu iteracja zawsze przebiega rosnąco. Wymaga, aby klucze były porównywalne (`Comparable`) lub aby dostarczyć komparator.

- **Wstawianie**: O(log n)
- **Wyszukiwanie**: O(log n)
- **Usuwanie**: O(log n)
- **Iteracja**: O(n), w porządku posortowanym

Cechą wyróżniającą jest utrzymywanie kluczy w porządku oraz dodatkowe metody nawigacyjne, np. `firstKey`, `lastKey`, `firstKeyAfter`.

```dart
import 'dart:collection';

void main() {
  // SplayTreeMap — klucze automatycznie posortowane rosnąco
  final ranking = SplayTreeMap<int, String>();
  ranking[3] = 'brąz';
  ranking[1] = 'złoto';
  ranking[2] = 'srebro';

  // Iteracja zawsze w porządku rosnącym kluczy, niezależnie od kolejności dodania
  print(ranking.keys.toList());
  // Metody nawigacyjne dostępne dzięki uporządkowaniu
  print('Najmniejszy klucz: ${ranking.firstKey()}');
  print('Klucz po 1: ${ranking.firstKeyAfter(1)}');
}
// Oczekiwane wyjście:
// [1, 2, 3]
// Najmniejszy klucz: 1
// Klucz po 1: 2
```

---

## Zbiory: `HashSet`, `LinkedHashSet`, `SplayTreeSet`

Zbiory przechowują **unikalne** wartości (brak duplikatów). Trzy implementacje odpowiadają dokładnie trzem implementacjom map — różnią się gwarancją porządku i złożonością.

### `HashSet` — brak porządku, najszybsza deduplikacja

`HashSet` to najszybszy zbiór, ale bez gwarancji kolejności iteracji.

- **Wstawianie** (`add`): O(1) amortyzowane
- **Wyszukiwanie** (`contains`): O(1) amortyzowane
- **Usuwanie** (`remove`): O(1) amortyzowane
- **Iteracja**: O(n), kolejność nieokreślona

```dart
import 'dart:collection';

void main() {
  // HashSet — błyskawiczna deduplikacja i sprawdzanie przynależności
  final odwiedzone = HashSet<int>();
  for (final id in [1, 2, 2, 3, 1, 4]) {
    odwiedzone.add(id); // duplikaty są automatycznie ignorowane
  }

  print('Liczba unikalnych: ${odwiedzone.length}');
  print('Czy odwiedzono 3? ${odwiedzone.contains(3)}'); // O(1)
}
// Oczekiwane wyjście:
// Liczba unikalnych: 4
// Czy odwiedzono 3? true
```

### `LinkedHashSet` — unikalność + kolejność wstawiania

`LinkedHashSet` zachowuje kolejność, w jakiej elementy zostały dodane. To domyślna implementacja literału zbioru `{}` oraz konstruktora `Set()`.

- **Wstawianie**: O(1) amortyzowane
- **Wyszukiwanie**: O(1) amortyzowane
- **Usuwanie**: O(1) amortyzowane
- **Iteracja**: O(n), w kolejności wstawiania

```dart
import 'dart:collection';

void main() {
  // LinkedHashSet — usuwa duplikaty, ale zachowuje pierwotną kolejność
  final unikalneSlowa = LinkedHashSet<String>();
  for (final s in ['kot', 'pies', 'kot', 'ryba', 'pies']) {
    unikalneSlowa.add(s);
  }

  // Kolejność odpowiada pierwszemu wystąpieniu każdego elementu
  print(unikalneSlowa.toList());
}
// Oczekiwane wyjście:
// [kot, pies, ryba]
```

### `SplayTreeSet` — elementy zawsze posortowane

`SplayTreeSet` przechowuje unikalne elementy w porządku sortowania. Wymaga elementów `Comparable` lub komparatora.

- **Wstawianie**: O(log n)
- **Wyszukiwanie**: O(log n)
- **Usuwanie**: O(log n)
- **Iteracja**: O(n), w porządku posortowanym

```dart
import 'dart:collection';

void main() {
  // SplayTreeSet — unikalne elementy utrzymywane w porządku rosnącym
  final liczby = SplayTreeSet<int>();
  liczby.addAll([5, 1, 3, 1, 4, 3]); // duplikaty ignorowane

  // Iteracja zawsze posortowana, niezależnie od kolejności dodania
  print(liczby.toList());
  // Metody korzystające z uporządkowania
  print('Najmniejszy: ${liczby.first}, największy: ${liczby.last}');
}
// Oczekiwane wyjście:
// [1, 3, 4, 5]
// Najmniejszy: 1, największy: 5
```

---

## `UnmodifiableListView` — tylko-do-odczytu widok listy

`UnmodifiableListView` opakowuje istniejącą listę i udostępnia ją w trybie **tylko do odczytu**. Każda próba modyfikacji (`add`, `[]=`, `remove`) zgłasza `UnsupportedError`. Jest to „widok" — nie kopiuje danych, więc zajmuje stałą ilość dodatkowej pamięci, a odczyt ma taką samą złożoność jak w opakowanej liście.

- **Wstawianie**: niedozwolone (`UnsupportedError`)
- **Wyszukiwanie** (dostęp po indeksie): O(1)
- **Usuwanie**: niedozwolone (`UnsupportedError`)
- **Iteracja**: O(n), w kolejności listy bazowej

Cechą wyróżniającą jest ochrona wewnętrznej listy przed modyfikacją z zewnątrz przy zerowym koszcie kopiowania. Uwaga: skoro to widok, zmiany w liście bazowej są widoczne przez widok.

```dart
import 'dart:collection';

void main() {
  final bazowa = [10, 20, 30];
  // Widok tylko do odczytu — opakowuje listę bez jej kopiowania
  final widok = UnmodifiableListView<int>(bazowa);

  print(widok[1]);        // odczyt dozwolony — O(1)
  print(widok.length);

  try {
    widok.add(40); // próba modyfikacji widoku
  } on UnsupportedError {
    print('Nie można modyfikować widoku tylko do odczytu');
  }

  // Zmiana w liście bazowej jest widoczna przez widok
  bazowa.add(40);
  print(widok.length);
}
// Oczekiwane wyjście:
// 20
// 3
// Nie można modyfikować widoku tylko do odczytu
// 4
```

---

## Tabela porównawcza kolekcji

Poniższa tabela zestawia wszystkie omawiane kolekcje pod kątem gwarancji uporządkowania, unikalności, modelu klucz–wartość oraz złożoności czasowej operacji podstawowych. Pozwala szybko dobrać właściwą strukturę do problemu.

| Kolekcja | Uporządkowanie | Unikalność | Klucz–wartość | Wstawianie | Wyszukiwanie | Usuwanie | Iteracja |
|----------|----------------|------------|---------------|------------|--------------|----------|----------|
| `LinkedList` | kolejność wstawiania | nie | nie (wartości) | O(1) | O(n) | O(1)* | O(n) |
| `Queue` | kolejność wstawiania | nie | nie (wartości) | O(1) amort. | O(n) | O(1) końce | O(n) |
| `HashMap` | brak | klucze | tak | O(1) amort. | O(1) amort. | O(1) amort. | O(n) |
| `LinkedHashMap` | kolejność wstawiania | klucze | tak | O(1) amort. | O(1) amort. | O(1) amort. | O(n) |
| `SplayTreeMap` | posortowane (klucze) | klucze | tak | O(log n) | O(log n) | O(log n) | O(n) |
| `HashSet` | brak | wartości | nie | O(1) amort. | O(1) amort. | O(1) amort. | O(n) |
| `LinkedHashSet` | kolejność wstawiania | wartości | nie | O(1) amort. | O(1) amort. | O(1) amort. | O(n) |
| `SplayTreeSet` | posortowane | wartości | nie | O(log n) | O(log n) | O(log n) | O(n) |
| `UnmodifiableListView` | kolejność listy bazowej | nie | nie (wartości) | — (brak) | O(1) po indeksie | — (brak) | O(n) |

\* `LinkedList`: usunięcie w O(1) dotyczy elementu, do którego mamy referencję (`unlink`); samo odnalezienie elementu po wartości to O(n).

Wskazówki wyboru:
- Potrzebujesz **najszybszego wyszukiwania** i nie zależy Ci na kolejności → `HashMap` / `HashSet`.
- Potrzebujesz **zachować kolejność wstawiania** → `LinkedHashMap` / `LinkedHashSet`.
- Potrzebujesz **elementów zawsze posortowanych** → `SplayTreeMap` / `SplayTreeSet`.
- Potrzebujesz **przetwarzania FIFO** lub taniego dostępu do obu końców → `Queue`.
- Potrzebujesz **taniego usuwania ze środka** przy posiadaniu referencji do elementu → `LinkedList`.
- Chcesz **udostępnić listę bez ryzyka modyfikacji** → `UnmodifiableListView`.

---

## Ćwiczenie 1: Dobór kolekcji do wzorca dostępu (intermediate)

### Opis problemu

Dla każdego z poniższych scenariuszy dobierz **jedną** najbardziej odpowiednią kolekcję z `dart:collection` i **uzasadnij** wybór jednym–dwoma zdaniami odnoszącymi się do uporządkowania, unikalności i złożoności operacji. Następnie zaimplementuj funkcję demonstrującą scenariusz A.

- **Scenariusz A (częste wyszukiwania):** budujesz cache identyfikatorów już przetworzonych rekordów; jedyne operacje to `dodaj(id)` oraz `czyPrzetworzony(id)`, wykonywane miliony razy. Kolejność nie ma znaczenia.
- **Scenariusz B (uporządkowana iteracja):** przechowujesz wyniki uczniów pod kluczem będącym liczbą punktów i musisz często wypisywać je od najwyższego do najniższego wyniku.
- **Scenariusz C (FIFO):** implementujesz kolejkę wydruków — dokumenty są drukowane w kolejności zgłoszenia, a zadania pobierasz z początku.

Zaimplementuj `Set<int> unikalneId(List<int> wejscie)` dla scenariusza A, wybierając kolekcję o wyszukiwaniu O(1).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| Scenariusz A: `unikalneId([1, 2, 2, 3]).contains(2)` | `true` |
| Scenariusz A: `unikalneId([1, 2, 2, 3]).length` | `3` |

### Wskazówki

1. Dla scenariusza A liczy się wyłącznie szybkie sprawdzanie przynależności — porządek nie jest potrzebny, więc rozważ strukturę haszującą.
2. Dla scenariusza B kluczem jest liczba punktów, a wynik ma być posortowany — to sugeruje mapę opartą na drzewie.
3. Dla scenariusza C liczy się tanie pobieranie z początku — `List.removeAt(0)` to O(n), więc poszukaj czegoś lepszego.

<details>
<summary>Rozwiązanie referencyjne</summary>

Uzasadnienia:
- **Scenariusz A → `HashSet<int>`**: potrzebujemy tylko unikalności i sprawdzania przynależności; brak wymogu porządku, a `HashSet` daje `contains`/`add` w O(1) amortyzowanym — najszybszy wybór.
- **Scenariusz B → `SplayTreeMap<int, ...>`**: klucze (punkty) muszą być stale posortowane, aby iteracja przebiegała w porządku; `SplayTreeMap` utrzymuje klucze posortowane i pozwala łatwo iterować malejąco (`keys.toList().reversed`).
- **Scenariusz C → `Queue`**: przetwarzanie FIFO z tanim `removeFirst` w O(1); `List.removeAt(0)` byłby O(n).

```dart
import 'dart:collection';

// Scenariusz A — HashSet zapewnia contains/add w O(1) amortyzowanym
Set<int> unikalneId(List<int> wejscie) {
  final zbior = HashSet<int>();
  zbior.addAll(wejscie); // duplikaty automatycznie pomijane
  return zbior;
}

void main() {
  final wynik = unikalneId([1, 2, 2, 3]);
  print(wynik.contains(2)); // true
  print(wynik.length);      // 3

  // Demonstracja scenariusza B — posortowane klucze
  final rankingi = SplayTreeMap<int, String>();
  rankingi[42] = 'Ala';
  rankingi[95] = 'Bartek';
  rankingi[73] = 'Cela';
  // Malejąco: od najwyższego wyniku
  print(rankingi.keys.toList().reversed.toList()); // [95, 73, 42]

  // Demonstracja scenariusza C — FIFO
  final wydruki = Queue<String>();
  wydruki.addLast('dok1');
  wydruki.addLast('dok2');
  print(wydruki.removeFirst()); // dok1 (O(1))
}
// Oczekiwane wyjście:
// true
// 3
// [95, 73, 42]
// dok1
```

</details>

---

## Ćwiczenie 2: Deduplikacja z zachowaniem kolejności (intermediate)

### Opis problemu

Napisz funkcję `List<T> bezDuplikatow<T>(List<T> wejscie)`, która usuwa duplikaty z listy, ale **zachowuje kolejność pierwszego wystąpienia** każdego elementu. Wybierz kolekcję, która jednocześnie gwarantuje unikalność i kolejność wstawiania, i uzasadnij w komentarzu, dlaczego nie użyłeś ani `HashSet` (brak porządku), ani `SplayTreeSet` (zmienia kolejność na posortowaną).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `bezDuplikatow([3, 1, 3, 2, 1])` | `[3, 1, 2]` |
| `bezDuplikatow(['b', 'a', 'b', 'c'])` | `[b, a, c]` |

### Wskazówki

1. Potrzebujesz struktury, która odrzuca duplikaty (unikalność) i jednocześnie pamięta kolejność dodawania.
2. `HashSet` odrzuci duplikaty, ale kolejność iteracji będzie nieokreślona — to nie spełnia wymagania.
3. `SplayTreeSet` posortuje elementy, więc również nie zachowa oryginalnej kolejności.

<details>
<summary>Rozwiązanie referencyjne</summary>

Uzasadnienie: `LinkedHashSet` gwarantuje unikalność (odrzuca duplikaty) i zachowuje kolejność wstawiania, dlatego jest jedyną z trzech implementacji zbioru spełniającą oba wymagania jednocześnie. Operacje `add`/`contains` pozostają O(1) amortyzowane.

```dart
import 'dart:collection';

// LinkedHashSet: unikalność + kolejność wstawiania w O(1) amortyzowanym.
// HashSet nie daje porządku, SplayTreeSet zmienia kolejność na posortowaną.
List<T> bezDuplikatow<T>(List<T> wejscie) {
  final zbior = LinkedHashSet<T>();
  zbior.addAll(wejscie); // duplikaty pomijane, kolejność zachowana
  return zbior.toList();
}

void main() {
  print(bezDuplikatow([3, 1, 3, 2, 1]));      // [3, 1, 2]
  print(bezDuplikatow(['b', 'a', 'b', 'c'])); // [b, a, c]
}
// Oczekiwane wyjście:
// [3, 1, 2]
// [b, a, c]
```

</details>

---

## Ćwiczenie 3: Uporządkowany rejestr zdarzeń (advanced)

### Opis problemu

Zaimplementuj klasę `RejestrZdarzen`, która przechowuje zdarzenia pod kluczem będącym znacznikiem czasu (liczba całkowita) i pozwala:

1. `void dodaj(int czas, String opis)` — zapisuje zdarzenie.
2. `List<String> odNajstarszego()` — zwraca opisy zdarzeń w porządku rosnącym względem czasu.
3. `int? poprzedniCzas(int czas)` — zwraca największy znacznik czasu mniejszy od podanego (lub `null`).

Wybierz kolekcję utrzymującą klucze w porządku posortowanym i uzasadnij, dlaczego zapewnia ona operacje `dodaj`/wyszukiwanie w O(log n) oraz iterację w kolejności rosnącej bez potrzeby sortowania.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `dodaj(30,'c')`, `dodaj(10,'a')`, `dodaj(20,'b')`, `odNajstarszego()` | `[a, b, c]` |
| po powyższym: `poprzedniCzas(20)` | `10` |

### Wskazówki

1. `SplayTreeMap` utrzymuje klucze posortowane — iteracja `values` jest od razu w porządku rosnącym kluczy, bez wywoływania `sort`.
2. Do znalezienia poprzedniego znacznika czasu użyj metody `lastKeyBefore`.
3. Wstawianie i wyszukiwanie w drzewie splay mają złożoność O(log n).

<details>
<summary>Rozwiązanie referencyjne</summary>

Uzasadnienie: `SplayTreeMap<int, String>` przechowuje klucze w drzewie zrównoważonym, więc wstawianie i wyszukiwanie to O(log n), a iteracja `values`/`keys` jest natychmiast posortowana rosnąco — nie trzeba osobno sortować. Metoda `lastKeyBefore` wykorzystuje uporządkowanie do znalezienia poprzedniego klucza w czasie O(log n).

```dart
import 'dart:collection';

class RejestrZdarzen {
  // Klucze (czas) automatycznie posortowane — iteracja bez sortowania
  final SplayTreeMap<int, String> _zdarzenia = SplayTreeMap<int, String>();

  void dodaj(int czas, String opis) {
    _zdarzenia[czas] = opis; // wstawianie O(log n)
  }

  List<String> odNajstarszego() {
    // values są zwracane w porządku rosnącym kluczy
    return _zdarzenia.values.toList();
  }

  int? poprzedniCzas(int czas) {
    // Największy klucz mniejszy od 'czas' — wykorzystuje uporządkowanie
    return _zdarzenia.lastKeyBefore(czas);
  }
}

void main() {
  final rejestr = RejestrZdarzen();
  rejestr.dodaj(30, 'c');
  rejestr.dodaj(10, 'a');
  rejestr.dodaj(20, 'b');

  print(rejestr.odNajstarszego()); // [a, b, c]
  print(rejestr.poprzedniCzas(20)); // 10
  print(rejestr.poprzedniCzas(10)); // null (brak wcześniejszego)
}
// Oczekiwane wyjście:
// [a, b, c]
// 10
// null
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`dart:collection`** dostarcza konkretne implementacje kolekcji różniące się gwarancją porządku i złożonością: wybór implementacji bezpośrednio wpływa na wydajność.
2. **Struktury haszujące** (`HashMap`, `HashSet`) dają operacje O(1) amortyzowane, lecz **bez gwarancji kolejności** iteracji — idealne do częstych wyszukiwań i deduplikacji.
3. **Warianty `LinkedHash…`** (`LinkedHashMap`, `LinkedHashSet`) zachowują **kolejność wstawiania** przy zachowaniu O(1) — to domyślne implementacje literałów `{}`.
4. **Struktury drzewiaste** (`SplayTreeMap`, `SplayTreeSet`) utrzymują elementy **posortowane** kosztem operacji O(log n) i udostępniają metody nawigacyjne (`firstKey`, `lastKeyBefore`).
5. **`Queue`** zapewnia tanie (O(1)) dodawanie i usuwanie z obu końców — właściwy wybór dla FIFO/LIFO, w przeciwieństwie do kosztownego `List.removeAt(0)`.
6. **`LinkedList`** pozwala usuwać i przepinać elementy w O(1), o ile mamy do nich referencję; wymaga elementów dziedziczących po `LinkedListEntry`.
7. **`UnmodifiableListView`** udostępnia listę tylko do odczytu bez kopiowania danych — chroni wewnętrzny stan, lecz odzwierciedla zmiany listy bazowej.

---

**Poprzedni moduł:** [Biblioteka standardowa — dart:core](01-dart-core.md)
**Następny moduł:** [Biblioteka standardowa — dart:async](03-dart-async.md)
