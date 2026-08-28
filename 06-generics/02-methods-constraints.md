---
id: "6.2"
title: "Metody generyczne i ograniczenia typów"
difficulty: "intermediate"
section: "06-generics"
prerequisites:
  - "Klasy generyczne"
  - "Klasy i konstruktory"
---

# 6.2 Metody generyczne i ograniczenia typów

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Klasy generyczne](01-generic-classes.md), [Klasy i konstruktory](../05-oop/01-classes-constructors.md)
- **Cele nauki:**
  1. Definiować i wywoływać metody oraz funkcje generyczne z własnymi parametrami typowymi
  2. Stosować ograniczenia typów (`extends`) do zawężenia dopuszczalnych argumentów typowych i uzyskania dostępu do składowych typu bazowego
  3. Rozpoznawać błąd kompilacji powstający przy naruszeniu ograniczenia typu i rozumieć jego komunikat
  4. Rozumieć kowariancję parametrów typowych w Dart oraz różnicę między bezpiecznym a niebezpiecznym przypisaniem prowadzącym do błędu w czasie działania

---

## Metody i funkcje generyczne

Metoda (lub funkcja) generyczna deklaruje własne parametry typowe w nawiasach ostrokątnych po nazwie: `T nazwa<T>(...)`. Parametr typowy jest lokalny dla tej metody — działa niezależnie od ewentualnych parametrów typowych klasy.

```dart
// Funkcja generyczna — parametr typowy T lokalny dla funkcji
T ostatni<T>(List<T> lista) => lista.last;

// Funkcja generyczna z dwoma parametrami typowymi
List<W> mapuj<T, W>(List<T> zrodlo, W Function(T) transformacja) {
  return [for (final e in zrodlo) transformacja(e)];
}

void main() {
  // T wywnioskowane z argumentu
  print(ostatni([1, 2, 3]));       // T = int
  print(ostatni(['x', 'y', 'z'])); // T = String

  // T = int, W = String — mapuje liczby na ich długość tekstową
  var opisy = mapuj([1, 22, 333], (n) => 'liczba $n');
  print(opisy);
}
// Oczekiwane wyjście:
// 3
// z
// [liczba 1, liczba 22, liczba 333]
```

Drugi przykład to metoda generyczna wewnątrz **niegenerycznej** klasy — parametr typowy należy tylko do metody.

```dart
// Metoda generyczna w zwykłej (niegenerycznej) klasie narzędziowej
class Narzedzia {
  // Zwraca pierwszy element spełniający warunek lub null
  T? znajdz<T>(List<T> lista, bool Function(T) warunek) {
    for (final element in lista) {
      if (warunek(element)) return element;
    }
    return null;
  }

  // Zamienia dwa elementy listy miejscami, zachowując typ
  void zamien<T>(List<T> lista, int i, int j) {
    final tmp = lista[i];
    lista[i] = lista[j];
    lista[j] = tmp;
  }
}

void main() {
  final n = Narzedzia();

  var pierwszaDuza = n.znajdz([3, 7, 12, 5], (x) => x > 10);
  print('Pierwsza > 10: $pierwszaDuza');

  var slowa = ['a', 'b', 'c'];
  n.zamien(slowa, 0, 2);
  print('Po zamianie: $slowa');
}
// Oczekiwane wyjście:
// Pierwsza > 10: 12
// Po zamianie: [c, b, a]
```

---

## Ograniczenia typów (`extends`)

Domyślnie parametr typowy `T` może być dowolnym typem, więc wewnątrz kodu generycznego mamy dostęp jedynie do składowych klasy `Object` (np. `toString`, `hashCode`). Aby korzystać z bogatszego API, **ograniczamy** parametr typowy słowem `extends`: `<T extends TypBazowy>`. Oznacza to „`T` musi być podtypem `TypBazowy`" i daje dostęp do wszystkich składowych `TypBazowy`.

Poniższy przykład ogranicza `T` do `num`, dzięki czemu wewnątrz funkcji można wykonywać operacje arytmetyczne. Zadziała ona zarówno dla `int`, jak i `double`.

```dart
// T ograniczone do num — mamy dostęp do operacji arytmetycznych num
T maksimum<T extends num>(List<T> liczby) {
  var najw = liczby.first;
  for (final x in liczby) {
    if (x > najw) najw = x; // operator > pochodzi z num
  }
  return najw;
}

void main() {
  print(maksimum([3, 7, 2, 9, 4]));       // działa dla List<int>
  print(maksimum([1.5, 2.75, 0.25]));     // działa dla List<double>
}
// Oczekiwane wyjście:
// 9
// 2.75
```

Drugi przykład ogranicza parametr typowy do interfejsu `Comparable`. Uwaga: w Dart `int` implementuje `Comparable<num>`, a nie `Comparable<int>`, dlatego jako ograniczenie stosujemy `Comparable<dynamic>` — pasuje ono do `int`, `double`, `String` i innych typów porównywalnych.

```dart
// Ograniczenie do Comparable<dynamic> — działa dla int, double, String itp.
// (int implementuje Comparable<num>, więc Comparable<int> NIE zadziałałoby)
T najmniejszy<T extends Comparable<dynamic>>(List<T> elementy) {
  var min = elementy.first;
  for (final e in elementy) {
    if (e.compareTo(min) < 0) min = e; // compareTo pochodzi z Comparable
  }
  return min;
}

void main() {
  print(najmniejszy([5, 3, 8, 1]));                 // int
  print(najmniejszy(['banan', 'jabłko', 'arbuz'])); // String — porządek leksykalny
}
// Oczekiwane wyjście:
// 1
// arbuz
```

---

## Naruszenie ograniczenia — błąd kompilacji

Gdy jako argument typowy podamy typ, który **nie** spełnia ograniczenia, kompilator zgłasza błąd. Dzięki temu nieprawidłowe użycie zostaje wykryte, zanim program się uruchomi.

Poniższy przykład pokazuje poprawne użycie ograniczenia `<T extends num>` — `int` i `double` są podtypami `num`.

```dart
// ✅ POPRAWNIE — argument typowy spełnia ograniczenie extends num
class Akumulator<T extends num> {
  T suma;
  Akumulator(this.suma);

  void dodaj(T wartosc) {
    // Operacja możliwa, bo T na pewno jest podtypem num
    suma = (suma + wartosc) as T;
  }
}

void main() {
  var akInt = Akumulator<int>(0); // int extends num — OK
  akInt.dodaj(5);
  akInt.dodaj(3);
  print('Suma int: ${akInt.suma}');

  var akDouble = Akumulator<double>(1.5); // double extends num — OK
  akDouble.dodaj(0.5);
  print('Suma double: ${akDouble.suma}');
}
// Oczekiwane wyjście:
// Suma int: 8
// Suma double: 2.0
```

A tak wygląda **naruszenie** ograniczenia — próba użycia `String` jako argumentu typowego dla parametru ograniczonego do `num`:

```dart
// ❌ BŁĄD KOMPILACJI — String nie jest podtypem num
class Akumulator<T extends num> {
  T suma;
  Akumulator(this.suma);
}

void main() {
  // String nie spełnia ograniczenia extends num
  var bledny = Akumulator<String>('tekst');
  // Błąd: 'String' doesn't conform to the bound 'num' of the type
  //       parameter 'T'.
  //       Try using a type that is or is a subclass of 'num'.
  print(bledny.suma);
}
```

---

## Kowariancja parametrów typowych

W Dart typy generyczne są **kowariantne** względem swojego parametru: jeśli `Kot` jest podtypem `Zwierze`, to `List<Kot>` jest podtypem `List<Zwierze>`. To wygodne przy odczycie, ale otwiera furtkę do niebezpiecznych operacji przy zapisie.

Poniższy przykład pokazuje **bezpieczne** wykorzystanie kowariancji — przypisujemy `List<Kot>` do zmiennej typu `List<Zwierze>` i tylko **odczytujemy** elementy. Odczyt jest zawsze bezpieczny, bo każdy `Kot` jest `Zwierze`.

```dart
// Bezpieczna kowariancja — odczyt z listy przypisanej do nadtypu
class Zwierze {
  String dzwiek() => 'jakiś dźwięk';
}

class Kot extends Zwierze {
  @override
  String dzwiek() => 'miau';
}

void main() {
  List<Kot> koty = [Kot(), Kot()];

  // Kowariancja: List<Kot> można przypisać do List<Zwierze>
  List<Zwierze> zwierzeta = koty;

  // Odczyt jest bezpieczny — każdy element na pewno jest Zwierze
  for (final z in zwierzeta) {
    print(z.dzwiek());
  }
}
// Oczekiwane wyjście:
// miau
// miau
```

Drugi przykład ilustruje **niebezpieczną** stronę kowariancji. Po przypisaniu `List<Kot>` do `List<Zwierze>` próba **zapisu** obiektu `Pies` (który jest `Zwierze`, ale nie `Kot`) kompiluje się poprawnie, lecz powoduje **błąd w czasie działania** (`TypeError`), ponieważ faktyczna lista przechowuje wyłącznie koty.

```dart
// Niebezpieczna kowariancja — zapis niezgodnego podtypu daje błąd runtime
class Zwierze {}

class Kot extends Zwierze {}

class Pies extends Zwierze {}

void main() {
  List<Kot> koty = [Kot()];
  List<Zwierze> jakoZwierzeta = koty; // dozwolone dzięki kowariancji

  print('Przed próbą zapisu: ${koty.length} kot(ów)');

  try {
    // Kompiluje się (Pies to Zwierze), ale faktyczna lista to List<Kot>
    jakoZwierzeta.add(Pies());
  } on TypeError {
    // Błąd wykryty w czasie działania — ochrona spójności typów
    print('Błąd w czasie działania: przechwycono TypeError');
  }
}
// Oczekiwane wyjście:
// Przed próbą zapisu: 1 kot(ów)
// Błąd w czasie działania: przechwycono TypeError
```

Wniosek: kowariancja jest bezpieczna przy odczycie, ale zapis przez referencję do nadtypu może naruszyć typ faktycznej kolekcji. Dart chroni przed tym kontrolą w czasie działania, jednak najlepiej unikać zapisu do kolekcji przypisanej do jej nadtypu.

---

## Ćwiczenie 1: Generyczne sortowanie z ograniczeniem (intermediate)

### Opis problemu

Napisz funkcję generyczną `List<T> posortowane<T extends Comparable<dynamic>>(List<T> lista)`, która zwraca **nową** posortowaną rosnąco listę, nie modyfikując oryginału. Wykorzystaj ograniczenie typu, aby zagwarantować, że elementy da się porównać metodą `compareTo`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `posortowane([3, 1, 2])` | `[1, 2, 3]` |
| `posortowane(['c', 'a', 'b'])` | `[a, b, c]` |

### Wskazówki

1. Ograniczenie `T extends Comparable<dynamic>` daje dostęp do metody `compareTo`. Nie używaj `Comparable<T>`, bo `int` implementuje `Comparable<num>`, a nie `Comparable<int>`.
2. Aby nie modyfikować oryginału, utwórz kopię: `List<T>.of(lista)` lub `[...lista]`.
3. Metoda `sort` przyjmuje komparator `(a, b) => a.compareTo(b)`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// Ograniczenie Comparable<dynamic> działa dla int, double, String itd.
List<T> posortowane<T extends Comparable<dynamic>>(List<T> lista) {
  final kopia = [...lista]; // kopiujemy, by nie zmieniać oryginału
  kopia.sort((a, b) => a.compareTo(b));
  return kopia;
}

void main() {
  var liczby = [3, 1, 2];
  print(posortowane(liczby)); // [1, 2, 3]
  print(liczby);              // [3, 1, 2] — oryginał bez zmian

  print(posortowane(['c', 'a', 'b'])); // [a, b, c]
}
// Oczekiwane wyjście:
// [1, 2, 3]
// [3, 1, 2]
// [a, b, c]
```

</details>

---

## Ćwiczenie 2: Generyczne drzewo binarne wyszukiwań (advanced)

### Opis problemu

Zaimplementuj generyczne **drzewo binarnych wyszukiwań** (BST) `Drzewo<T extends Comparable<dynamic>>` przechowujące unikalne, porównywalne wartości. Klasa powinna udostępniać:

1. Metodę `void wstaw(T wartosc)` — wstawia wartość zgodnie z regułą BST (mniejsze w lewo, większe w prawo).
2. Metodę `bool zawiera(T wartosc)` — sprawdza, czy wartość znajduje się w drzewie.
3. Metodę `List<T> wPorzadku()` — zwraca wartości w porządku rosnącym (przejście in-order).

Wykorzystaj ograniczenie `Comparable<dynamic>`, aby móc porównywać elementy metodą `compareTo`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `wstaw(5)`, `wstaw(3)`, `wstaw(8)`, potem `wPorzadku()` | `[3, 5, 8]` |
| po wstawieniu `5, 3, 8`, potem `zawiera(3)` | `true` |

### Wskazówki

1. Zamodeluj węzeł jako osobną generyczną klasę `Wezel<T>` z polami `wartosc`, `lewy` i `prawy` (oba typu `Wezel<T>?`).
2. Wstawianie i wyszukiwanie realizuj rekurencyjnie, porównując przez `wartosc.compareTo(inna)`.
3. Przejście in-order: najpierw lewe poddrzewo, potem bieżąca wartość, potem prawe poddrzewo — daje kolejność rosnącą.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Wezel<T extends Comparable<dynamic>> {
  T wartosc;
  Wezel<T>? lewy;
  Wezel<T>? prawy;
  Wezel(this.wartosc);
}

class Drzewo<T extends Comparable<dynamic>> {
  Wezel<T>? _korzen;

  void wstaw(T wartosc) {
    _korzen = _wstawWezel(_korzen, wartosc);
  }

  Wezel<T> _wstawWezel(Wezel<T>? wezel, T wartosc) {
    if (wezel == null) return Wezel(wartosc);
    final porownanie = wartosc.compareTo(wezel.wartosc);
    if (porownanie < 0) {
      wezel.lewy = _wstawWezel(wezel.lewy, wartosc);
    } else if (porownanie > 0) {
      wezel.prawy = _wstawWezel(wezel.prawy, wartosc);
    }
    // porownanie == 0 → wartość już istnieje, pomijamy (unikalność)
    return wezel;
  }

  bool zawiera(T wartosc) {
    var biezacy = _korzen;
    while (biezacy != null) {
      final porownanie = wartosc.compareTo(biezacy.wartosc);
      if (porownanie == 0) return true;
      biezacy = porownanie < 0 ? biezacy.lewy : biezacy.prawy;
    }
    return false;
  }

  List<T> wPorzadku() {
    final wynik = <T>[];
    void przejdz(Wezel<T>? wezel) {
      if (wezel == null) return;
      przejdz(wezel.lewy);   // lewe poddrzewo
      wynik.add(wezel.wartosc); // wartość
      przejdz(wezel.prawy);  // prawe poddrzewo
    }

    przejdz(_korzen);
    return wynik;
  }
}

void main() {
  var drzewo = Drzewo<int>();
  for (final x in [5, 3, 8, 1, 4]) {
    drzewo.wstaw(x);
  }
  print(drzewo.wPorzadku()); // posortowane rosnąco
  print(drzewo.zawiera(3));  // true
  print(drzewo.zawiera(7));  // false
}
// Oczekiwane wyjście:
// [1, 3, 4, 5, 8]
// true
// false
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **Metody generyczne** deklarują własne parametry typowe (`T nazwa<T>(...)`), niezależne od parametrów typowych klasy — mogą występować także w klasach niegenerycznych.
2. **Ograniczenie `extends`** (`<T extends TypBazowy>`) zawęża dopuszczalne argumenty typowe i udostępnia wewnątrz kodu składowe typu bazowego (np. operatory `num` czy `compareTo` z `Comparable`).
3. W Dart `int` implementuje `Comparable<num>`, a nie `Comparable<int>` — dlatego jako ograniczenie porównywalności stosuje się `Comparable<dynamic>` (lub `Comparable<num>` dla liczb), by objąć `int`, `double` i `String`.
4. **Naruszenie ograniczenia** (podanie typu spoza dozwolonego zakresu) to **błąd kompilacji** z komunikatem „doesn't conform to the bound" — wykryty przed uruchomieniem programu.
5. Typy generyczne w Dart są **kowariantne**: `List<Kot>` jest podtypem `List<Zwierze>`. Odczyt jest bezpieczny.
6. **Zapis** przez referencję do nadtypu może naruszyć typ faktycznej kolekcji — kompiluje się, lecz Dart zgłasza `TypeError` w czasie działania, chroniąc spójność typów.

---

**Poprzedni moduł:** [Klasy generyczne](01-generic-classes.md)
**Następny moduł:** [Programowanie asynchroniczne — Futures i async/await](../07-concurrency/01-async-await.md)
