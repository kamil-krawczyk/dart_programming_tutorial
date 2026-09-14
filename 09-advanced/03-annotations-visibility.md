---
id: "9.3"
title: "Adnotacje metadanych, widoczność bibliotek i importy warunkowe"
difficulty: "advanced"
section: "09-advanced"
prerequisites:
  - "Klasy i konstruktory"
  - "Dziedziczenie"
  - "Rozszerzenia (extension methods i extension types)"
---

# 9.3 Adnotacje metadanych, widoczność bibliotek i importy warunkowe

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Dziedziczenie](../05-oop/02-inheritance.md), [Rozszerzenia (extension methods i extension types)](../09-advanced/01-extensions.md)
- **Cele nauki:**
  1. Stosować wbudowane adnotacje metadanych (`@override`, `@Deprecated`, `@pragma`) oraz definiować własne adnotacje
  2. Kontrolować widoczność importowanych symboli za pomocą kombinatorów `show` i `hide`
  3. Nadawać przestrzeniom nazw prefiksy importu przez `as` oraz eksponować symbole innych bibliotek przez `export`
  4. Dzielić bibliotekę na wiele plików za pomocą `part` i `part of`
  5. Wybierać implementację zależnie od dostępnej platformy przy użyciu importów warunkowych (`if (dart.library.io)`)

---

## Czym są metadane i adnotacje?

**Adnotacja** (metadana) to znacznik poprzedzony symbolem `@`, który dołączamy do deklaracji — klasy, metody, pola, parametru czy biblioteki. Adnotacje nie zmieniają działania kodu w czasie wykonania same z siebie; są to **dane o kodzie**, które mogą być odczytywane przez:

- kompilator i analizator (np. `@override` włącza dodatkowe sprawdzenia),
- narzędzia zewnętrzne i generatory kodu (przez refleksję lub biblioteki takie jak `source_gen`),
- programistów czytających kod (dokumentują intencję).

Adnotacją może być wywołanie konstruktora `const` lub odwołanie do stałej `const`. Dart udostępnia kilka wbudowanych adnotacji, a my możemy też tworzyć własne.

W drugiej części modułu przejdziemy do **widoczności bibliotek** — czyli sposobów kontrolowania, które symbole są importowane, jak je nazywamy i jak organizujemy bibliotekę w wielu plikach. Na koniec poznasz **importy warunkowe**, dzięki którym jeden pakiet może działać zarówno w przeglądarce, jak i na serwerze.

---

## Wbudowane adnotacje: @override, @Deprecated, @pragma

Najczęściej używana adnotacja to `@override`. Umieszczamy ją nad składową, która przesłania składową z klasy nadrzędnej lub interfejsu. Dzięki niej analizator zgłosi błąd, jeśli literówka sprawi, że metoda niczego nie przesłania.

Adnotacja `@Deprecated('komunikat')` oznacza element jako przestarzały. Domyślnie analizator zgłasza ostrzeżenie tylko wtedy, gdy przestarzały element jest używany **z innego pakietu** niż ten, w którym go zadeklarowano — użycie w tym samym pakiecie (np. w tym samym pliku, jak w przykładzie niżej) nie generuje ostrzeżenia, chyba że włączysz opcjonalną regułę lintera `deprecated_member_use_from_same_package`. (Istnieje też stała `@deprecated` bez komunikatu, ale zalecana jest forma z komunikatem.)

Poniższy przykład pokazuje `@override` przy przesłanianiu metody oraz `@Deprecated` na starej metodzie, którą chcemy wycofać.

```dart
class Repozytorium {
  // @Deprecated oznacza metodę jako przestarzałą. Ostrzeżenie analizatora
  // (z podanym komunikatem migracyjnym) pojawi się przy użyciu z INNEGO
  // pakietu — w tym samym pakiecie (jak w main() poniżej) domyślnie go nie ma.
  @Deprecated('Użyj metody zapisz() zamiast tej metody')
  void save() => print('stara metoda save()');

  void zapisz() => print('nowa metoda zapisz()');
}

class RepozytoriumPlikowe extends Repozytorium {
  // @override wymusza sprawdzenie, że metoda faktycznie coś przesłania.
  @override
  void zapisz() => print('zapis do pliku');
}

void main() {
  final repo = RepozytoriumPlikowe();
  repo.zapisz(); // wywołanie przesłoniętej metody

  // Wywołanie save() nadal działa. Ostrzeżenie "'save' is deprecated and
  // shouldn't be used." pojawiłoby się, gdyby ktoś wywołał save() z innego
  // pakietu — tutaj, w tym samym pliku, `dart analyze` nie zgłosi problemu.
  repo.save();
}
// Oczekiwane wyjście:
// zapis do pliku
// stara metoda save()
```

Adnotacja `@pragma` przekazuje wskazówki kompilatorowi. Popularny przykład to `@pragma('vm:entry-point')`, który zapobiega usunięciu (tree-shaking) elementu wywoływanego np. z natywnego kodu lub z isolate'a. Poniżej używamy jej na funkcji, która ma być punktem wejścia isolate'a.

```dart
// @pragma('vm:entry-point') informuje kompilator, aby nie usuwał tej funkcji
// podczas tree-shakingu, nawet jeśli nie widzi jej bezpośredniego wywołania.
@pragma('vm:entry-point')
void punktWejscia(String wiadomosc) {
  print('Isolate otrzymał: $wiadomosc');
}

void main() {
  // W realnym kodzie tę funkcję uruchomiłby Isolate.spawn; tutaj wywołujemy
  // ją bezpośrednio, aby zademonstrować działanie.
  punktWejscia('start');
}
// Oczekiwane wyjście:
// Isolate otrzymał: start
```

---

## Własne adnotacje

Aby stworzyć własną adnotację, definiujemy klasę z konstruktorem `const` i używamy jej instancji jako metadanej. Same adnotacje nie wykonują żadnej logiki — nadają jedynie znaczenie, które odczytają narzędzia lub kod korzystający z refleksji (`dart:mirrors`) czy generacji kodu.

Poniższy przykład definiuje adnotację `Todo` opisującą zadanie do wykonania oraz prostą adnotację-znacznik `experymentalne`. Dołączamy je do klasy i metody, demonstrując składnię z argumentami oraz bez.

```dart
// Własna adnotacja z polami — konstruktor MUSI być const.
class Todo {
  final String opis;
  final String autor;
  const Todo(this.opis, {required this.autor});
}

// Adnotacja-znacznik bez pól, użyta jako stała const.
class Experymentalne {
  const Experymentalne();
}

const experymentalne = Experymentalne();

// Dołączamy adnotacje do deklaracji — nazwa poprzedzona @.
@Todo('Dodać walidację danych wejściowych', autor: 'Ala')
class Formularz {
  @experymentalne
  void wyslij() => print('wysłano formularz');
}

void main() {
  // Adnotacje nie zmieniają działania w czasie wykonania — kod działa normalnie.
  Formularz().wyslij();
}
// Oczekiwane wyjście:
// wysłano formularz
```

> **Uwaga:** Argumenty adnotacji muszą być wyrażeniami stałymi (`const`). Nie można przekazać do adnotacji wartości wyliczanej w czasie wykonania.

---

## Widoczność bibliotek: show i hide

Domyślnie `import 'biblioteka.dart';` sprowadza **wszystkie** publiczne symbole danej biblioteki. Aby ograniczyć zakres importu, używamy **kombinatorów**:

- `show` — importuje **tylko** wymienione symbole,
- `hide` — importuje wszystko **oprócz** wymienionych symboli.

Kombinatory pomagają unikać kolizji nazw i wyraźnie dokumentują, czego z danej biblioteki faktycznie używamy. Poniższy przykład importuje z `dart:math` wyłącznie to, co potrzebne, a z `dart:collection` ukrywa jeden typ.

```dart
// show: importujemy z dart:math TYLKO te trzy symbole.
import 'dart:math' show pi, sqrt, Random;

// hide: importujemy wszystko z dart:collection OPRÓCZ HashMap.
import 'dart:collection' hide HashMap;

void main() {
  print(pi);            // dostępne dzięki show
  print(sqrt(16));      // dostępne dzięki show
  print(Random(1).nextInt(100)); // dostępne dzięki show

  // Queue jest dostępne, bo hide ukryło tylko HashMap
  final kolejka = Queue<int>();
  kolejka.addAll([1, 2, 3]);
  print(kolejka.first);
}
// Oczekiwane wyjście:
// 3.141592653589793
// 4.0
// 4
// 1
```

> **Uwaga:** Gdybyśmy powyżej użyli `min` lub `max`, kompilator zgłosiłby błąd, ponieważ `show pi, sqrt, Random` nie sprowadził tych nazw.

---

## Prefiksy importu: as

Kiedy dwie biblioteki definiują symbol o tej samej nazwie, powstaje kolizja. Rozwiązuje ją słowo kluczowe `as`, które nadaje importowanej bibliotece **prefiks** (przestrzeń nazw). Wszystkie jej symbole są wtedy dostępne pod tym prefiksem, np. `mat.max(...)`.

Poniższy przykład importuje `dart:math` z prefiksem `mat`, dzięki czemu lokalna funkcja `max` nie koliduje z funkcją `max` z biblioteki.

```dart
// as nadaje bibliotece prefiks; jej symbole są dostępne jako mat.<nazwa>.
import 'dart:math' as mat;

// Lokalna funkcja o nazwie kolidującej z dart:math.max
String max(String a, String b) => a.length >= b.length ? a : b;

void main() {
  // Wersja z prefiksem — funkcja max z dart:math (większa liczba)
  print(mat.max(3, 9)); // 9
  print(mat.pi);        // dostęp do stałej przez prefiks

  // Wersja bez prefiksu — nasza lokalna funkcja na Stringach
  print(max('kot', 'słoń')); // 'słoń' (dłuższy wyraz)
}
// Oczekiwane wyjście:
// 9
// 3.141592653589793
// słoń
```

Prefiks można łączyć z `show`/`hide`, np. `import 'pakiet.dart' as p show A, B;`. Warto też wiedzieć o `import '...' deferred as ...;`, które opóźnia ładowanie biblioteki do pierwszego użycia (przydatne przy dużych zależnościach w aplikacjach webowych).

---

## Podział biblioteki na pliki: part i part of

Bardzo duże biblioteki można rozdzielić na wiele plików, zachowując je jako **jedną bibliotekę** o wspólnej przestrzeni nazw. Służą do tego dyrektywy:

- `part 'plik.dart';` — w pliku głównym: dołącza plik składowy,
- `part of 'plik_glowny.dart';` — w pliku składowym: deklaruje przynależność do biblioteki głównej.

Kluczowa cecha: wszystkie pliki złączone przez `part` **współdzielą przestrzeń nazw** — mają wzajemny dostęp nawet do symboli prywatnych (z podkreśleniem `_`). Importy deklaruje się tylko w pliku głównym; pliki `part` z nich korzystają.

> **Uwaga:** Poniższy przykład rozkłada się na dwa pliki i nie jest uruchamialny jako pojedynczy fragment — ilustruje strukturę biblioteki wieloplikowej.

Plik główny `geometria.dart`:

```dart
// Plik główny biblioteki. Importy deklarujemy tutaj — pliki part je dziedziczą.
import 'dart:math';

// part dołącza plik składowy do tej samej biblioteki.
part 'ksztalty.dart';

// Symbol prywatny widoczny również w pliku part (wspólna przestrzeń nazw).
const _dokladnosc = 2;

void main() {
  final k = Kolo(3);
  // pole() jest zdefiniowane w pliku part, a mimo to jest tu dostępne.
  print(k.pole().toStringAsFixed(_dokladnosc));
}
```

Plik składowy `ksztalty.dart`:

```dart
// part of wskazuje, do której biblioteki należy ten plik.
part of 'geometria.dart';

class Kolo {
  final double promien;
  Kolo(this.promien);

  // Korzystamy z pi (import z pliku głównego) — bez własnego importu.
  double pole() => pi * promien * promien;
}
```

Gdyby oba pliki istniały obok siebie, `dart run geometria.dart` wypisałoby `28.27`.

> **Wskazówka:** W nowoczesnym kodzie Dart `part`/`part of` stosuje się głównie z generatorami kodu (pliki `*.g.dart`). Do zwykłego dzielenia kodu na moduły preferuje się osobne biblioteki łączone przez `import`/`export`.

---

## Eksponowanie symboli: export

Dyrektywa `export 'plik.dart';` sprawia, że symbole z innej biblioteki stają się częścią **publicznego API** bieżącej biblioteki. Dzięki temu autor pakietu może zebrać wiele wewnętrznych plików za jednym „plikiem-fasadą", a użytkownik importuje tylko jeden plik.

`export` również współpracuje z `show`/`hide`, pozwalając wyeksponować tylko wybraną część.

> **Uwaga:** Poniższy przykład obejmuje dwa pliki; plik-fasada re-eksportuje symbole pliku wewnętrznego.

Plik wewnętrzny `modele.dart`:

```dart
class Uzytkownik {
  final String nazwa;
  Uzytkownik(this.nazwa);
}

class Sekret {
  final String wartosc;
  Sekret(this.wartosc);
}
```

Plik-fasada `api.dart`:

```dart
// export udostępnia symbole innej biblioteki jako własne publiczne API.
// show ogranicza eksport do wybranych typów — Sekret pozostaje ukryty.
export 'modele.dart' show Uzytkownik;
```

Plik konsumenta `main.dart`:

```dart
// Importujemy tylko fasadę — Uzytkownik jest dostępny, Sekret NIE.
import 'api.dart';

void main() {
  final u = Uzytkownik('Ala');
  print('Użytkownik: ${u.nazwa}');
  // final s = Sekret('x'); // BŁĄD: Sekret nie został wyeksportowany
}
// Oczekiwane wyjście (przy komplecie plików):
// Użytkownik: Ala
```

---

## Importy warunkowe

**Import warunkowy** pozwala wybrać różne implementacje tej samej biblioteki zależnie od dostępności platformowych bibliotek Dart. Składnia:

```dart
import 'implementacja_domyslna.dart'
    if (dart.library.io) 'implementacja_io.dart'
    if (dart.library.html) 'implementacja_web.dart';
```

Kompilator wybiera **pierwszy** wariant, którego warunek jest spełniony:

- `dart.library.io` — dostępne, gdy program działa na maszynie wirtualnej Dart (aplikacje serwerowe, CLI, Flutter na urządzeniach),
- `dart.library.html` — dostępne przy kompilacji do przeglądarki.

Jeśli żaden warunek nie jest spełniony, używana jest biblioteka domyślna (pierwsza z listy). Wszystkie warianty muszą udostępniać **taki sam interfejs publiczny** (te same nazwy klas i funkcji), aby kod korzystający z importu był niezależny od platformy.

> **Uwaga:** Poniższy przykład obejmuje trzy pliki. Kod konsumenta jest przenośny — nie wie, którą implementację wybierze kompilator.

Interfejs domyślny `platforma_stub.dart`:

```dart
// Wariant domyślny: awaryjna implementacja, gdy żadna platforma nie pasuje.
String nazwaPlatformy() => 'nieznana';
```

Wariant dla VM `platforma_io.dart`:

```dart
// Ten plik zostanie wybrany, gdy dostępne jest dart.library.io.
import 'dart:io';

String nazwaPlatformy() => Platform.operatingSystem; // np. 'linux', 'macos'
```

Wariant dla przeglądarki `platforma_web.dart`:

```dart
// Ten plik zostanie wybrany przy kompilacji do przeglądarki.
String nazwaPlatformy() => 'web';
```

Kod konsumenta `main.dart`:

```dart
// Import warunkowy: kompilator wybierze właściwy wariant automatycznie.
import 'platforma_stub.dart'
    if (dart.library.io) 'platforma_io.dart'
    if (dart.library.html) 'platforma_web.dart';

void main() {
  // Kod jest przenośny — nie wie, która implementacja została użyta.
  print('Działam na platformie: ${nazwaPlatformy()}');
}
// Oczekiwane wyjście przy uruchomieniu na Dart VM (linux):
// Działam na platformie: linux
```

---

## Ćwiczenie 1: Własna adnotacja walidacji

### Opis problemu

Zdefiniuj własną adnotację `Waliduj`, która przechowuje dwa pola: `maksDlugosc` (typu `int`) oraz `wymagane` (typu `bool`, domyślnie `true`). Adnotacja musi mieć konstruktor `const`. Następnie utwórz klasę `Rejestracja` z polem `login` i **udekoruj to pole** adnotacją `@Waliduj(maksDlugosc: 20)`. W funkcji `main` po prostu utwórz obiekt `Rejestracja` i wypisz wartość jego pola, aby pokazać, że adnotacja nie wpływa na działanie w czasie wykonania.

**Poziom trudności:** basic

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `Rejestracja('ala').login` | `ala` |
| `Rejestracja('bogdan123').login` | `bogdan123` |

### Wskazówki

1. Konstruktor adnotacji musi być `const`, a pola `final`.
2. Użyj parametrów nazwanych: `const Waliduj({required this.maksDlugosc, this.wymagane = true});`.
3. Adnotację umieść bezpośrednio nad polem `login` w klasie `Rejestracja`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// Własna adnotacja z polami — konstruktor const i pola final.
class Waliduj {
  final int maksDlugosc;
  final bool wymagane;
  const Waliduj({required this.maksDlugosc, this.wymagane = true});
}

class Rejestracja {
  // Dekorujemy pole adnotacją z argumentem nazwanym.
  @Waliduj(maksDlugosc: 20)
  final String login;

  Rejestracja(this.login);
}

void main() {
  // Adnotacja jest metadaną — nie zmienia zachowania w czasie wykonania.
  print(Rejestracja('ala').login);
  print(Rejestracja('bogdan123').login);
}
// Oczekiwane wyjście:
// ala
// bogdan123
```

</details>

---

## Ćwiczenie 2: Usuwanie kolizji nazw przez as i show

### Opis problemu

Masz lokalną funkcję `String log(String s) => 'LOG: $s';`, która koliduje z funkcją `log` (logarytm naturalny) z `dart:math`. Zaimportuj `dart:math` tak, aby:

1. dostępna była stała `e` bezpośrednio (bez prefiksu),
2. logarytm z `dart:math` był dostępny pod prefiksem `mat`,
3. Twoja lokalna funkcja `log` działała bez konfliktu.

W `main` wypisz: wynik lokalnej funkcji `log('test')`, wartość `e` oraz `mat.log(e)` (logarytm naturalny z `e`, który wynosi `1.0`).

**Poziom trudności:** intermediate

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `log('test')` | `LOG: test` |
| `mat.log(e)` | `1.0` |

### Wskazówki

1. Możesz użyć dwóch dyrektyw importu tej samej biblioteki: jednej z `show e`, drugiej z `as mat`.
2. Alternatywnie jedna dyrektywa `import 'dart:math' as mat;` plus osobny `import 'dart:math' show e;`.
3. Prefiks `mat` sprawia, że `log` z biblioteki jest dostępny tylko jako `mat.log`, więc nie koliduje z lokalną funkcją.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// Jeden import udostępnia tylko stałą e bez prefiksu...
import 'dart:math' show e;
// ...drugi udostępnia całą bibliotekę pod prefiksem mat.
import 'dart:math' as mat;

// Lokalna funkcja log — nie koliduje, bo log z biblioteki jest pod mat.
String log(String s) => 'LOG: $s';

void main() {
  print(log('test'));        // lokalna funkcja
  print(e);                  // stała e dostępna bez prefiksu (show e)
  print(mat.log(e));         // logarytm naturalny z e = 1.0
}
// Oczekiwane wyjście:
// LOG: test
// 2.718281828459045
// 1.0
```

</details>

---

**Poprzedni moduł:** [Rekordy i dopasowywanie wzorców (Dart 3)](../09-advanced/02-records-patterns.md)
**Następny moduł:** [Obsługa błędów i wyjątków](../09-advanced/04-error-handling.md)
