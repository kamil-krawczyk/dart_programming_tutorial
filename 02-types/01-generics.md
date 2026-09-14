---
id: "2.1"
title: "Typy generyczne"
difficulty: "intermediate"
section: "02-types"
prerequisites:
  - "Zmienne i typy danych"
  - "Operatory"
---

# 2.1 Typy generyczne

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Zmienne i typy danych](../01-basics/01-variables-types.md), [Operatory](../01-basics/02-operators.md)
- **Cele nauki:**
  1. Rozumieć czym są typy generyczne i dlaczego zapewniają bezpieczeństwo typów przy zachowaniu reużywalności kodu
  2. Tworzyć własne klasy i metody generyczne z wieloma parametrami typu
  3. Stosować ograniczenia typów (bounded type parameters) i rozumieć ich wpływ na API
  4. Rozumieć kowariancję w Dart i identyfikować potencjalnie niebezpieczne przypisania
  5. Definiować aliasy typów generycznych (typedef) dla czytelniejszego kodu

---

## Klasy generyczne

Generyki pozwalają na tworzenie klas i funkcji, które operują na różnych typach danych bez utraty informacji o typie. Zamiast pisać osobne klasy dla każdego typu, definiujemy klasę z parametrem typu (np. `T`), który jest konkretyzowany przy użyciu.

### Wbudowane kolekcje generyczne

Dart posiada wbudowane kolekcje generyczne — `List<T>`, `Set<T>` i `Map<K, V>` — które są najczęściej używanymi przykładami generyków.

Poniższy przykład pokazuje jak parametr typu zapewnia bezpieczeństwo typów w `List<T>`:

```dart
void main() {
  // List<T> — lista z parametrem typu T
  List<String> imiona = ['Anna', 'Jan', 'Ewa'];
  imiona.add('Tomek'); // OK — String pasuje do List<String>
  // imiona.add(42);   // Błąd kompilacji — int nie jest String

  // Dart wnioskuje typ z elementów
  var liczby = [1, 2, 3]; // wnioskowany: List<int>
  liczby.add(4); // OK
  // liczby.add('tekst'); // Błąd kompilacji — String nie jest int

  print(imiona); // [Anna, Jan, Ewa, Tomek]
  print(liczby); // [1, 2, 3, 4]
}
// Oczekiwane wyjście:
// [Anna, Jan, Ewa, Tomek]
// [1, 2, 3, 4]
```

Poniższy przykład demonstruje użycie `Set<T>` z typem generycznym:

```dart
void main() {
  // Set<T> — zbiór unikalnych elementów typu T
  Set<int> unikalneLiczby = {1, 2, 3};
  unikalneLiczby.add(4);
  unikalneLiczby.add(2); // ignorowane — już istnieje
  unikalneLiczby.add(1); // ignorowane — już istnieje

  Set<String> tagi = {'dart', 'flutter', 'programowanie'};

  print(unikalneLiczby); // {1, 2, 3, 4}
  print(tagi.contains('dart')); // true
}
// Oczekiwane wyjście:
// {1, 2, 3, 4}
// true
```

Poniższy przykład pokazuje `Map<K, V>` z dwoma parametrami typów — kluczem i wartością:

```dart
void main() {
  // Map<K, V> — mapa z parametrami typu K (klucz) i V (wartość)
  Map<String, int> wyniki = {'Anna': 95, 'Jan': 82, 'Ewa': 91};
  wyniki['Tomek'] = 88; // OK — String klucz, int wartość
  // wyniki[42] = 'test'; // Błąd kompilacji — niespójne typy

  // Map z bardziej złożonymi typami
  Map<String, List<int>> ocenyPrzedmiotow = {
    'matematyka': [5, 4, 5],
    'fizyka': [4, 3, 4],
  };

  print(wyniki);
  print(ocenyPrzedmiotow['matematyka']);
}
// Oczekiwane wyjście:
// {Anna: 95, Jan: 82, Ewa: 91, Tomek: 88}
// [5, 4, 5]
```

### Własna klasa generyczna z jednym parametrem typu

Tworzenie własnej klasy generycznej pozwala na enkapsulację logiki niezależnej od konkretnego typu.

Poniższy przykład implementuje prostą generyczną klasę `Pudelko<T>`, która przechowuje opcjonalną wartość:

```dart
/// Generyczne pudełko przechowujące opcjonalną wartość typu T
class Pudelko<T> {
  T? _zawartosc; // pole generyczne — typ znany dopiero przy użyciu

  bool get jestPuste => _zawartosc == null;

  void wloz(T element) {
    _zawartosc = element;
  }

  T? wyjmij() {
    final wynik = _zawartosc;
    _zawartosc = null; // opróżnienie pudełka
    return wynik;
  }

  @override
  String toString() => jestPuste ? 'Pudelko(puste)' : 'Pudelko($_zawartosc)';
}

void main() {
  // Konkretyzacja z typem String
  var pudelkoNapisow = Pudelko<String>();
  pudelkoNapisow.wloz('Dart');
  print(pudelkoNapisow); // Pudelko(Dart)

  // Konkretyzacja z typem int
  var pudelkoLiczb = Pudelko<int>();
  pudelkoLiczb.wloz(42);
  print(pudelkoLiczb.wyjmij()); // 42
  print(pudelkoLiczb); // Pudelko(puste)
}
// Oczekiwane wyjście:
// Pudelko(Dart)
// 42
// Pudelko(puste)
```

### Własna klasa generyczna z wieloma parametrami typu

Klasa może mieć wiele parametrów typu, co przydaje się np. przy implementacji par, wyników operacji lub mapowań.

Poniższy przykład definiuje klasę `Para<A, B>` przechowującą dwie wartości różnych typów:

```dart
/// Generyczna para — przechowuje dwie wartości potencjalnie różnych typów
class Para<A, B> {
  final A pierwszy;
  final B drugi;

  Para(this.pierwszy, this.drugi);

  /// Zamienia kolejność elementów w parze
  Para<B, A> zamien() => Para(drugi, pierwszy);

  @override
  String toString() => 'Para($pierwszy, $drugi)';
}

void main() {
  // Para<String, int> — klucz tekstowy z wartością liczbową
  var wpis = Para<String, int>('wiek', 28);
  print(wpis); // Para(wiek, 28)
  print(wpis.pierwszy.runtimeType); // String
  print(wpis.drugi.runtimeType); // int

  // Zamiana kolejności — typy się odwracają
  var odwrocony = wpis.zamien(); // Para<int, String>
  print(odwrocony); // Para(28, wiek)
}
// Oczekiwane wyjście:
// Para(wiek, 28)
// String
// int
// Para(28, wiek)
```

Poniższy przykład demonstruje klasę `Wynik<T, E>` używaną do reprezentowania sukcesu lub błędu (pattern znany z programowania funkcyjnego):

```dart
/// Generyczny typ wyniku — Either/Result pattern
/// T — typ wartości sukcesu, E — typ błędu
class Wynik<T, E> {
  final T? _wartosc;
  final E? _blad;
  final bool sukces;

  Wynik.ok(T wartosc)
      : _wartosc = wartosc,
        _blad = null,
        sukces = true;

  Wynik.blad(E blad)
      : _wartosc = null,
        _blad = blad,
        sukces = false;

  T get wartosc {
    if (!sukces) throw StateError('Brak wartości — wynik to błąd');
    return _wartosc as T;
  }

  E get blad {
    if (sukces) throw StateError('Brak błędu — wynik to sukces');
    return _blad as E;
  }

  @override
  String toString() => sukces ? 'Ok($_wartosc)' : 'Blad($_blad)';
}

void main() {
  // Wynik<int, String> — sukces zwraca int, błąd opisany Stringiem
  Wynik<int, String> wynik1 = Wynik.ok(42);
  Wynik<int, String> wynik2 = Wynik.blad('Nie znaleziono');

  print(wynik1); // Ok(42)
  print(wynik1.wartosc); // 42
  print(wynik2); // Blad(Nie znaleziono)
  print(wynik2.blad); // Nie znaleziono
}
// Oczekiwane wyjście:
// Ok(42)
// 42
// Blad(Nie znaleziono)
// Nie znaleziono
```

---

## Metody generyczne

Parametry typu można dodawać nie tylko do klas, ale również do pojedynczych metod. Pozwala to na tworzenie reużywalnych funkcji, które zachowują informację o typie.

### Funkcje generyczne (top-level)

Poniższy przykład demonstruje generyczną funkcję `pierwszyLubDomyslny<T>`, która bezpiecznie zwraca pierwszy element listy:

```dart
/// Generyczna funkcja — T jest parametrem typu metody
T pierwszyLubDomyslny<T>(List<T> lista, T domyslny) {
  return lista.isEmpty ? domyslny : lista.first;
}

void main() {
  // Dart wnioskuje T na podstawie argumentów
  var wynik1 = pierwszyLubDomyslny([10, 20, 30], 0);
  var wynik2 = pierwszyLubDomyslny(<String>[], 'brak');

  print(wynik1); // 10
  print(wynik2); // brak

  // Jawna specyfikacja typu
  var wynik3 = pierwszyLubDomyslny<double>([1.5, 2.5], 0.0);
  print(wynik3); // 1.5
}
// Oczekiwane wyjście:
// 10
// brak
// 1.5
```

Poniższy przykład pokazuje generyczną funkcję zamieniającą elementy pary:

```dart
/// Generyczna funkcja z dwoma parametrami typu
(B, A) zamienKolejnosc<A, B>(A pierwszy, B drugi) {
  return (drugi, pierwszy); // zwraca record z zamienionymi elementami
}

void main() {
  var wynik = zamienKolejnosc<String, int>('hello', 42);
  print(wynik); // (42, hello)
  print(wynik.$1.runtimeType); // int
  print(wynik.$2.runtimeType); // String
}
// Oczekiwane wyjście:
// (42, hello)
// int
// String
```

### Metody generyczne w klasie

Metody generyczne mogą też być częścią klasy (niezależnie od tego, czy sama klasa jest generyczna).

Poniższy przykład pokazuje klasę z metodą generyczną `mapuj<R>`:

```dart
class Kolekcja<T> {
  final List<T> _elementy;

  Kolekcja(this._elementy);

  /// Metoda generyczna — R jest niezależne od T klasy
  List<R> mapuj<R>(R Function(T) transformacja) {
    return _elementy.map(transformacja).toList();
  }

  /// Filtrowanie z zachowaniem typu T
  Kolekcja<T> filtruj(bool Function(T) predykat) {
    return Kolekcja(_elementy.where(predykat).toList());
  }
}

void main() {
  var liczby = Kolekcja<int>([1, 2, 3, 4, 5]);

  // mapuj<String> — konwertuje int na String (R = String)
  var napisy = liczby.mapuj<String>((n) => 'Wartość: $n');
  print(napisy); // [Wartość: 1, Wartość: 2, ...]

  // filtruj — zachowuje typ int
  var parzyste = liczby.filtruj((n) => n % 2 == 0);
  print(parzyste.mapuj((n) => n)); // [2, 4]
}
// Oczekiwane wyjście:
// [Wartość: 1, Wartość: 2, Wartość: 3, Wartość: 4, Wartość: 5]
// [2, 4]
```

---

## Ograniczenia typów (bounded type parameters)

Domyślnie parametr typu generycznego jest ograniczony do `Object?` — może być dowolnym typem. Używając słowa kluczowego `extends`, możemy zawęzić dozwolone typy do podtypów określonej klasy, co daje dostęp do metod i właściwości tej klasy.

### Poprawne użycie ograniczeń

Poniższy przykład definiuje ograniczenie `T extends Comparable<dynamic>`, co pozwala na porównywanie elementów. Używamy `Comparable<dynamic>` zamiast `Comparable<T>`, ponieważ np. `int` implementuje `Comparable<num>` (a nie `Comparable<int>`) — ograniczenie `Comparable<dynamic>` akceptuje wszystkie porównywalne typy:

```dart
/// T musi implementować Comparable — dzięki temu możemy użyć compareTo()/sort()
class PosortowanaLista<T extends Comparable<dynamic>> {
  final List<T> _elementy = [];

  void dodaj(T element) {
    _elementy.add(element);
    _elementy.sort(); // sort() wymaga Comparable — stąd constraint
  }

  List<T> get elementy => List.unmodifiable(_elementy);

  T get minimum => _elementy.first;
  T get maksimum => _elementy.last;

  @override
  String toString() => 'PosortowanaLista($_elementy)';
}

void main() {
  // int implementuje Comparable — constraint spełniony
  var liczby = PosortowanaLista<int>();
  liczby.dodaj(30);
  liczby.dodaj(10);
  liczby.dodaj(20);

  print(liczby); // PosortowanaLista([10, 20, 30])
  print('Min: ${liczby.minimum}, Max: ${liczby.maksimum}');

  // String też implementuje Comparable<String>
  var slowa = PosortowanaLista<String>();
  slowa.dodaj('banan');
  slowa.dodaj('awokado');
  slowa.dodaj('czereśnia');

  print(slowa); // PosortowanaLista([awokado, banan, czereśnia])
}
// Oczekiwane wyjście:
// PosortowanaLista([10, 20, 30])
// Min: 10, Max: 30
// PosortowanaLista([awokado, banan, czereśnia])
```

Poniższy przykład pokazuje ograniczenie do klasy bazowej z hierachią dziedziczenia:

```dart
/// Klasa bazowa definiująca interfejs kształtu
abstract class Ksztalt {
  double get pole;
  String get nazwa;
}

class Kolo extends Ksztalt {
  final double promien;
  Kolo(this.promien);

  @override
  double get pole => 3.14159 * promien * promien;

  @override
  String get nazwa => 'Koło(r=$promien)';
}

class Prostokat extends Ksztalt {
  final double szerokosc, wysokosc;
  Prostokat(this.szerokosc, this.wysokosc);

  @override
  double get pole => szerokosc * wysokosc;

  @override
  String get nazwa => 'Prostokąt(${szerokosc}x$wysokosc)';
}

/// T extends Ksztalt — możemy odwoływać się do .pole i .nazwa
T najwiekszy<T extends Ksztalt>(List<T> ksztalty) {
  return ksztalty.reduce((a, b) => a.pole >= b.pole ? a : b);
}

void main() {
  var kola = [Kolo(1.0), Kolo(3.0), Kolo(2.0)];
  var max = najwiekszy(kola); // T wnioskowany jako Kolo
  print('${max.nazwa}: pole = ${max.pole.toStringAsFixed(2)}');
}
// Oczekiwane wyjście:
// Koło(r=3.0): pole = 28.27
```

### Błąd kompilacji przy naruszeniu ograniczenia

Poniższy przykład demonstruje co się dzieje, gdy próbujemy użyć typu niespełniającego ograniczenia:

```dart
class PosortowanaLista<T extends Comparable<dynamic>> {
  final List<T> _elementy = [];
  void dodaj(T element) {
    _elementy.add(element);
    _elementy.sort();
  }
}

/// Klasa bez implementacji Comparable
class Punkt {
  final double x, y;
  Punkt(this.x, this.y);
}

void main() {
  // OK — int implementuje Comparable
  var liczby = PosortowanaLista<int>();
  liczby.dodaj(5);

  // BŁĄD KOMPILACJI:
  // 'Punkt' doesn't conform to the bound 'Comparable<dynamic>' of the type parameter 'T'
  // var punkty = PosortowanaLista<Punkt>(); // ← nie skompiluje się!
}
// Błąd kompilacji:
// 'Punkt' doesn't conform to the bound 'Comparable<dynamic>' of the type parameter 'T'.
```

---

## Kowariancja (covariance)

W Dart, typy generyczne są kowariantne — oznacza to, że `List<Cat>` jest uznawane za podtyp `List<Animal>` jeśli `Cat extends Animal`. Jest to wygodne, ale może prowadzić do błędów runtime w pewnych sytuacjach.

### Bezpieczne przypisanie kowariantne

Poniższy przykład pokazuje bezpieczne użycie kowariancji — odczytujemy elementy z listy, nie modyfikujemy jej:

```dart
class Zwierze {
  final String nazwa;
  Zwierze(this.nazwa);

  @override
  String toString() => nazwa;
}

class Kot extends Zwierze {
  Kot(super.nazwa);
  void mrucz() => print('$nazwa mruczy...');
}

class Pies extends Zwierze {
  Pies(super.nazwa);
  void szczekaj() => print('$nazwa szczeka!');
}

/// Funkcja przyjmuje List<Zwierze> — dzięki kowariancji
/// możemy przekazać List<Kot> lub List<Pies>
void wyswietlZwierzeta(List<Zwierze> zwierzeta) {
  for (var z in zwierzeta) {
    print('Zwierzę: $z'); // bezpieczne — tylko odczyt
  }
}

void main() {
  List<Kot> koty = [Kot('Mruczek'), Kot('Filemon')];

  // Kowariantne przypisanie — List<Kot> jako List<Zwierze>
  // Bezpieczne, bo tylko odczytujemy
  wyswietlZwierzeta(koty);
}
// Oczekiwane wyjście:
// Zwierzę: Mruczek
// Zwierzę: Filemon
```

### Niebezpieczne przypisanie kowariantne (runtime error)

Poniższy przykład demonstruje sytuację, w której kowariancja prowadzi do błędu runtime — próba dodania elementu nieodpowiedniego typu do listy:

```dart
class Zwierze {
  final String nazwa;
  Zwierze(this.nazwa);
}

class Kot extends Zwierze {
  Kot(super.nazwa);
}

class Pies extends Zwierze {
  Pies(super.nazwa);
}

void main() {
  List<Kot> koty = [Kot('Mruczek'), Kot('Filemon')];

  // Kowariantne przypisanie — kompilator pozwala
  List<Zwierze> zwierzeta = koty; // List<Kot> traktowane jako List<Zwierze>

  // NIEBEZPIECZNE — runtime error!
  // Rzeczywista lista to wciąż List<Kot>, ale referencja mówi List<Zwierze>
  try {
    zwierzeta.add(Pies('Burek')); // próba dodania Psa do listy Kotów!
  } on TypeError catch (e) {
    print('Błąd runtime: $e');
    // type 'Pies' is not a subtype of type 'Kot' of 'value'
  }

  print('Koty nadal bezpieczne: $koty');
}
// Oczekiwane wyjście:
// Błąd runtime: type 'Pies' is not a subtype of type 'Kot' of 'value'
// Koty nadal bezpieczne: [Instance of 'Kot', Instance of 'Kot']
```

Kluczowa zasada: kowariancja jest bezpieczna przy odczytywaniu (producent), ale niebezpieczna przy zapisywaniu (konsument). Gdy funkcja przyjmuje `List<Zwierze>` i tylko czyta z niej, przekazanie `List<Kot>` jest bezpieczne. Gdy funkcja dodaje elementy do listy, przekazanie `List<Kot>` jako `List<Zwierze>` może spowodować błąd runtime.

---

## Generyczne aliasy typów (typedef)

Słowo kluczowe `typedef` pozwala na tworzenie aliasów dla typów generycznych, co poprawia czytelność kodu, szczególnie dla złożonych sygnatur typów.

Poniższy przykład tworzy aliasy dla typów funkcji i złożonych struktur danych:

```dart
/// Alias dla funkcji porównującej dwa elementy typu T
typedef Komparator<T> = int Function(T a, T b);

/// Alias dla złożonego typu mapy
typedef Rejestr<T> = Map<String, List<T>>;

/// Alias dla callbacku transformującego wartość
typedef Transformacja<T, R> = R Function(T input);

int porownajDlugosc(String a, String b) => a.length.compareTo(b.length);

void main() {
  // Użycie typedef Komparator<String>
  Komparator<String> porownaj = porownajDlugosc;
  var slowa = ['programowanie', 'Dart', 'generyki'];
  slowa.sort(porownaj); // sortowanie po długości
  print(slowa); // [Dart, generyki, programowanie]

  // Użycie typedef Rejestr<int>
  Rejestr<int> oceny = {
    'Anna': [5, 4, 5],
    'Jan': [3, 4, 3],
  };
  print(oceny['Anna']); // [5, 4, 5]

  // Użycie typedef Transformacja<int, String>
  Transformacja<int, String> doTekstu = (n) => 'Liczba: $n';
  print(doTekstu(42)); // Liczba: 42
}
// Oczekiwane wyjście:
// [Dart, generyki, programowanie]
// [5, 4, 5]
// Liczba: 42
```

Poniższy przykład pokazuje typedef używany z klasami generycznymi, tworzący czytelny alias dla złożonej struktury:

```dart
/// Alias dla odpowiedzi API — Map z metadanymi i danymi
typedef OdpowiedzApi<T> = ({int statusCode, String wiadomosc, T? dane});

/// Alias dla walidatora — funkcja zwracająca opcjonalny komunikat błędu
typedef Walidator<T> = String? Function(T wartosc);

OdpowiedzApi<List<String>> pobierzUzytkownikow() {
  // Symulacja odpowiedzi API
  return (
    statusCode: 200,
    wiadomosc: 'OK',
    dane: ['Anna', 'Jan', 'Ewa'],
  );
}

void main() {
  // Typedef sprawia, że typ zwracany jest czytelny
  OdpowiedzApi<List<String>> odpowiedz = pobierzUzytkownikow();
  print('Status: ${odpowiedz.statusCode}');
  print('Dane: ${odpowiedz.dane}');

  // Walidator<String> — sprawdza czy string nie jest pusty
  Walidator<String> niepusty = (s) => s.isEmpty ? 'Pole wymagane' : null;
  print(niepusty('')); // Pole wymagane
  print(niepusty('tekst')); // null (brak błędu)
}
// Oczekiwane wyjście:
// Status: 200
// Dane: [Anna, Jan, Ewa]
// Pole wymagane
// null
```

---

## Ćwiczenie 1 (intermediate)

### Opis problemu

Zaimplementuj generyczną klasę `Stos<T>` (stack) obsługującą operacje: `push(T element)`, `pop()` zwracające `T?`, `peek()` zwracające `T?` (bez usuwania), oraz `get isEmpty`. Następnie użyj stosu do odwrócenia listy dowolnego typu.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| Lista: `[1, 2, 3, 4, 5]` | Odwrócona: `[5, 4, 3, 2, 1]` |
| Lista: `['a', 'b', 'c']` | Odwrócona: `['c', 'b', 'a']` |

### Wskazówki

1. Wewnętrznie użyj `List<T>` do przechowywania elementów stosu
2. `push` dodaje na koniec listy, `pop` usuwa i zwraca ostatni element
3. Napisz generyczną funkcję `odwroc<T>(List<T> lista)`, która używa `Stos<T>` do odwrócenia elementów

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Stos<T> {
  final List<T> _elementy = [];

  bool get isEmpty => _elementy.isEmpty;
  int get rozmiar => _elementy.length;

  void push(T element) {
    _elementy.add(element);
  }

  T? pop() {
    if (isEmpty) return null;
    return _elementy.removeLast();
  }

  T? peek() {
    if (isEmpty) return null;
    return _elementy.last;
  }

  @override
  String toString() => 'Stos($_elementy)';
}

/// Generyczna funkcja odwracająca listę za pomocą stosu
List<T> odwroc<T>(List<T> lista) {
  var stos = Stos<T>();
  for (var element in lista) {
    stos.push(element);
  }

  var wynik = <T>[];
  while (!stos.isEmpty) {
    wynik.add(stos.pop() as T);
  }
  return wynik;
}

void main() {
  print(odwroc([1, 2, 3, 4, 5])); // [5, 4, 3, 2, 1]
  print(odwroc(['a', 'b', 'c'])); // [c, b, a]

  var stos = Stos<int>();
  stos.push(10);
  stos.push(20);
  print('Peek: ${stos.peek()}'); // 20
  print('Pop: ${stos.pop()}'); // 20
  print('Rozmiar: ${stos.rozmiar}'); // 1
}
```

</details>

---

## Ćwiczenie 2 (advanced)

### Opis problemu

Zaimplementuj generyczną klasę `Cache<K, V>` z ograniczeniem czasu życia wpisów. Cache powinien:
- Przechowywać pary klucz-wartość z parametrami typu `K` (klucz) i `V extends Object` (wartość nie-nullable)
- Automatycznie usuwać wpisy starsze niż podany `Duration ttl` (time-to-live)
- Udostępniać metody: `put(K klucz, V wartosc)`, `get(K klucz)` zwracające `V?`, `containsKey(K klucz)`, `usunPrzeterminowane()`
- Użyć ograniczenia typu `V extends Object` aby zagwarantować, że wartości nie będą nullable

Przetestuj cache z typami `Cache<String, int>` i `Cache<int, List<String>>`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `put('a', 1)`, `put('b', 2)`, czekaj 50ms, `put('c', 3)`, TTL=40ms | `get('a')` → `null` (przeterminowany), `get('c')` → `3` (aktualny) |
| `put(1, ['x'])`, `put(2, ['y', 'z'])`, TTL=1s | `get(1)` → `['x']`, `get(2)` → `['y', 'z']`, `containsKey(3)` → `false` |

### Wskazówki

1. Przechowuj wpisy jako `Map<K, ({V wartosc, DateTime wstawiono})>` — record z wartością i czasem wstawienia
2. W metodzie `get` sprawdź, czy czas od wstawienia nie przekroczył `ttl` — jeśli tak, usuń wpis i zwróć `null`
3. `V extends Object` zapewnia, że `null` z `get` jednoznacznie oznacza brak wartości (nie mylić z wartością `null`)
4. Użyj `DateTime.now().difference(wstawiono) > ttl` do sprawdzenia ważności

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
/// Generyczny cache z TTL (time-to-live)
/// V extends Object — wartości muszą być non-nullable
class Cache<K, V extends Object> {
  final Duration ttl;
  final Map<K, ({V wartosc, DateTime wstawiono})> _wpisy = {};

  Cache({required this.ttl});

  void put(K klucz, V wartosc) {
    _wpisy[klucz] = (wartosc: wartosc, wstawiono: DateTime.now());
  }

  V? get(K klucz) {
    final wpis = _wpisy[klucz];
    if (wpis == null) return null;

    // Sprawdzenie czy wpis nie jest przeterminowany
    if (DateTime.now().difference(wpis.wstawiono) > ttl) {
      _wpisy.remove(klucz); // usunięcie przeterminowanego wpisu
      return null;
    }
    return wpis.wartosc;
  }

  bool containsKey(K klucz) => get(klucz) != null;

  void usunPrzeterminowane() {
    final teraz = DateTime.now();
    _wpisy.removeWhere((_, wpis) => teraz.difference(wpis.wstawiono) > ttl);
  }

  int get rozmiar => _wpisy.length;
}

Future<void> main() async {
  // Cache<String, int> z TTL 100ms
  var cache = Cache<String, int>(ttl: Duration(milliseconds: 100));
  cache.put('a', 1);
  cache.put('b', 2);

  print('Przed TTL: a=${cache.get("a")}, b=${cache.get("b")}');

  // Czekamy aż wpisy się przeterminują
  await Future.delayed(Duration(milliseconds: 150));

  print('Po TTL: a=${cache.get("a")}, b=${cache.get("b")}');

  // Nowy wpis po oczekiwaniu — wciąż aktualny
  cache.put('c', 3);
  print('Nowy wpis: c=${cache.get("c")}');

  // Cache z listami
  var cache2 = Cache<int, List<String>>(ttl: Duration(seconds: 1));
  cache2.put(1, ['x']);
  cache2.put(2, ['y', 'z']);
  print('Lista: ${cache2.get(1)}, ${cache2.get(2)}');
  print('Zawiera 3: ${cache2.containsKey(3)}');
}
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. Generyki zapewniają bezpieczeństwo typów bez rezygnacji z reużywalności — zamiast używać `dynamic`, parametryzujemy typy
2. Klasy generyczne mogą mieć wiele parametrów typu (np. `Map<K, V>`, `Wynik<T, E>`) — każdy parametr reprezentuje inny aspekt typu
3. Ograniczenia typów (`T extends X`) dają dostęp do metod klasy bazowej i zapobiegają użyciu niekompatybilnych typów na etapie kompilacji
4. Kowariancja w Dart jest wygodna, ale niebezpieczna przy modyfikacji kolekcji — czytanie z `List<Podtyp>` traktowanej jako `List<Nadtyp>` jest bezpieczne, ale dodawanie elementów może powodować błędy runtime
5. Aliasy typów (`typedef`) poprawiają czytelność złożonych sygnatur typów i mogą być generyczne
6. Metody generyczne pozwalają na parametryzację typu niezależnie od klasy, w której się znajdują

---

**Następny moduł:** [If/else i pętle](../03-control-flow/01-conditionals-loops.md)
**Poprzedni moduł:** [Null safety](../01-basics/03-null-safety.md)
