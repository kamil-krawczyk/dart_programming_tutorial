---
id: "5.3"
title: "Mixiny"
difficulty: "intermediate"
section: "05-oop"
prerequisites:
  - "Dziedziczenie i interfejsy"
  - "Klasy i konstruktory"
---

# 5.3 Mixiny

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Dziedziczenie i interfejsy](../05-oop/02-inheritance.md), [Klasy i konstruktory](../05-oop/01-classes-constructors.md)
- **Cele nauki:**
  1. Rozumieć, czym są mixiny i jak umożliwiają ponowne wykorzystanie kodu bez klasycznego dziedziczenia
  2. Deklarować mixiny słowem kluczowym `mixin` i dołączać je do klas za pomocą `with`
  3. Ograniczać stosowanie mixinu do wybranej hierarchii klas za pomocą klauzuli `on`
  4. Rozpoznawać typowe błędy związane z mixinami i wiedzieć, jak ich unikać

---

## Słowo kluczowe `with` — dołączanie zachowań

Mixin to mechanizm **ponownego wykorzystania kodu** w wielu hierarchiach klas. Zamiast dziedziczyć po jednej nadklasie (`extends`), klasa może „wmieszać" (ang. *mix in*) zachowania z jednego lub wielu mixinów za pomocą słowa kluczowego `with`. Dzięki temu unikamy problemów wielokrotnego dziedziczenia, zachowując możliwość współdzielenia implementacji.

Poniższy przykład definiuje mixin `Logowalny` dodający metodę logowania i dołącza go do klasy usługi.

```dart
// Deklaracja mixinu — dostarcza gotowe zachowanie
mixin Logowalny {
  void log(String wiadomosc) {
    print('[LOG] $wiadomosc'); // wspólna implementacja logowania
  }
}

// with dołącza zachowania mixinu do klasy
class UslugaPlatnosci with Logowalny {
  void zaplac(double kwota) {
    log('Płatność: $kwota zł'); // metoda pochodzi z mixinu
  }
}

void main() {
  UslugaPlatnosci().zaplac(99.99);
}
// Oczekiwane wyjście:
// [LOG] Płatność: 99.99 zł
```

Drugi przykład pokazuje dołączanie **wielu mixinów** do jednej klasy. Kolejność po `with` ma znaczenie — mixiny są nakładane od lewej do prawej, a późniejszy mixin może nadpisać składnik wcześniejszego.

```dart
mixin Identyfikowalny {
  String get id => 'ID-${hashCode.abs()}';
}

mixin ZnacznikCzasu {
  DateTime get utworzono => DateTime(2024, 1, 1); // stała wartość dla powtarzalności
}

// Klasa łączy dwa mixiny naraz
class Dokument with Identyfikowalny, ZnacznikCzasu {
  final String tytul;
  Dokument(this.tytul);
}

void main() {
  final d = Dokument('Umowa');
  print('Tytuł: ${d.tytul}');
  print('Ma ID: ${d.id.startsWith('ID-')}'); // metoda z Identyfikowalny
  print('Rok: ${d.utworzono.year}');           // metoda z ZnacznikCzasu
}
// Oczekiwane wyjście:
// Tytuł: Umowa
// Ma ID: true
// Rok: 2024
```

---

## Deklaracja `mixin`

Mixin deklaruje się słowem kluczowym `mixin`. W odróżnieniu od zwykłej klasy, mixinu **nie można instancjonować** ani używać jako nadklasy w `extends` — służy wyłącznie do dołączania przez `with`. Mixin może zawierać metody, gettery, settery, a także pola instancji (jednak nie może deklarować konstruktora).

Poniższy przykład definiuje mixin z polem i metodami operującymi na tym polu — mixiny mogą przechowywać stan.

```dart
// Mixin ze stanem — pole licznik jest dołączane do klasy docelowej
mixin LicznikWywolan {
  int _licznik = 0; // pole instancji w mixinie

  void zarejestruj() => _licznik++;
  int get liczbaWywolan => _licznik;
}

class Endpoint with LicznikWywolan {
  final String sciezka;
  Endpoint(this.sciezka);

  void obsluz() {
    zarejestruj(); // aktualizuje stan z mixinu
  }
}

void main() {
  final e = Endpoint('/api/users');
  e.obsluz();
  e.obsluz();
  e.obsluz();
  print('${e.sciezka}: ${e.liczbaWywolan} wywołań');
}
// Oczekiwane wyjście:
// /api/users: 3 wywołań
```

Drugi przykład pokazuje mixin definiujący metodę wykorzystywaną przez wiele niezależnych klas — kluczowa zaleta mixinów nad dziedziczeniem.

```dart
mixin Porownywalny {
  int get wartosc; // getter abstrakcyjny — klasa docelowa musi go dostarczyć

  bool wiekszyNiz(Porownywalny inny) => wartosc > inny.wartosc;
}

// Dwie niepowiązane klasy współdzielą logikę porównania przez ten sam mixin
class Produkt with Porownywalny {
  @override
  final int wartosc; // cena
  Produkt(this.wartosc);
}

class Zawodnik with Porownywalny {
  @override
  final int wartosc; // liczba punktów
  Zawodnik(this.wartosc);
}

void main() {
  print(Produkt(100).wiekszyNiz(Produkt(50)));   // true
  print(Zawodnik(3).wiekszyNiz(Zawodnik(8)));    // false
}
// Oczekiwane wyjście:
// true
// false
```

---

## Klauzula `on` — ograniczanie mixinu

Klauzula `on` ogranicza mixin do klas, które dziedziczą po (lub są) wskazanym typie. Dzięki temu mixin może **bezpiecznie odwoływać się** do składników tego typu (w tym przez `super`), mając gwarancję, że klasa docelowa je posiada. Mixin z klauzulą `on Typ` może być dołączony wyłącznie do klas będących podtypami `Typ`.

Poniższy przykład definiuje mixin ograniczony do klasy `Zwierze` — dzięki `on` mixin ma dostęp do pól i metod `Zwierze`.

```dart
class Zwierze {
  final String imie;
  Zwierze(this.imie);

  String dzwiek() => '...';
}

// Mixin dostępny tylko dla podtypów Zwierze — ma dostęp do jego składników
mixin GlosneZwierze on Zwierze {
  String glosnyDzwiek() => '${dzwiek().toUpperCase()}! (${imie})'; // korzysta z dzwiek() i imie z Zwierze
}

class Pies extends Zwierze with GlosneZwierze {
  Pies(String imie) : super(imie);

  @override
  String dzwiek() => 'hau';
}

void main() {
  print(Pies('Reksio').glosnyDzwiek());
}
// Oczekiwane wyjście:
// HAU! (Reksio)
```

Drugi przykład pokazuje mixin z klauzulą `on`, który nadpisuje metodę nadklasy i wywołuje `super` — częsty wzorzec do „owijania" zachowania (np. walidacja, buforowanie).

```dart
abstract class Repozytorium {
  String zapisz(String dane) => 'zapisano: $dane';
}

// Mixin on Repozytorium może wywołać super.zapisz z nadklasy
mixin Walidacja on Repozytorium {
  @override
  String zapisz(String dane) {
    if (dane.isEmpty) return 'BŁĄD: puste dane';
    return 'walidacja OK -> ${super.zapisz(dane)}'; // super odwołuje się do Repozytorium
  }
}

class RepozytoriumPlikowe extends Repozytorium with Walidacja {}

void main() {
  final repo = RepozytoriumPlikowe();
  print(repo.zapisz('dokument'));
  print(repo.zapisz(''));
}
// Oczekiwane wyjście:
// walidacja OK -> zapisano: dokument
// BŁĄD: puste dane
```

---

## Typowe błędy z mixinami

### Błędne użycie: naruszenie klauzuli `on`

Dołączenie mixinu z klauzulą `on Typ` do klasy, która nie jest podtypem `Typ`, jest błędem kompilacji.

```dart
class Zwierze {
  String dzwiek() => '...';
}

mixin GlosneZwierze on Zwierze {
  String glosno() => dzwiek().toUpperCase();
}

// BŁĄD: Robot nie dziedziczy po Zwierze, więc nie może użyć mixinu GlosneZwierze
class Robot with GlosneZwierze {}

void main() {
  Robot();
}
// Błąd kompilacji:
// 'GlosneZwierze' can't be mixed onto 'Object' because 'Object' doesn't
// implement 'Zwierze'.
// Try extending or implementing 'Zwierze', or ...

// Poprawna wersja:
// class Robot extends Zwierze with GlosneZwierze { ... }
```

### Błędne użycie: instancjonowanie mixinu

Mixinu nie można instancjonować bezpośrednio ani używać jako typu konstruowanego.

```dart
mixin Logowalny {
  void log(String s) => print('[LOG] $s');
}

void main() {
  // BŁĄD: mixin nie jest klasą i nie może być instancjonowany
  final l = Logowalny();
  l.log('test');
}
// Błąd kompilacji:
// Mixins can't be instantiated.
```

### Konflikt składników przy wielu mixinach

Gdy dwa mixiny definiują składnik o tej samej nazwie, „wygrywa" mixin wymieniony **później** po `with`. Poniższy przykład ilustruje tę regułę rozwiązywania konfliktów.

```dart
mixin A {
  String kto() => 'A';
}

mixin B {
  String kto() => 'B';
}

// Kolejność: A najpierw, B później — B nadpisuje kto()
class Kombinacja with A, B {}

void main() {
  print(Kombinacja().kto()); // ostatni mixin (B) wygrywa
}
// Oczekiwane wyjście:
// B
```

---

## Ćwiczenie 1: Mixin serializacji JSON

### Opis problemu

Zdefiniuj mixin `JsonSerializowalny` z:

- abstrakcyjnym getterem `Map<String, dynamic> get pola;`
- konkretną metodą `String toJson()` zwracającą pola sformatowane jako uproszczony JSON w formacie `{"klucz":wartosc,...}` (pary posortowane nie są wymagane; użyj kolejności z mapy).

Następnie utwórz dwie niepowiązane klasy — `Uzytkownik` (pola: `nazwa`, `wiek`) oraz `Produkt` (pola: `tytul`, `cena`) — obie korzystające z mixinu.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Uzytkownik('Ala', 30).toJson()` | `{"nazwa":Ala,"wiek":30}` |
| `Produkt('Kubek', 25).toJson()` | `{"tytul":Kubek,"cena":25}` |

### Wskazówki

1. Zadeklaruj w mixinie abstrakcyjny getter `pola` — klasy docelowe muszą go dostarczyć
2. W `toJson()` iteruj po `pola.entries` i sklej pary `"$klucz":$wartosc` przecinkami
3. Możesz użyć `pola.entries.map(...).join(',')` do złożenia wyniku

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
mixin JsonSerializowalny {
  Map<String, dynamic> get pola; // kontrakt dla klasy docelowej

  String toJson() {
    final pary = pola.entries.map((e) => '"${e.key}":${e.value}').join(',');
    return '{$pary}';
  }
}

class Uzytkownik with JsonSerializowalny {
  final String nazwa;
  final int wiek;
  Uzytkownik(this.nazwa, this.wiek);

  @override
  Map<String, dynamic> get pola => {'nazwa': nazwa, 'wiek': wiek};
}

class Produkt with JsonSerializowalny {
  final String tytul;
  final int cena;
  Produkt(this.tytul, this.cena);

  @override
  Map<String, dynamic> get pola => {'tytul': tytul, 'cena': cena};
}

void main() {
  print(Uzytkownik('Ala', 30).toJson());
  print(Produkt('Kubek', 25).toJson());
}
// Oczekiwane wyjście:
// {"nazwa":Ala,"wiek":30}
// {"tytul":Kubek,"cena":25}
```

</details>

---

## Ćwiczenie 2: Mixin `on` z rejestrowaniem zdarzeń

### Opis problemu

Zaprojektuj:

- Abstrakcyjną klasę `Widget` z metodą `String renderuj()`.
- Mixin `Klikalny` z klauzulą `on Widget`, który dodaje metodę `String klik()`. Metoda zwraca tekst w formacie `Klik na: <wynik renderuj()>`, korzystając z metody `renderuj()` klasy docelowej.
- Konkretną klasę `Przycisk` dziedziczącą po `Widget` z dołączonym mixinem `Klikalny`. `renderuj()` zwraca `[Przycisk: <etykieta>]`.

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Przycisk('OK').renderuj()` | `[Przycisk: OK]` |
| `Przycisk('OK').klik()` | `Klik na: [Przycisk: OK]` |

### Wskazówki

1. Klauzula `on Widget` gwarantuje mixinowi dostęp do metody `renderuj()`
2. W mixinie możesz bezpośrednio wywołać `renderuj()` — kompilator wie, że klasa docelowa jest podtypem `Widget`
3. `Przycisk` musi `extends Widget with Klikalny` (najpierw `extends`, potem `with`)

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
abstract class Widget {
  String renderuj();
}

// Mixin ograniczony do podtypów Widget — ma dostęp do renderuj()
mixin Klikalny on Widget {
  String klik() => 'Klik na: ${renderuj()}';
}

class Przycisk extends Widget with Klikalny {
  final String etykieta;
  Przycisk(this.etykieta);

  @override
  String renderuj() => '[Przycisk: $etykieta]';
}

void main() {
  final b = Przycisk('OK');
  print(b.renderuj());
  print(b.klik());
}
// Oczekiwane wyjście:
// [Przycisk: OK]
// Klik na: [Przycisk: OK]
```

</details>

---

**Poprzedni moduł:** [Dziedziczenie i interfejsy](../05-oop/02-inheritance.md)
**Następny moduł:** [Modyfikatory klas w Dart 3](../05-oop/04-dart3-modifiers.md)
