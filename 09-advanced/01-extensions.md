---
id: "9.1"
title: "Rozszerzenia — extension methods i extension types"
difficulty: "advanced"
section: "09-advanced"
prerequisites:
  - "Klasy i konstruktory"
  - "Klasy generyczne"
  - "Null safety"
---

# 9.1 Rozszerzenia — extension methods i extension types

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Klasy generyczne](../06-generics/01-generic-classes.md), [Null safety](../01-basics/03-null-safety.md)
- **Cele nauki:**
  1. Definiować **metody rozszerzające** (extension methods), które dodają nowe funkcje do istniejących typów bez ich modyfikowania ani dziedziczenia
  2. Pisać rozszerzenia na typach wbudowanych (`String`, `int`, `List`) dostarczające reużywalne metody narzędziowe
  3. Definiować **typy rozszerzające** (extension types) z Dart 3 — cienkie, zerokosztowe opakowania na istniejących typach
  4. Rozumieć różnicę między zachowaniem w czasie kompilacji (extension methods, extension types) a rzeczywistą reprezentacją w czasie wykonania
  5. Wybierać właściwy mechanizm — extension method czy extension type — w zależności od problemu

---

## Czym są rozszerzenia?

Dart udostępnia dwa pokrewne, ale różne mechanizmy pozwalające „dokleić" nowe zachowanie do istniejących typów bez ich modyfikowania:

- **Extension methods** (metody rozszerzające) — pozwalają dodać metody, gettery, settery i operatory do dowolnego istniejącego typu (nawet takiego, którego kodu nie kontrolujesz, jak `String` czy `int`). Rozszerzenie **nie tworzy nowego typu** — to wyłącznie wygodny zapis, który kompilator zamienia na zwykłe wywołanie funkcji statycznej.
- **Extension types** (typy rozszerzające, Dart 3) — tworzą **nowy typ statyczny** będący cienkim opakowaniem na istniejącym typie (typie reprezentacji). W czasie wykonania nie istnieje żaden dodatkowy obiekt — wartość jest po prostu tą opakowaną wartością. To narzędzie do budowania **abstrakcji zerokosztowych** (zero-cost abstractions).

Oba mechanizmy działają głównie w czasie kompilacji, ale w zupełnie różny sposób: extension method dokleja funkcje do istniejącego typu, natomiast extension type wprowadza nowy typ widoczny dla systemu typów. Poznamy oba, a na końcu porównamy je i wskażemy, kiedy sięgać po który.

---

## Extension methods — podstawy

Metodę rozszerzającą definiujemy za pomocą słowa kluczowego `extension`, wskazując po `on`, jaki typ rozszerzamy. Wewnątrz ciała słowo `this` odnosi się do rozszerzanej wartości.

Poniższy przykład dodaje do typu `int` metodę i getter — mimo że nie mamy dostępu do kodu klasy `int`.

```dart
// Nazwana ekstensja IntExtensions rozszerza typ int
extension IntExtensions on int {
  // Getter: czy liczba jest parzysta (this odnosi się do wartości int)
  bool get czyParzysta => this % 2 == 0;

  // Metoda z argumentem: podnosi liczbę do potęgi wykładnika
  int doPotegi(int wykladnik) {
    var wynik = 1;
    for (var i = 0; i < wykladnik; i++) {
      wynik *= this;
    }
    return wynik;
  }
}

void main() {
  // Wywołujemy metody rozszerzające jak zwykłe metody int
  print(4.czyParzysta); // true
  print(7.czyParzysta); // false
  print(2.doPotegi(10)); // 1024
}
// Oczekiwane wyjście:
// true
// false
// 1024
```

Rozszerzenie może być **nienazwane**, ale nazwanie go (`extension IntExtensions on int`) pozwala kontrolować konflikty i jawnie je importować lub ukrywać w innych plikach.

---

## Przykład: rozszerzenie na String

Rozszerzenia najczęściej wykorzystuje się do dodania reużywalnych metod narzędziowych do typów wbudowanych. Poniższy przykład wzbogaca `String` o operacje często potrzebne przy walidacji i formatowaniu tekstu.

```dart
// Rozszerzenie dodające metody narzędziowe do typu String
extension StringUtils on String {
  // Getter: czy tekst jest pusty lub składa się tylko z białych znaków
  bool get czyPustyLubBialy => trim().isEmpty;

  // Zamienia pierwszą literę na wielką (pozostałe bez zmian)
  String zWielkiejLitery() {
    if (isEmpty) return this;
    // substring(0, 1) to pierwszy znak, substring(1) to reszta
    return '${this[0].toUpperCase()}${substring(1)}';
  }

  // Odwraca kolejność znaków w tekście
  String odwroc() => split('').reversed.join();
}

void main() {
  print('   '.czyPustyLubBialy); // true (same spacje)
  print('dart'.czyPustyLubBialy); // false

  print('dart'.zWielkiejLitery()); // Dart
  print('kajak'.odwroc()); // kajak (palindrom)
  print('abc'.odwroc()); // cba
}
// Oczekiwane wyjście:
// true
// false
// Dart
// kajak
// cba
```

---

## Przykład: rozszerzenie generyczne na List

Rozszerzenia mogą być **generyczne** — parametr typu deklarujemy przy słowie `extension`, dzięki czemu działają dla list dowolnego typu elementu. Poniższy przykład dodaje do `List<T>` bezpieczny dostęp do elementu oraz dzielenie na porcje (chunking).

```dart
// Generyczne rozszerzenie na List<T> — działa dla listy dowolnego typu
extension ListUtils<T> on List<T> {
  // Bezpieczny dostęp: zwraca element lub null zamiast rzucać wyjątek
  T? elementLubNull(int indeks) {
    if (indeks < 0 || indeks >= length) return null;
    return this[indeks];
  }

  // Dzieli listę na porcje o zadanym rozmiarze
  List<List<T>> naPorcje(int rozmiar) {
    final wynik = <List<T>>[];
    for (var i = 0; i < length; i += rozmiar) {
      // sublist z ograniczeniem, by nie wyjść poza koniec listy
      final koniec = (i + rozmiar < length) ? i + rozmiar : length;
      wynik.add(sublist(i, koniec));
    }
    return wynik;
  }
}

void main() {
  final liczby = [1, 2, 3, 4, 5];

  // Bezpieczny dostęp — brak wyjątku dla indeksu spoza zakresu
  print(liczby.elementLubNull(2)); // 3
  print(liczby.elementLubNull(10)); // null

  // Podział na porcje po 2 elementy
  print(liczby.naPorcje(2)); // [[1, 2], [3, 4], [5]]

  // To samo rozszerzenie działa dla List<String>
  print(['a', 'b', 'c'].naPorcje(2)); // [[a, b], [c]]
}
// Oczekiwane wyjście:
// 3
// null
// [[1, 2], [3, 4], [5]]
// [[a, b], [c]]
```

---

## Rozstrzyganie w czasie kompilacji

Kluczowa cecha extension methods: to, która metoda zostanie wywołana, rozstrzyga się w **czasie kompilacji** na podstawie **statycznego typu** wyrażenia, a nie jego rzeczywistego typu w czasie wykonania. Extension methods nie są więc polimorficzne jak zwykłe metody instancyjne.

Poniższy przykład pokazuje, że ta sama wartość widziana jako `dynamic` traci dostęp do metody rozszerzającej — bo kompilator nie zna jej statycznego typu.

```dart
extension NaInt on int {
  String opis() => 'liczba $this';
}

void main() {
  int liczba = 5;
  print(liczba.opis()); // działa: statyczny typ to int

  // Jako dynamic kompilator nie wie, że to int -> brak dostępu do ekstensji
  dynamic dyn = 5;
  try {
    // Wywołanie na dynamic próbuje odnaleźć metodę w czasie wykonania,
    // a metody rozszerzające nie istnieją jako prawdziwe metody int
    print(dyn.opis());
  } on NoSuchMethodError {
    print('Brak metody opis() na typie dynamic');
  }
}
// Oczekiwane wyjście:
// liczba 5
// Brak metody opis() na typie dynamic
```

To potwierdza, że extension method jest tylko „lukrem składniowym": kompilator zamienia `liczba.opis()` na wywołanie statycznej funkcji pomocniczej. Nic nie zostaje dodane do samego typu `int` w czasie wykonania.

---

## Extension types (Dart 3) — podstawy

**Extension type** tworzy nowy typ statyczny opakowujący istniejący typ (**typ reprezentacji**). Deklaracja zawiera konstruktor określający wartość reprezentacji. W czasie wykonania nie powstaje żaden nowy obiekt — extension type jest **zerokosztowym** widokiem na wartość reprezentacji.

Typowe zastosowanie to nadanie „surowej" wartości (np. `int` czy `String`) znaczenia domenowego i ograniczenie zestawu operacji. Poniższy przykład opakowuje `int` w typ `Wiek`, który udostępnia tylko sensowne operacje.

```dart
// Extension type opakowujący int jako typ domenowy "Wiek".
// Składnia: extension type Nazwa(TypReprezentacji nazwaPola)
extension type Wiek(int wartosc) {
  // Metoda dostępna na typie Wiek
  bool get pelnoletni => wartosc >= 18;

  // Możemy dodawać lata, zwracając nowy Wiek
  Wiek dodajLata(int lata) => Wiek(wartosc + lata);
}

void main() {
  final w = Wiek(16);
  print(w.wartosc); // 16 — dostęp do wartości reprezentacji
  print(w.pelnoletni); // false

  final starszy = w.dodajLata(5);
  print(starszy.wartosc); // 21
  print(starszy.pelnoletni); // true
}
// Oczekiwane wyjście:
// 16
// false
// 21
// true
```

Domyślnie extension type **nie ujawnia** metod typu reprezentacji — `Wiek` nie jest po prostu `int`, więc nie możesz na nim wywołać np. `+`. Dzięki temu tworzysz szczelną abstrakcję: użytkownik widzi tylko operacje, które zdefiniujesz.

### implements — ujawnianie interfejsu reprezentacji

Jeśli chcesz, aby extension type udostępniał metody typu reprezentacji i był z nim przypisywalny, użyj klauzuli `implements`. Poniższy przykład tworzy typ `Id` na bazie `String`, który zachowuje dostęp do metod `String`.

```dart
// implements String sprawia, że Id "jest" Stringiem dla systemu typów
extension type Id(String wartosc) implements String {
  // Dodatkowa walidacja jako metoda pomocnicza
  bool get poprawny => wartosc.isNotEmpty && wartosc.length <= 20;
}

void main() {
  final id = Id('user-42');

  // Dzięki implements String dostępne są metody String
  print(id.toUpperCase()); // USER-42
  print(id.length); // 7

  // ...oraz nasza własna metoda
  print(id.poprawny); // true

  // Id jest przypisywalny do String
  String jakoString = id;
  print(jakoString); // user-42
}
// Oczekiwane wyjście:
// USER-42
// 7
// true
// user-42
```

---

## Extension types są zerokosztowe w czasie wykonania

Najważniejsza cecha extension type: nowy typ istnieje **wyłącznie w czasie kompilacji**. W czasie wykonania wartość jest tożsama z wartością reprezentacji — nie ma opakowującego obiektu, nie ma narzutu pamięci ani alokacji.

Poniższy przykład pokazuje, że w czasie wykonania `Wiek` jest po prostu `int`.

```dart
extension type Wiek(int wartosc) {}

void main() {
  final w = Wiek(30);

  // W czasie wykonania obiekt jest zwykłym int — brak opakowania
  print(w is int); // true
  print(identical(w, 30)); // true — to ta sama wartość, nie kopia

  // Runtime type to int, a nie "Wiek"
  print((w as dynamic).runtimeType); // int
}
// Oczekiwane wyjście:
// true
// true
// int
```

To zasadnicza różnica względem opakowania przez zwykłą klasę (`class Wiek { final int wartosc; ... }`), które tworzyłoby prawdziwy obiekt na stercie z osobną tożsamością i narzutem pamięci. Extension type daje bezpieczeństwo typów **za darmo** w czasie wykonania — ale ceną jest to, że kontrola typu jest tylko statyczna (patrz sekcja porównawcza).

---

## Sekcja wyjaśniająca: extension methods vs extension types

Oba mechanizmy pozwalają wzbogacić istniejące typy, ale rozwiązują różne problemy. Poniższe porównanie zestawia ich zachowanie i wskazuje, kiedy wybrać który.

### Porównanie

| Cecha | Extension methods | Extension types |
|-------|-------------------|-----------------|
| **Czy tworzy nowy typ?** | Nie — dokleja metody do istniejącego typu | Tak — nowy typ statyczny opakowujący typ reprezentacji |
| **Reprezentacja w czasie wykonania** | Brak — czysty lukier składniowy, wywołanie funkcji statycznej | Wartość reprezentacji (np. `int`); brak opakowującego obiektu |
| **Kontrola typu** | Statyczna — dostępność metody zależy od statycznego typu | Statyczna — nowy typ istnieje tylko dla kompilatora |
| **Ograniczanie API typu bazowego** | Nie — nie da się „ukryć" istniejących metod | Tak — domyślnie ujawnia tylko zdefiniowane metody (chyba że `implements`) |
| **Bezpieczeństwo w czasie wykonania** | Nie zmienia — wartość ma swój oryginalny typ | Ograniczone — po rzutowaniu na `dynamic` typ znika (to nadal `int`) |
| **Narzut pamięci/wydajności** | Zerowy | Zerowy (zero-cost abstraction) |
| **Główne zastosowanie** | Dodawanie metod narzędziowych do istniejących typów | Tworzenie typów domenowych i szczelnych abstrakcji nad prymitywami |

### Compile-time vs runtime — sedno różnicy

- **Extension method** to wyłącznie mechanizm **czasu kompilacji**: kompilator przepisuje `wartosc.metoda()` na wywołanie statycznej funkcji pomocniczej. W czasie wykonania nie istnieje żadna „metoda rozszerzająca" — dlatego nie działa przez `dynamic` i nie jest polimorficzna.
- **Extension type** to również konstrukcja **czasu kompilacji**, ale wprowadza do systemu typów **nowy typ**. W czasie wykonania wartość jest identyczna z wartością reprezentacji — dlatego po rzutowaniu na `dynamic` typ domenowy znika i widoczny jest goły typ reprezentacji. Bezpieczeństwo jest więc gwarantowane statycznie, a nie w czasie wykonania.

### Kiedy wybrać który?

- **Wybierz extension method**, gdy chcesz **dodać zachowanie** do istniejącego typu, ale nadal traktować wartości jako ten sam typ. Idealne do bibliotek narzędziowych: `String.zWielkiejLitery()`, `List.naPorcje()`, `DateTime.czyDzisiaj()`. Nie potrzebujesz nowego typu — tylko wygodniejszej składni.
- **Wybierz extension type**, gdy chcesz nadać prymitywnej wartości **znaczenie domenowe** i **ograniczyć zestaw operacji**, nie płacąc za opakowanie w czasie wykonania. Idealne do typów takich jak `UserId`, `Wiek`, `Kelwiny`, gdzie zależy Ci, by kompilator nie pozwolił pomylić `UserId` z gołym `String` czy dodać do siebie dwóch niezwiązanych identyfikatorów.
- **Rozważ zwykłą klasę**, gdy potrzebujesz prawdziwej tożsamości obiektu w czasie wykonania, dziedziczenia lub gdy narzut jednej alokacji nie ma znaczenia, a zależy Ci na pełnej hermetyzacji także w czasie wykonania.

---

## Ćwiczenie 1: Rozszerzenie DurationFormat na int

### Opis problemu

Napisz rozszerzenie `CzasUtils` na typie `int`, traktujące liczbę jako liczbę **sekund**, z metodą `String jakoCzas()`, która formatuje wartość jako `H:MM:SS` (godziny bez wiodącego zera, minuty i sekundy zawsze dwucyfrowe). Załóż wartości nieujemne.

**Poziom trudności:** basic

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `3661.jakoCzas()` | `1:01:01` |
| `59.jakoCzas()` | `0:00:59` |

### Wskazówki

1. Godziny to `this ~/ 3600`, pozostałe sekundy to `this % 3600`
2. Minuty to `(this % 3600) ~/ 60`, a sekundy to `this % 60`
3. Dwucyfrowe formatowanie uzyskasz przez `toString().padLeft(2, '0')`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
extension CzasUtils on int {
  // Traktuje wartość int jako liczbę sekund i formatuje jako H:MM:SS
  String jakoCzas() {
    final godziny = this ~/ 3600; // dzielenie całkowite
    final minuty = (this % 3600) ~/ 60;
    final sekundy = this % 60;

    // padLeft zapewnia dwie cyfry dla minut i sekund
    final mm = minuty.toString().padLeft(2, '0');
    final ss = sekundy.toString().padLeft(2, '0');
    return '$godziny:$mm:$ss';
  }
}

void main() {
  print(3661.jakoCzas()); // 1:01:01
  print(59.jakoCzas()); // 0:00:59
  print(7325.jakoCzas()); // 2:02:05
}
// Oczekiwane wyjście:
// 1:01:01
// 0:00:59
// 2:02:05
```

</details>

---

## Ćwiczenie 2: Extension type Temperatura

### Opis problemu

Zdefiniuj extension type `Kelwiny` opakowujący `double`. Dodaj:

- getter `naCelsjusze` zwracający temperaturę w stopniach Celsjusza (`K - 273.15`),
- getter `poprawna` sprawdzający, że temperatura nie jest poniżej zera absolutnego (`>= 0`),
- konstruktor nazwany `Kelwiny.zCelsjuszy(double c)` tworzący wartość z temperatury w Celsjuszach.

Dzięki extension type kompilator nie pozwoli pomylić `Kelwiny` ze zwykłym `double`, a mimo to w czasie wykonania nie ma żadnego narzutu.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Kelwiny(300).naCelsjusze` | `26.850000000000023` |
| `Kelwiny.zCelsjuszy(0).wartosc` | `273.15` |

### Wskazówki

1. Składnia głównego konstruktora: `extension type Kelwiny(double wartosc) { ... }`
2. Konstruktor nazwany definiujesz wewnątrz: `Kelwiny.zCelsjuszy(double c) : this(c + 273.15);`
3. `naCelsjusze` to po prostu `wartosc - 273.15`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// Extension type opakowujący double jako temperaturę w kelwinach
extension type Kelwiny(double wartosc) {
  // Konstruktor nazwany: tworzy Kelwiny z temperatury w Celsjuszach
  Kelwiny.zCelsjuszy(double c) : this(c + 273.15);

  // Przeliczenie na stopnie Celsjusza
  double get naCelsjusze => wartosc - 273.15;

  // Zero absolutne to 0 K — niższe wartości są fizycznie niepoprawne
  bool get poprawna => wartosc >= 0;
}

void main() {
  final t = Kelwiny(300);
  print(t.naCelsjusze); // 26.850000000000023 (błąd zmiennoprzecinkowy)
  print(t.poprawna); // true

  final zamarzanie = Kelwiny.zCelsjuszy(0);
  print(zamarzanie.wartosc); // 273.15

  final bledna = Kelwiny(-5);
  print(bledna.poprawna); // false

  // W czasie wykonania to nadal double — abstrakcja zerokosztowa
  print(t is double); // true
}
// Oczekiwane wyjście:
// 26.850000000000023
// true
// 273.15
// false
// true
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **Extension methods** dodają metody, gettery, settery i operatory do istniejących typów (nawet wbudowanych jak `String`, `int`, `List`) bez ich modyfikowania ani dziedziczenia — to lukier składniowy rozstrzygany w czasie kompilacji.
2. Rozstrzyganie extension method zależy od **statycznego typu** wyrażenia; dlatego nie działa przez `dynamic` i nie jest polimorficzne — w czasie wykonania nie istnieje żadna prawdziwa metoda na typie bazowym.
3. **Extension types** (Dart 3) tworzą **nowy typ statyczny** opakowujący typ reprezentacji; służą do nadawania prymitywom znaczenia domenowego i ograniczania zestawu operacji.
4. Extension type jest **zerokosztowy** w czasie wykonania — wartość jest tożsama z wartością reprezentacji (np. `Wiek` to w runtime po prostu `int`), bez alokacji i narzutu pamięci.
5. Klauzula `implements` na extension type udostępnia metody typu reprezentacji i czyni wartość z nim przypisywalną; bez niej abstrakcja jest szczelna i ujawnia tylko zdefiniowane metody.
6. **Wybieraj extension method** do reużywalnych narzędzi na istniejących typach, a **extension type** do bezpiecznych, zerokosztowych typów domenowych; sięgaj po zwykłą klasę, gdy potrzebujesz prawdziwej tożsamości i hermetyzacji także w czasie wykonania.

---

**Poprzedni moduł:** [Biblioteka standardowa — dart:math i dart:typed_data](../08-standard-library/06-dart-math-typed.md)
**Następny moduł:** [Rekordy i pattern matching](../09-advanced/02-records-patterns.md)
