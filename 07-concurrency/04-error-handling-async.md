---
id: "7.4"
title: "Obsługa błędów w kodzie asynchronicznym"
difficulty: "advanced"
section: "07-concurrency"
prerequisites:
  - "Futures i async/await"
  - "Streams"
---

# 7.4 Obsługa błędów w kodzie asynchronicznym

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Futures i async/await](../07-concurrency/01-async-await.md), [Streams](../07-concurrency/02-streams.md)
- **Cele nauki:**
  1. Rozumieć mechanizmy obsługi błędów w Dart (`throw`, `rethrow`, `try`/`catch`/`finally`, klauzula `on`) oraz różnicę między hierarchiami `Exception` i `Error`
  2. Poprawnie obsługiwać błędy synchroniczne oraz asynchroniczne, w tym za pomocą `try`/`catch` wokół oczekiwanych (`await`) Futures
  3. Projektować własną hierarchię wyjątków z polami kontekstowymi i przesłoniętym `toString`, aby przekazywać czytelne informacje diagnostyczne
  4. Kontrolować propagację błędów przez granice asynchroniczne: nieobsłużone błędy Future, przechwytywanie strefowe (`runZonedGuarded`) oraz przekazywanie błędów w strumieniach (Stream)

---

## Wprowadzenie: dwa światy błędów

W Dart błąd może zostać zgłoszony w dwóch kontekstach: **synchronicznym** (natychmiast, w bieżącym stosie wywołań) oraz **asynchronicznym** (później, gdy zakończy się `Future` lub gdy `Stream` wyemituje zdarzenie błędu). Kluczowa trudność polega na tym, że mechanizm `try`/`catch` przechwytuje wyłącznie błędy zgłoszone **synchronicznie w bloku `try`**. Błąd, który pojawia się w niezakończonym jeszcze `Future`, nie zostanie przez taki blok złapany, chyba że użyjemy `await`.

Ten moduł pokazuje, jak działają podstawowe konstrukcje obsługi błędów, jak zaprojektować własną hierarchię wyjątków i — co najważniejsze — jak błędy przemieszczają się przez granice asynchroniczne.

---

## Podstawy: `throw`, `rethrow`, `try`/`catch`/`finally`, klauzula `on`

Błąd zgłasza się słowem kluczowym `throw`. Można rzucić dowolny obiekt, ale w praktyce rzuca się instancje typów implementujących `Exception` lub rozszerzających `Error`. Blok `try` obejmuje kod, który może zawieść; `catch` przechwytuje rzucony obiekt; `finally` wykonuje się zawsze — niezależnie od tego, czy wystąpił błąd.

Poniższy przykład pokazuje pełną strukturę `try`/`on`/`catch`/`finally` oraz przechwytywanie po typie za pomocą klauzuli `on`.

```dart
// Klauzula on filtruje wyjątki po typie; catch wiąże obiekt błędu i stack trace
double podziel(int a, int b) {
  if (b == 0) {
    throw ArgumentError('Dzielnik nie może być zerem'); // zgłoszenie błędu
  }
  return a / b;
}

void main() {
  try {
    print(podziel(10, 2)); // 5.0
    print(podziel(1, 0)); // rzuci ArgumentError
  } on ArgumentError catch (e) {
    // on <Typ> catch (e) — łapie tylko wskazany typ wyjątku
    print('Złapano ArgumentError: ${e.message}');
  } finally {
    print('Blok finally wykonuje się zawsze');
  }
}
// Oczekiwane wyjście:
// 5.0
// Złapano ArgumentError: Dzielnik nie może być zerem
// Blok finally wykonuje się zawsze
```

Słowo kluczowe `rethrow` pozwala częściowo obsłużyć błąd (np. zalogować go), a następnie przekazać go dalej **z zachowaniem oryginalnego stack trace**. Jest to lepsze niż `throw e`, które gubi pierwotny ślad stosu.

```dart
void operacjaNiskopoziomowa() {
  throw StateError('Nieprawidłowy stan wewnętrzny');
}

void operacjaPosrednia() {
  try {
    operacjaNiskopoziomowa();
  } on StateError catch (e) {
    print('Logowanie na poziomie pośrednim: ${e.message}');
    rethrow; // przekazuje ten sam błąd wyżej, zachowując oryginalny stack trace
  }
}

void main() {
  try {
    operacjaPosrednia();
  } catch (e) {
    // catch bez on łapie każdy rzucony obiekt
    print('Ostateczna obsługa: $e');
  }
}
// Oczekiwane wyjście:
// Logowanie na poziomie pośrednim: Nieprawidłowy stan wewnętrzny
// Ostateczna obsługa: Bad state: Nieprawidłowy stan wewnętrzny
```

---

## Hierarchia `Exception` vs `Error`

Dart rozróżnia dwie koncepcyjne rodziny błędów:

- **`Exception`** — reprezentuje sytuacje **spodziewane i możliwe do obsłużenia** w czasie działania (np. nieprawidłowe dane wejściowe od użytkownika, brak pliku, błąd sieci). Kod aplikacji powinien je przechwytywać i obsługiwać.
- **`Error`** — reprezentuje **błędy programisty**, których zwykrycie oznacza, że kod jest błędny (np. `RangeError`, `StateError`, `TypeError`, `AssertionError`). Zamiast je łapać, należy naprawić kod. Przechwytywanie `Error` jest zwykle antywzorcem.

Poniższy przykład pokazuje typowe reprezentacje obu rodzin i to, jak można je rozróżnić w bloku `catch`.

```dart
// Exception -> sytuacja obsługiwalna; Error -> błąd programisty
void przetworzWiek(int wiek) {
  if (wiek < 0) {
    // FormatException to Exception — spodziewany, obsługiwalny błąd danych
    throw const FormatException('Wiek nie może być ujemny');
  }
  final lista = [1, 2, 3];
  print(lista[wiek]); // dla wiek >= 3 rzuci RangeError (Error)
}

void main() {
  for (final w in [1, -1, 10]) {
    try {
      przetworzWiek(w);
    } on Exception catch (e) {
      print('Obsługiwalny wyjątek: $e');
    } on Error catch (e) {
      print('Błąd programisty (napraw kod!): ${e.runtimeType}');
    }
  }
}
// Oczekiwane wyjście:
// 2
// Obsługiwalny wyjątek: FormatException: Wiek nie może być ujemny
// Błąd programisty (napraw kod!): RangeError
```

Drugi przykład pokazuje, że każdy rzucony błąd niesie ze sobą `StackTrace`, dostępny jako drugi argument `catch`. Ślad stosu jest nieoceniony przy diagnozowaniu problemów.

```dart
void warstwaA() => warstwaB();
void warstwaB() => throw Exception('Awaria w warstwie B');

void main() {
  try {
    warstwaA();
  } catch (e, stackTrace) {
    // Drugi parametr catch wiąże StackTrace w momencie rzucenia błędu
    print('Błąd: $e');
    final pierwszaLinia = stackTrace.toString().split('\n').first;
    print('Pierwsza ramka stosu zawiera "warstwaB": '
        '${pierwszaLinia.contains('warstwaB')}');
  }
}
// Oczekiwane wyjście:
// Błąd: Exception: Awaria w warstwie B
// Pierwsza ramka stosu zawiera "warstwaB": true
```

---

## Obsługa synchroniczna vs asynchroniczna

### Obsługa synchroniczna

W kodzie synchronicznym błąd propaguje natychmiast w górę stosu wywołań, aż napotka pasujący `catch`. Poniższy przykład parsuje liczby i obsługuje niepoprawny format synchronicznie.

```dart
// Błąd synchroniczny propaguje natychmiast w bieżącym stosie wywołań
int parsujLiczbe(String tekst) {
  final wynik = int.tryParse(tekst);
  if (wynik == null) {
    throw FormatException('Nie można sparsować liczby', tekst);
  }
  return wynik;
}

void main() {
  for (final s in ['42', 'abc']) {
    try {
      print('Sparsowano: ${parsujLiczbe(s)}');
    } on FormatException catch (e) {
      print('Błąd formatu dla "${e.source}"');
    }
  }
}
// Oczekiwane wyjście:
// Sparsowano: 42
// Błąd formatu dla "abc"
```

### Obsługa asynchroniczna z `try`/`catch` wokół `await`

Gdy funkcja jest oznaczona `async`, a błędna operacja jest **oczekiwana** przez `await`, zwykły `try`/`catch` działa tak samo intuicyjnie jak dla kodu synchronicznego. To zalecany sposób obsługi błędów asynchronicznych.

```dart
import 'dart:async';

// Symulacja operacji sieciowej, która po chwili kończy się błędem
Future<String> pobierzDane(String url) async {
  await Future<void>.delayed(const Duration(milliseconds: 10));
  if (url.isEmpty) {
    throw Exception('Pusty adres URL');
  }
  return 'dane z $url';
}

Future<void> main() async {
  try {
    final ok = await pobierzDane('https://example.com');
    print(ok);
    // await sprawia, że błąd z Future jest rzucany w tym bloku try
    final zle = await pobierzDane('');
    print(zle);
  } catch (e) {
    print('Złapano błąd asynchroniczny: $e');
  } finally {
    print('Zakończono próbę pobierania');
  }
}
// Oczekiwane wyjście:
// dane z https://example.com
// Złapano błąd asynchroniczny: Exception: Pusty adres URL
// Zakończono próbę pobierania
```

Drugi przykład kontrastuje `await` z podejściem opartym na `Future.catchError`, które przechwytuje błąd bez `await` w łańcuchu wywołań.

```dart
import 'dart:async';

Future<int> ryzykownaOperacja(bool zawiedz) async {
  await Future<void>.delayed(const Duration(milliseconds: 5));
  if (zawiedz) {
    throw StateError('Operacja zawiodła');
  }
  return 100;
}

Future<void> main() async {
  // catchError obsługuje błąd Future bez używania await/try-catch
  final wynik = await ryzykownaOperacja(true).catchError((Object e) {
    print('catchError przechwycił: $e');
    return -1; // wartość zastępcza zwracana po błędzie
  });
  print('Wynik po obsłudze: $wynik');
}
// Oczekiwane wyjście:
// catchError przechwycił: Bad state: Operacja zawiodła
// Wynik po obsłudze: -1
```

---

## Własna hierarchia wyjątków (3+ klasy)

W realnych aplikacjach warto definiować **własne typy wyjątków**, aby przekazywać kontekst błędu (np. kod HTTP, nazwę pola, identyfikator zasobu) i umożliwić selektywne przechwytywanie po typie. Dobra hierarchia składa się z klasy bazowej i wyspecjalizowanych podklas, z których każda niesie własne pola kontekstowe i przesłania `toString`.

Poniższy przykład definiuje bazowy `WyjatekAplikacji` oraz dwie wyspecjalizowane podklasy: `WyjatekSieci` (z kodem statusu) i `WyjatekWalidacji` (z nazwą pola i regułą). To trzyklasowa hierarchia zgodna z wymaganiem obsługi błędów.

```dart
// Klasa bazowa hierarchii — wspólny kontrakt dla wszystkich wyjątków aplikacji
abstract class WyjatekAplikacji implements Exception {
  final String komunikat;
  const WyjatekAplikacji(this.komunikat);

  @override
  String toString() => 'WyjatekAplikacji: $komunikat';
}

// Podklasa 1 — dodaje pole kontekstowe: kod statusu HTTP
class WyjatekSieci extends WyjatekAplikacji {
  final int kodStatusu;
  const WyjatekSieci(super.komunikat, this.kodStatusu);

  @override
  String toString() => 'WyjatekSieci($kodStatusu): $komunikat';
}

// Podklasa 2 — dodaje pola kontekstowe: nazwa pola i naruszona reguła
class WyjatekWalidacji extends WyjatekAplikacji {
  final String pole;
  final String regula;
  const WyjatekWalidacji(super.komunikat, this.pole, this.regula);

  @override
  String toString() =>
      'WyjatekWalidacji[pole=$pole, regula=$regula]: $komunikat';
}

void main() {
  final bledy = <WyjatekAplikacji>[
    const WyjatekSieci('Brak odpowiedzi serwera', 503),
    const WyjatekWalidacji('Nieprawidłowy email', 'email', 'format'),
  ];
  for (final b in bledy) {
    print(b); // korzysta z przesłoniętego toString każdej podklasy
  }
}
// Oczekiwane wyjście:
// WyjatekSieci(503): Brak odpowiedzi serwera
// WyjatekWalidacji[pole=email, regula=format]: Nieprawidłowy email
```

Drugi przykład pokazuje, jak hierarchia umożliwia **selektywne przechwytywanie**: możemy złapać konkretną podklasę (`WyjatekSieci`) lub cały rodzaj przez typ bazowy (`WyjatekAplikacji`).

```dart
abstract class WyjatekAplikacji implements Exception {
  final String komunikat;
  const WyjatekAplikacji(this.komunikat);
  @override
  String toString() => 'WyjatekAplikacji: $komunikat';
}

class WyjatekSieci extends WyjatekAplikacji {
  final int kodStatusu;
  const WyjatekSieci(super.komunikat, this.kodStatusu);
  @override
  String toString() => 'WyjatekSieci($kodStatusu): $komunikat';
}

class WyjatekWalidacji extends WyjatekAplikacji {
  final String pole;
  final String regula;
  const WyjatekWalidacji(super.komunikat, this.pole, this.regula);
  @override
  String toString() => 'WyjatekWalidacji[$pole/$regula]: $komunikat';
}

void obsluz(WyjatekAplikacji e) {
  try {
    throw e;
  } on WyjatekSieci catch (sieci) {
    // Najbardziej szczegółowa klauzula on musi być pierwsza
    print('Ponów żądanie — status ${sieci.kodStatusu}');
  } on WyjatekAplikacji catch (app) {
    // Łapie każdy inny wyjątek aplikacji (w tym WyjatekWalidacji)
    print('Ogólna obsługa: $app');
  }
}

void main() {
  obsluz(const WyjatekSieci('Timeout', 504));
  obsluz(const WyjatekWalidacji('Za krótkie hasło', 'haslo', 'min-8'));
}
// Oczekiwane wyjście:
// Ponów żądanie — status 504
// Ogólna obsługa: WyjatekWalidacji[haslo/min-8]: Za krótkie hasło
```

---

## Propagacja błędów przez granice asynchroniczne

To najtrudniejsza część obsługi błędów asynchronicznych. Błąd, który powstaje w `Future` lub `Stream`, może "wyciec" poza zasięg zwykłego `try`/`catch`, jeśli nie zadbamy o odpowiedni mechanizm.

### Nieobsłużone błędy Future

Jeśli `Future` zakończy się błędem, a nigdzie nie dodano do niego obsługi błędu (ani `await` w bloku `try`, ani `.catchError`, ani `.then(onError:)`), błąd staje się **nieobsłużony** i trafia do bieżącej strefy (Zone) jako błąd niezłapany. Poniższy przykład pokazuje częsty błąd: brak `await` sprawia, że `try`/`catch` **nie** przechwytuje błędu.

```dart
import 'dart:async';

Future<void> zawodzi() async {
  await Future<void>.delayed(const Duration(milliseconds: 5));
  throw Exception('Błąd w Future');
}

Future<void> main() async {
  // PUŁAPKA: brak await sprawia, że błąd NIE zostanie złapany przez ten try.
  // Błąd Future stałby się nieobsłużony; tutaj dołączamy .catchError, aby
  // pokazać, że przechwytuje go dopiero handler Future, a nie otaczający catch.
  try {
    zawodzi().catchError(
      (Object e) => print('Handler Future złapał (nie try/catch): $e'),
    );
  } catch (e) {
    print('Ten catch NIE zadziała: $e'); // nigdy się nie wykona
  }
  print('Blok try zakończony — błąd Future jeszcze nie wystąpił');

  // POPRAWNIE: await sprawia, że błąd wraca do bloku try
  try {
    await zawodzi(); // teraz błąd propaguje do tego catch
  } catch (e) {
    print('Poprawnie złapano: $e');
  }
}
// Oczekiwane wyjście:
// Blok try zakończony — błąd Future jeszcze nie wystąpił
// Handler Future złapał (nie try/catch): Exception: Błąd w Future
// Poprawnie złapano: Exception: Błąd w Future
```

### Przechwytywanie strefowe: `runZonedGuarded`

Aby przechwycić błędy, które w innym wypadku byłyby nieobsłużone (np. z zapomnianych `await` czy z callbacków timerów), używa się `runZonedGuarded`. Uruchamia ono kod w osobnej **strefie (Zone)** i przekierowuje wszystkie niezłapane błędy asynchroniczne do jednego handlera. To fundament globalnej obsługi błędów w aplikacjach.

```dart
import 'dart:async';

Future<void> zadanieWTle() async {
  await Future<void>.delayed(const Duration(milliseconds: 10));
  throw StateError('Awaria zadania w tle');
}

void main() {
  // runZonedGuarded przechwytuje niezłapane błędy asynchroniczne w strefie
  runZonedGuarded(
    () {
      zadanieWTle(); // brak await — normalnie błąd byłby nieobsłużony
      print('Zadanie w tle uruchomione');
    },
    (Object blad, StackTrace slad) {
      // Ten handler łapie błąd, mimo braku await w kodzie strefy
      print('Strefa przechwyciła: $blad');
    },
  );
}
// Oczekiwane wyjście:
// Zadanie w tle uruchomione
// Strefa przechwyciła: Bad state: Awaria zadania w tle
```

### Przekazywanie błędów w strumieniach (Stream)

Strumienie przekazują błędy jako osobny rodzaj zdarzenia (obok zdarzeń z danymi). Subskrybent obsługuje je przez parametr `onError` metody `listen` lub — w pętli `await for` — przez otaczający `try`/`catch`. Poniższy przykład pokazuje strumień emitujący dane i błąd, obsłużony przez `onError`.

```dart
import 'dart:async';

// Strumień emitujący dwie wartości, następnie błąd, następnie kolejną wartość
Stream<int> licznikZBledem() async* {
  yield 1;
  yield 2;
  yield* Stream<int>.error(Exception('Błąd w strumieniu'));
  yield 3; // po błędzie z addError strumień może kontynuować
}

Future<void> main() async {
  final completer = Completer<void>();
  licznikZBledem().listen(
    (wartosc) => print('Dane: $wartosc'),
    onError: (Object e) => print('onError: $e'), // przechwytuje zdarzenie błędu
    onDone: () {
      print('Strumień zakończony');
      completer.complete();
    },
  );
  await completer.future;
}
// Oczekiwane wyjście:
// Dane: 1
// Dane: 2
// onError: Exception: Błąd w strumieniu
// Dane: 3
// Strumień zakończony
```

Drugi przykład pokazuje obsługę błędów strumienia w pętli `await for`. Tu błąd propaguje jak wyjątek synchroniczny do otaczającego `try`/`catch`, przerywając iterację.

```dart
import 'dart:async';

Stream<int> daneCzujnika() async* {
  yield 10;
  yield 20;
  throw Exception('Utrata sygnału czujnika'); // przerywa strumień
}

Future<void> main() async {
  try {
    // await for propaguje błąd strumienia jak zwykły wyjątek
    await for (final odczyt in daneCzujnika()) {
      print('Odczyt: $odczyt');
    }
  } catch (e) {
    print('Przerwano odczyt: $e');
  }
  print('Kontynuacja po obsłudze błędu strumienia');
}
// Oczekiwane wyjście:
// Odczyt: 10
// Odczyt: 20
// Przerwano odczyt: Exception: Utrata sygnału czujnika
// Kontynuacja po obsłudze błędu strumienia
```

Trzeci przykład pokazuje transformację błędów w strumieniu za pomocą `handleError`, która pozwala przechwycić i przekształcić błąd bez przerywania subskrypcji.

```dart
import 'dart:async';

Future<void> main() async {
  final kontroler = StreamController<int>();

  // handleError przechwytuje błędy i pozwala je zignorować lub przekształcić
  final subskrypcja = kontroler.stream.handleError((Object e) {
    print('handleError: zignorowano $e');
  }).listen((wartosc) => print('Wartość: $wartosc'));

  kontroler
    ..add(1)
    ..addError(Exception('przejściowy błąd')) // przekazany do handleError
    ..add(2);
  await kontroler.close();
  await subskrypcja.cancel();
}
// Oczekiwane wyjście:
// Wartość: 1
// handleError: zignorowano Exception: przejściowy błąd
// Wartość: 2
```

---

## Ćwiczenie 1: Strategia obsługi błędów dla aplikacji trójwarstwowej (advanced)

### Opis problemu

Zaprojektuj strategię obsługi błędów dla aplikacji o **trzech warstwach**: warstwie danych (data), warstwie usług (service) i warstwie prezentacji (presentation). Błędy z warstw niższych mają być **tłumaczone/opakowywane** na każdej granicy, aby wyższe warstwy nie były zależne od szczegółów niższych.

Zaimplementuj:

1. **Hierarchię wyjątków** złożoną z co najmniej 3 klas: bazowego `WyjatekAplikacji` oraz podklas `WyjatekDanych` (z polem `zrodlo`) i `WyjatekUslugi` (z polem `operacja` oraz opcjonalną przyczyną `przyczyna`). Każda klasa przesłania `toString`.
2. **Warstwę danych** — funkcja `Future<String> pobierzRekord(int id)`, która dla `id <= 0` rzuca `WyjatekDanych('nieprawidłowe id', zrodlo: 'baza')`, a dla poprawnego `id` zwraca `'rekord-$id'`.
3. **Warstwę usług** — funkcja `Future<String> pobierzProfil(int id)`, która wywołuje warstwę danych, a **każdy** `WyjatekDanych` opakowuje w `WyjatekUslugi('nie udało się pobrać profilu', operacja: 'pobierzProfil', przyczyna: e)` (używając `rethrow`-podobnego wzorca `catch` + `throw` z zachowaniem przyczyny).
4. **Warstwę prezentacji** — funkcja `Future<String> pokazProfil(int id)`, która wywołuje warstwę usług i zwraca przyjazny komunikat: dla sukcesu `'Profil: <dane>'`, a dla `WyjatekUslugi` — `'Nie można wyświetlić profilu (operacja: <operacja>)'`.

Błąd z warstwy danych nie może "wyciec" w oryginalnej postaci do warstwy prezentacji — musi zostać przetłumaczony na `WyjatekUslugi`.

**Poziom trudności:** advanced

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `await pokazProfil(5)` | `Profil: rekord-5` |
| `await pokazProfil(0)` | `Nie można wyświetlić profilu (operacja: pobierzProfil)` |

### Wskazówki

1. W warstwie usług użyj `on WyjatekDanych catch (e)`, a następnie rzuć nowy `WyjatekUslugi`, przekazując oryginalny wyjątek w polu `przyczyna`. Dzięki temu zachowasz kontekst do celów logowania.
2. Warstwa prezentacji powinna łapać wyłącznie `WyjatekUslugi` — nie powinna wiedzieć nic o `WyjatekDanych`. To dowód poprawnego tłumaczenia błędów na granicy.
3. Wszystkie funkcje są `async`, więc używaj `try`/`catch` wokół `await`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// --- Hierarchia wyjątków (3 klasy) ---
abstract class WyjatekAplikacji implements Exception {
  final String komunikat;
  const WyjatekAplikacji(this.komunikat);
  @override
  String toString() => 'WyjatekAplikacji: $komunikat';
}

class WyjatekDanych extends WyjatekAplikacji {
  final String zrodlo;
  const WyjatekDanych(super.komunikat, {required this.zrodlo});
  @override
  String toString() => 'WyjatekDanych[$zrodlo]: $komunikat';
}

class WyjatekUslugi extends WyjatekAplikacji {
  final String operacja;
  final Object? przyczyna;
  const WyjatekUslugi(super.komunikat, {required this.operacja, this.przyczyna});
  @override
  String toString() =>
      'WyjatekUslugi[$operacja]: $komunikat (przyczyna: $przyczyna)';
}

// --- Warstwa danych ---
Future<String> pobierzRekord(int id) async {
  await Future<void>.delayed(const Duration(milliseconds: 5));
  if (id <= 0) {
    throw const WyjatekDanych('nieprawidłowe id', zrodlo: 'baza');
  }
  return 'rekord-$id';
}

// --- Warstwa usług: tłumaczy WyjatekDanych na WyjatekUslugi ---
Future<String> pobierzProfil(int id) async {
  try {
    return await pobierzRekord(id);
  } on WyjatekDanych catch (e) {
    // Opakowanie błędu warstwy niższej — zachowujemy oryginał jako przyczynę
    throw WyjatekUslugi(
      'nie udało się pobrać profilu',
      operacja: 'pobierzProfil',
      przyczyna: e,
    );
  }
}

// --- Warstwa prezentacji: tłumaczy WyjatekUslugi na komunikat dla użytkownika ---
Future<String> pokazProfil(int id) async {
  try {
    final dane = await pobierzProfil(id);
    return 'Profil: $dane';
  } on WyjatekUslugi catch (e) {
    return 'Nie można wyświetlić profilu (operacja: ${e.operacja})';
  }
}

Future<void> main() async {
  print(await pokazProfil(5));
  print(await pokazProfil(0));
}
// Oczekiwane wyjście:
// Profil: rekord-5
// Nie można wyświetlić profilu (operacja: pobierzProfil)
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`try`/`catch` łapie tylko błędy synchroniczne w bloku `try`** — aby przechwycić błąd z `Future`, trzeba go `await`-ować w tym bloku lub użyć `.catchError`.
2. **`Exception` to sytuacje obsługiwalne, `Error` to błędy programisty** — łap `Exception`, a `Error` naprawiaj w kodzie zamiast przechwytywać.
3. **`rethrow` zachowuje oryginalny stack trace** — używaj go zamiast `throw e`, gdy chcesz częściowo obsłużyć błąd i przekazać go dalej.
4. **Własna hierarchia wyjątków** (baza + wyspecjalizowane podklasy z polami kontekstowymi i `toString`) umożliwia selektywne przechwytywanie i czytelną diagnostykę.
5. **Zapomniany `await` powoduje nieobsłużone błędy Future** — błąd wtedy nie trafia do lokalnego `catch`, lecz do strefy.
6. **`runZonedGuarded` to globalna sieć bezpieczeństwa** — przechwytuje niezłapane błędy asynchroniczne w całej strefie, także z callbacków i zapomnianych `await`.
7. **Strumienie przekazują błędy jako osobne zdarzenia** — obsługuj je przez `onError` w `listen`, `try`/`catch` wokół `await for`, lub `handleError` do transformacji bez przerywania subskrypcji.

---

**Poprzedni moduł:** [Isolates i współbieżność](../07-concurrency/03-isolates.md)
**Następny moduł:** [dart:core](../08-standard-library/01-dart-core.md)
