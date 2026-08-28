---
id: "1.3"
title: "Null safety"
difficulty: "beginner"
section: "01-basics"
prerequisites:
  - "Zmienne i typy danych"
---

# 1.3 Null safety

## Informacje o module

- **Poziom trudności:** beginner
- **Wymagania wstępne:** [Zmienne i typy danych](01-variables-types.md)
- **Cele nauki:**
  1. Rozumieć system sound null safety w Dart i różnicę między typami nullable a non-nullable
  2. Stosować operatory null-aware (`?.`, `??`, `??=`, `!`) w odpowiednich sytuacjach
  3. Rozumieć rolę `late` w kontekście null safety oraz rozpoznawać błędy kompilacji i runtime związane z null

---

## Czym jest sound null safety?

Dart implementuje **sound null safety** — oznacza to, że system typów gwarantuje w czasie kompilacji, że zmienna non-nullable nigdy nie będzie zawierać `null`. Jest to „sound" (solidny), ponieważ jeśli system typów mówi, że zmienna nie jest null, to na pewno nie jest — bez wyjątków.

Poniższy przykład pokazuje podstawową zasadę: typy non-nullable odrzucają null w czasie kompilacji:

```dart
void main() {
  // Typ non-nullable — NIGDY nie może być null
  String imie = 'Anna';
  int wiek = 25;

  // Typ nullable — MOŻE być null (oznaczony ?)
  String? pseudonim = null;
  int? numerDomu;  // domyślna wartość null

  print('Imię: $imie');          // Anna
  print('Pseudonim: $pseudonim'); // null
  print('Wiek: $wiek');           // 25
  print('Numer domu: $numerDomu'); // null
}
// Oczekiwane wyjście:
// Imię: Anna
// Pseudonim: null
// Wiek: 25
// Numer domu: null
```

Hierarchia typów z null safety:

```dart
void main() {
  // Każdy typ T ma swój odpowiednik nullable T?
  // T jest podtypem T?, ale T? NIE jest podtypem T

  String tekst = 'hello';
  String? nullableTekst = tekst; // OK — String jest podtypem String?

  // String innyTekst = nullableTekst; // Błąd kompilacji!
  // Nie można przypisać String? do String bez sprawdzenia null

  // Prawidłowa konwersja — z wartością domyślną
  String bezpieczny = nullableTekst ?? 'domyślna';
  print(bezpieczny); // hello
}
// Oczekiwane wyjście:
// hello
```

---

## Zapobieganie błędom w czasie kompilacji

Główną zaletą null safety jest wykrywanie potencjalnych błędów `NullPointerException` już w czasie kompilacji, zanim program zostanie uruchomiony.

Poniższy przykład demonstruje błędy kompilacji, które zapobiegają użyciu null w niebezpieczny sposób:

```dart
void main() {
  String? tekst = pobierzTekst(); // może zwrócić null

  // BŁĄD KOMPILACJI: The property 'length' can't be unconditionally accessed
  // because the receiver can be 'null'.
  // print(tekst.length); // ← ten kod nie skompiluje się!

  // Prawidłowe podejścia:

  // 1. Sprawdzenie null (type promotion)
  if (tekst != null) {
    print(tekst.length); // OK — Dart wie, że tekst nie jest null
  }

  // 2. Operator null-aware ?.
  print(tekst?.length); // zwróci null zamiast rzucać błąd

  // 3. Operator ?? z wartością domyślną
  print(tekst?.length ?? 0); // 0 gdy tekst jest null
}

String? pobierzTekst() => null; // symulacja braku danych
// Oczekiwane wyjście:
// null
// 0
```

Dart wymusza null-check przed użyciem wartości nullable w kontekstach wymagających non-nullable:

```dart
// Funkcja wymagająca non-nullable argumentu
int obliczDlugosc(String tekst) {
  return tekst.length;
}

void main() {
  String? input = 'Dart';

  // obliczDlugosc(input); // BŁĄD KOMPILACJI:
  // The argument type 'String?' can't be assigned to the parameter type 'String'.

  // Rozwiązanie 1: sprawdzenie null
  if (input != null) {
    print(obliczDlugosc(input)); // OK — type promotion do String
  }

  // Rozwiązanie 2: podanie wartości domyślnej
  print(obliczDlugosc(input ?? '')); // OK — ?? gwarantuje non-null

  // Rozwiązanie 3: operator ! (tylko gdy MAMY PEWNOŚĆ że nie null)
  print(obliczDlugosc(input!)); // OK — ale niebezpieczne jeśli input byłby null
}
// Oczekiwane wyjście:
// 4
// 4
// 4
```

---

## Operator null-aware `?.` (conditional member access)

Operator `?.` pozwala bezpiecznie uzyskać dostęp do właściwości lub metody obiektu, który może być null. Jeśli obiekt jest null, całe wyrażenie zwraca null zamiast rzucać wyjątek.

```dart
void main() {
  String? tekst = 'Hello Dart';

  // ?. — bezpieczny dostęp do właściwości
  print(tekst?.length);         // 10
  print(tekst?.toUpperCase());  // HELLO DART

  tekst = null;
  print(tekst?.length);         // null (nie rzuca błędu)
  print(tekst?.toUpperCase());  // null

  // Łańcuch ?. — bezpieczne zagnieżdżone wywołania
  List<String>? lista = ['a', 'b', 'c'];
  print(lista?.first?.toUpperCase()); // A

  lista = null;
  print(lista?.first?.toUpperCase()); // null
}
// Oczekiwane wyjście:
// 10
// HELLO DART
// null
// null
// A
// null
```

Operator `?.` w praktycznym scenariuszu z zagnieżdżonymi obiektami:

```dart
class Adres {
  final String? miasto;
  final String? ulica;
  Adres({this.miasto, this.ulica});
}

class Uzytkownik {
  final String nazwa;
  final Adres? adres;
  Uzytkownik(this.nazwa, {this.adres});
}

void main() {
  var user1 = Uzytkownik('Anna', adres: Adres(miasto: 'Kraków', ulica: 'Główna'));
  var user2 = Uzytkownik('Jan'); // bez adresu

  // ?. łańcuchowo — bezpiecznie nawiguje nullable pola
  print(user1.adres?.miasto?.toUpperCase()); // KRAKÓW
  print(user2.adres?.miasto?.toUpperCase()); // null (adres jest null)
  print(user1.adres?.ulica);                 // Główna
}
// Oczekiwane wyjście:
// KRAKÓW
// null
// Główna
```

---

## Operator `??` (if-null / null-coalescing)

Operator `??` zwraca lewą stronę jeśli nie jest null, a prawą stronę w przeciwnym przypadku. To elegancki sposób podawania wartości domyślnych.

```dart
void main() {
  String? imie = null;

  // ?? — wartość domyślna gdy lewa strona jest null
  var wyswietlane = imie ?? 'Anonim';
  print(wyswietlane); // Anonim

  imie = 'Kasia';
  wyswietlane = imie ?? 'Anonim';
  print(wyswietlane); // Kasia

  // Łańcuch ?? — pierwsza nie-null wartość
  String? a = null;
  String? b = null;
  String? c = 'trzecia';
  var wynik = a ?? b ?? c ?? 'żadna';
  print(wynik); // trzecia
}
// Oczekiwane wyjście:
// Anonim
// Kasia
// trzecia
```

Praktyczne użycie `??` z mapami i opcjonalnymi parametrami:

```dart
Map<String, String> konfiguracja = {
  'host': 'localhost',
  'port': '8080',
};

String pobierzUstawienie(String klucz, {String domyslna = 'nieznane'}) {
  // Map operator [] zwraca V? (nullable), więc ?? jest idealny
  return konfiguracja[klucz] ?? domyslna;
}

void main() {
  print(pobierzUstawienie('host'));                  // localhost
  print(pobierzUstawienie('baza'));                  // nieznane
  print(pobierzUstawienie('baza', domyslna: 'db1')); // db1
}
// Oczekiwane wyjście:
// localhost
// nieznane
// db1
```

---

## Operator `??=` (null-aware assignment)

Operator `??=` przypisuje wartość zmiennej TYLKO jeśli jej bieżąca wartość to null. Przydatny do leniwej inicjalizacji i ustawiania domyślnych wartości.

```dart
void main() {
  String? nazwa;
  print('Przed: $nazwa'); // null

  // ??= przypisuje tylko jeśli zmienna jest null
  nazwa ??= 'Dart';
  print('Po pierwszym ??=: $nazwa'); // Dart

  // Drugie ??= nie przypisze — nazwa już nie jest null
  nazwa ??= 'Java';
  print('Po drugim ??=: $nazwa'); // Dart (bez zmiany)
}
// Oczekiwane wyjście:
// Przed: null
// Po pierwszym ??=: Dart
// Po drugim ??=: Dart
```

Użycie `??=` do leniwej inicjalizacji cache:

```dart
class Cache {
  Map<String, String>? _dane;

  // ??= inicjalizuje mapę dopiero przy pierwszym użyciu
  Map<String, String> get dane => _dane ??= {};

  void zapisz(String klucz, String wartosc) {
    dane[klucz] = wartosc; // dane nigdy nie jest null dzięki ??= w getterze
  }

  String? odczytaj(String klucz) => _dane?[klucz];
}

void main() {
  var cache = Cache();
  print('Przed zapisem: ${cache.odczytaj("x")}'); // null (_dane nie istnieje)

  cache.zapisz('x', '42');
  print('Po zapisie: ${cache.odczytaj("x")}'); // 42
}
// Oczekiwane wyjście:
// Przed zapisem: null
// Po zapisie: 42
```

---

## Operator `!` (null assertion / bang operator)

Operator `!` mówi kompilatorowi: „jestem pewien, że ta wartość nie jest null". Jeśli wartość okaże się null w runtime, program rzuci wyjątek. Należy go używać oszczędnie i tylko gdy mamy pewność.

```dart
void main() {
  String? tekst = 'Hello';

  // ! — wymusza traktowanie nullable jako non-nullable
  String pewnyTekst = tekst!; // OK — tekst faktycznie nie jest null
  print(pewnyTekst.length); // 5

  // Użycie ! na wyniku metody
  Map<String, int> mapa = {'a': 1, 'b': 2};
  // Operator [] na Map zwraca V? (nullable)
  // Używamy ! bo WIEMY że klucz istnieje
  int wartosc = mapa['a']!;
  print(wartosc); // 1
}
// Oczekiwane wyjście:
// 5
// 1
```

Bezpieczne vs niebezpieczne użycie `!`:

```dart
void main() {
  // DOBRE użycie ! — po walidacji w logice programu
  var lista = ['pierwszy', 'drugi', 'trzeci'];
  String? znaleziony = lista.cast<String?>().firstWhere(
    (e) => e?.startsWith('d') ?? false,
    orElse: () => null,
  );

  if (znaleziony != null) {
    // Tu wiemy że znaleziony nie jest null, ale kompilator
    // nie zawsze może to wywnioskować w złożonych przypadkach
    print('Znaleziono: ${znaleziony.toUpperCase()}');
  }

  // ZŁAPMY potencjalny błąd ! w mapie
  Map<String, int> config = {'timeout': 30};
  // Lepiej użyj ?? zamiast ! gdy nie masz pewności:
  int port = config['port'] ?? 8080; // bezpieczne
  print('Port: $port');
}
// Oczekiwane wyjście:
// Znaleziono: DRUGI
// Port: 8080
```

---

## Naruszenie null safety w runtime (operator `!` na null)

Mimo że null safety zapobiega większości błędów w czasie kompilacji, operator `!` pozwala obejść te zabezpieczenia. Gdy `!` zostanie użyty na wartości null, program rzuci `TypeError` w runtime.

Poniższy przykład demonstruje co dzieje się gdy null safety zostanie naruszona:

```dart
void main() {
  String? tekst = null;

  try {
    // ! na wartości null rzuca TypeError w runtime
    var dlugoscc = tekst!.length; // BOOM!
    print(dlugoscc); // nigdy nie zostanie wykonane
  } on TypeError catch (e) {
    print('Runtime error: $e');
    print('Typ błędu: TypeError — null check operator used on a null value');
  }
}
// Oczekiwane wyjście:
// Runtime error: Null check operator used on a null value
// Typ błędu: TypeError — null check operator used on a null value
```

Bardziej realistyczny scenariusz naruszenia null safety — błąd ukryty w logice programu:

```dart
class Repozytorium {
  final Map<int, String> _uzytkownicy = {1: 'Anna', 2: 'Jan'};

  // Metoda zwraca nullable — użytkownik może nie istnieć
  String? znajdz(int id) => _uzytkownicy[id];
}

void main() {
  var repo = Repozytorium();

  // Programista błędnie zakłada że ID zawsze istnieje
  try {
    var nazwa = repo.znajdz(999)!; // 999 nie istnieje → null → BOOM!
    print('Użytkownik: $nazwa');
  } on TypeError catch (e) {
    print('Błąd: użytkownik nie istnieje');
    print('Szczegóły: $e');
  }

  // Prawidłowe podejście — obsługa braku danych
  var bezpieczna = repo.znajdz(999) ?? 'nieznany';
  print('Bezpieczne: $bezpieczna');
}
// Oczekiwane wyjście:
// Błąd: użytkownik nie istnieje
// Szczegóły: Null check operator used on a null value
// Bezpieczne: nieznany
```

---

## Zmienne `late` w kontekście null safety

Modyfikator `late` pozwala zadeklarować zmienną non-nullable bez natychmiastowej inicjalizacji, obiecując kompilatorowi że wartość zostanie przypisana przed pierwszym odczytem.

```dart
class SerwisAPI {
  // late — non-nullable, ale inicjalizowane później
  late String baseUrl;
  late int timeout;

  void konfiguruj(String url, {int timeoutSec = 30}) {
    baseUrl = url;
    timeout = timeoutSec;
  }

  void wykonajZapytanie() {
    // W tym momencie late zmienne MUSZĄ być już zainicjalizowane
    print('Zapytanie do: $baseUrl (timeout: ${timeout}s)');
  }
}

void main() {
  var serwis = SerwisAPI();
  serwis.konfiguruj('https://api.example.com', timeoutSec: 10);
  serwis.wykonajZapytanie();
}
// Oczekiwane wyjście:
// Zapytanie do: https://api.example.com (timeout: 10s)
```

Gdy `late` zmienna jest odczytana przed inicjalizacją, Dart rzuca `LateInitializationError`:

```dart
class Kontroller {
  late String nazwa; // zadeklarowana bez inicjalizacji
}

void main() {
  var ctrl = Kontroller();

  // Próba odczytu late zmiennej PRZED przypisaniem wartości
  try {
    print(ctrl.nazwa); // BOOM! Nie została jeszcze zainicjalizowana
  } on Error catch (e) {
    print('Błąd: $e');
    // LateInitializationError: Field 'nazwa' has not been initialized.
  }

  // Po przypisaniu — działa prawidłowo
  ctrl.nazwa = 'MainController';
  print('Nazwa: ${ctrl.nazwa}');
}
// Oczekiwane wyjście:
// Błąd: LateInitializationError: Field 'nazwa' has not been initialized.
// Nazwa: MainController
```

---

## Type promotion (promocja typów)

Dart automatycznie promuje typ zmiennej nullable do non-nullable po sprawdzeniu, że nie jest null. To eliminuje potrzebę wielokrotnego rzutowania.

```dart
void main() {
  Object? wartosc = 'Dart jest super';

  // Bez promotion — nie można użyć metod String
  // print(wartosc.length); // Błąd: 'length' nie jest zdefiniowane dla Object?

  // Po sprawdzeniu is — automatyczna promocja
  if (wartosc is String) {
    // Tu wartosc jest automatycznie String (nie Object?)
    print(wartosc.toUpperCase());  // DART JEST SUPER
    print(wartosc.length);         // 15
    print(wartosc.split(' '));     // [Dart, jest, super]
  }

  // Po sprawdzeniu != null — promocja do non-nullable
  int? liczba = 42;
  if (liczba != null) {
    // Tu liczba jest int (nie int?)
    var podwojona = liczba * 2; // OK — operacja na non-nullable int
    print(podwojona); // 84
  }
}
// Oczekiwane wyjście:
// DART JEST SUPER
// 15
// [Dart, jest, super]
// 84
```

Ograniczenia type promotion — nie działa w każdym kontekście:

```dart
class Kontener {
  String? _wartosc;

  void ustaw(String w) => _wartosc = w;

  void wypisz() {
    // Pola instancji NIE są promowane (mogą być zmienione z zewnątrz)
    // if (_wartosc != null) {
    //   print(_wartosc.length); // Błąd! Pole może być zmienione między sprawdzeniem a użyciem
    // }

    // Rozwiązanie: przypisanie do zmiennej lokalnej
    var lokalna = _wartosc;
    if (lokalna != null) {
      print(lokalna.length); // OK — zmienna lokalna jest bezpiecznie promowana
    }
  }
}

void main() {
  var k = Kontener();
  k.ustaw('Test');
  k.wypisz();
}
// Oczekiwane wyjście:
// 4
```

---

## Wzorce projektowe z null safety

Poniższy przykład prezentuje typowe wzorce kodu, które pomagają pisać bezpieczny kod z null safety:

```dart
// Wzorzec 1: Wczesny return (early return)
String formatujNazwe(String? imie, String? nazwisko) {
  if (imie == null && nazwisko == null) return 'Anonim';
  if (imie == null) return nazwisko!;
  if (nazwisko == null) return imie;
  return '$imie $nazwisko';
}

// Wzorzec 2: Operator ?. z ?? dla domyślnych
int dlugoscTekstu(String? tekst) {
  return tekst?.length ?? 0;
}

// Wzorzec 3: Collection if z null check
List<String> budujListeOpcji(String? opcja1, String? opcja2, String? opcja3) {
  return [
    if (opcja1 != null) opcja1,
    if (opcja2 != null) opcja2,
    if (opcja3 != null) opcja3,
  ];
}

void main() {
  print(formatujNazwe('Anna', 'Kowalska'));  // Anna Kowalska
  print(formatujNazwe(null, 'Kowalska'));     // Kowalska
  print(formatujNazwe(null, null));           // Anonim

  print(dlugoscTekstu('Hello'));  // 5
  print(dlugoscTekstu(null));     // 0

  print(budujListeOpcji('A', null, 'C')); // [A, C]
}
// Oczekiwane wyjście:
// Anna Kowalska
// Kowalska
// Anonim
// 5
// 0
// [A, C]
```

Wzorzec z `late final` — leniwa jednorazowa inicjalizacja:

```dart
class BazaDanych {
  // late final z inicjalizatorem — obliczone leniwie, raz
  late final String connectionString = _budujConnectionString();

  String _budujConnectionString() {
    print('Budowanie connection string...');
    // Symulacja kosztownej operacji
    return 'postgres://user:pass@localhost:5432/mydb';
  }
}

void main() {
  var db = BazaDanych();
  print('Obiekt utworzony'); // connection string jeszcze nie obliczony

  // Dopiero tu następuje inicjalizacja
  print(db.connectionString);
  // Drugie odwołanie — używa cache
  print(db.connectionString);
}
// Oczekiwane wyjście:
// Obiekt utworzony
// Budowanie connection string...
// postgres://user:pass@localhost:5432/mydb
// postgres://user:pass@localhost:5432/mydb
```

---

## Ćwiczenie 1

### Opis problemu

Napisz funkcję `bezpiecznyDostep` która przyjmuje `Map<String, dynamic>?` i klucz (`String`). Funkcja powinna bezpiecznie zwrócić wartość jako `String`, obsługując wszystkie przypadki null: mapa może być null, klucz może nie istnieć, wartość może być null. Użyj operatorów `?.`, `??` i łańcuchowych wywołań null-aware. Nie używaj operatora `!`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| mapa: `{'name': 'Dart', 'version': null}`, klucz: `'name'` | `Dart` |
| mapa: `{'name': 'Dart', 'version': null}`, klucz: `'version'` | `(brak danych)` |
| mapa: `{'name': 'Dart'}`, klucz: `'author'` | `(brak danych)` |
| mapa: `null`, klucz: `'name'` | `(brak danych)` |

### Wskazówki

1. Użyj `?.[]` do bezpiecznego dostępu do elementu mapy: `mapa?[klucz]`
2. Użyj `?.toString()` do bezpiecznej konwersji na String
3. Operator `??` na końcu łańcucha da wartość domyślną

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
String bezpiecznyDostep(Map<String, dynamic>? mapa, String klucz) {
  // ?[] bezpiecznie dostaje element, ?.toString() konwertuje, ?? daje domyślną
  return mapa?[klucz]?.toString() ?? '(brak danych)';
}

void main() {
  var dane = <String, dynamic>{'name': 'Dart', 'version': null};

  print(bezpiecznyDostep(dane, 'name'));      // Dart
  print(bezpiecznyDostep(dane, 'version'));   // (brak danych)
  print(bezpiecznyDostep(dane, 'author'));    // (brak danych)
  print(bezpiecznyDostep(null, 'name'));      // (brak danych)
}
```

</details>

---

## Ćwiczenie 2

### Opis problemu

Stwórz klasę `Profil` z opcjonalnymi polami: `imie` (`String?`), `email` (`String?`), `wiek` (`int?`), i `adres` jako zagnieżdżony obiekt `Adres?` (z polami `miasto` i `kodPocztowy`, oba `String?`). Napisz metodę `podsumowanie()` która zwraca sformatowany tekst używając wyłącznie operatorów null-aware (bez `if (x != null)`). Każde pole bez wartości powinno pokazywać tekst zastępczy.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| imie: `'Anna'`, email: `null`, wiek: `25`, adres: `Adres(miasto: 'Kraków', kodPocztowy: null)` | `Anna (25 lat)\nEmail: nie podano\nMiasto: Kraków\nKod: nie podano` |
| imie: `null`, email: `'jan@mail.com'`, wiek: `null`, adres: `null` | `Anonim\nEmail: jan@mail.com\nMiasto: nie podano\nKod: nie podano` |

### Wskazówki

1. Użyj `??` do podawania wartości zastępczych: `imie ?? 'Anonim'`
2. Użyj `?.` do bezpiecznego dostępu do pól zagnieżdżonego obiektu: `adres?.miasto`
3. Do formatowania wieku możesz użyć: `wiek != null ? '$wiek lat' : null` lub `wiek?.toString()`
4. Połącz `?.` i `??` w łańcuchy: `adres?.miasto ?? 'nie podano'`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
class Adres {
  final String? miasto;
  final String? kodPocztowy;
  Adres({this.miasto, this.kodPocztowy});
}

class Profil {
  final String? imie;
  final String? email;
  final int? wiek;
  final Adres? adres;

  Profil({this.imie, this.email, this.wiek, this.adres});

  String podsumowanie() {
    // Łańcuch operatorów null-aware bez if-ów
    var wiekTekst = wiek?.toString() ?? '?';
    var nazwaWyswietlana = imie ?? 'Anonim';
    var wiekFragment = wiek != null ? ' ($wiekTekst lat)' : '';

    return '$nazwaWyswietlana$wiekFragment\n'
        'Email: ${email ?? "nie podano"}\n'
        'Miasto: ${adres?.miasto ?? "nie podano"}\n'
        'Kod: ${adres?.kodPocztowy ?? "nie podano"}';
  }
}

void main() {
  var profil1 = Profil(
    imie: 'Anna',
    wiek: 25,
    adres: Adres(miasto: 'Kraków'),
  );
  print(profil1.podsumowanie());
  print('---');

  var profil2 = Profil(email: 'jan@mail.com');
  print(profil2.podsumowanie());
}
```

</details>

---

## Ćwiczenie 3

### Opis problemu

Napisz funkcję `przetworzDane(List<String?>? dane)` która:
1. Jeśli lista jest null, zwraca pustą listę
2. Filtruje wartości null z listy
3. Konwertuje pozostałe stringi na wielkie litery
4. Zwraca wynik jako `List<String>` (non-nullable)

Użyj kombinacji operatorów null-aware i metod kolekcji. Nie używaj `!`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `['dart', null, 'flutter', null, 'pub']` | `[DART, FLUTTER, PUB]` |
| `null` | `[]` |
| `[null, null]` | `[]` |
| `['hello']` | `[HELLO]` |

### Wskazówki

1. Użyj `??` aby obsłużyć null listę: `dane ?? []`
2. Metoda `whereType<String>()` filtruje elementy i zwraca tylko te danego typu (odrzuca null)
3. `.map((s) => s.toUpperCase())` konwertuje każdy element
4. `.toList()` konwertuje wynik na List

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
List<String> przetworzDane(List<String?>? dane) {
  return (dane ?? [])          // jeśli dane null → pusta lista
      .whereType<String>()     // filtruje null (zostawia tylko String, nie String?)
      .map((s) => s.toUpperCase()) // konwersja na wielkie litery
      .toList();               // wynik jako List<String>
}

void main() {
  print(przetworzDane(['dart', null, 'flutter', null, 'pub']));
  print(przetworzDane(null));
  print(przetworzDane([null, null]));
  print(przetworzDane(['hello']));
}
```

</details>

---

**Następny moduł:** [Typy generyczne](../02-types/01-generics.md)
**Poprzedni moduł:** [Operatory](02-operators.md)
