---
id: "9.4"
title: "Obsługa błędów i wyjątków"
difficulty: "intermediate"
section: "09-advanced"
prerequisites:
  - "Klasy i konstruktory"
  - "Funkcje: deklaracje i parametry"
---

# 9.4 Obsługa błędów i wyjątków

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Klasy i konstruktory](../05-oop/01-classes-constructors.md), [Deklaracje funkcji i parametry](../04-functions/01-declarations-params.md)
- **Cele nauki:**
  1. Poprawnie używać konstrukcji `throw`, `rethrow`, `try`/`catch`/`finally` oraz klauzuli `on` do zgłaszania i przechwytywania błędów
  2. Rozróżniać koncepcyjne rodziny błędów `Exception` (sytuacje obsługiwalne) i `Error` (błędy programisty) oraz odczytywać `StackTrace`
  3. Projektować własną hierarchię wyjątków z polami kontekstowymi i przesłoniętym `toString`, aby przekazywać czytelne informacje diagnostyczne
  4. Obsługiwać błędy zarówno synchronicznie, jak i asynchronicznie (z `try`/`catch` wokół oczekiwanych — `await` — Futures)

---

## Wprowadzenie: dlaczego obsługa błędów ma znaczenie

Niezawodna aplikacja nie zakłada, że wszystko zawsze się powiedzie. Plik może nie istnieć, dane wejściowe mogą być błędne, a operacja może naruszyć niezmiennik biznesowy. Dart oferuje spójny mechanizm zgłaszania błędów (`throw`) oraz ich przechwytywania (`try`/`catch`), a także bogatą hierarchię typów, która pomaga odróżnić błędy **spodziewane i obsługiwalne** od **błędów w kodzie**.

Ten moduł skupia się na fundamentach obsługi błędów: składni `throw`/`rethrow`/`try`/`catch`/`finally`, klauzuli `on`, różnicy między `Exception` a `Error`, projektowaniu własnych wyjątków oraz odczytywaniu śladu stosu (`StackTrace`). Programowanie asynchroniczne i propagacja błędów przez strefy (`runZonedGuarded`) oraz strumienie są omówione szerzej w module [Obsługa błędów w kodzie asynchronicznym](../07-concurrency/04-error-handling-async.md).

---

## `throw` i podstawowa struktura `try`/`catch`/`finally`

Błąd zgłasza się słowem kluczowym `throw`. Technicznie można rzucić dowolny obiekt, ale zaleca się rzucanie instancji typów implementujących `Exception` lub rozszerzających `Error`. Blok `try` obejmuje kod, który może zawieść, `catch` przechwytuje rzucony obiekt, a `finally` wykonuje się **zawsze** — zarówno gdy błąd wystąpił, jak i gdy nie.

Poniższy przykład pokazuje otwieranie „zasobu” (uproszczony uchwyt), który musi zostać zwolniony w bloku `finally` niezależnie od tego, czy przetwarzanie się powiodło.

```dart
// finally gwarantuje zwolnienie zasobu nawet, gdy wystąpi błąd
class UchwytPliku {
  final String nazwa;
  bool otwarty = true;
  UchwytPliku(this.nazwa);
  void zamknij() => otwarty = false; // zwolnienie zasobu
}

void przetworz(UchwytPliku uchwyt, bool zawiedz) {
  try {
    if (zawiedz) {
      throw StateError('Uszkodzone dane w ${uchwyt.nazwa}'); // zgłoszenie błędu
    }
    print('Przetworzono ${uchwyt.nazwa}');
  } finally {
    uchwyt.zamknij(); // wykonuje się zawsze — także po throw
    print('Zamknięto ${uchwyt.nazwa} (otwarty=${uchwyt.otwarty})');
  }
}

void main() {
  try {
    przetworz(UchwytPliku('dane.txt'), true);
  } catch (e) {
    print('Obsłużono na zewnątrz: $e');
  }
}
// Oczekiwane wyjście:
// Zamknięto dane.txt (otwarty=false)
// Obsłużono na zewnątrz: Bad state: Uszkodzone dane w dane.txt
```

Drugi przykład pokazuje, że wartość zwrócona w bloku `try` jest ustalana, a mimo to `finally` i tak się wykona przed faktycznym powrotem z funkcji.

```dart
// finally wykonuje się nawet po napotkaniu return w try
int policzZLogiem(int a, int b) {
  try {
    return a + b; // wartość obliczona i zapamiętana
  } finally {
    print('finally: obliczenie zakończone'); // wykona się przed zwróceniem
  }
}

void main() {
  final wynik = policzZLogiem(2, 3);
  print('Wynik: $wynik');
}
// Oczekiwane wyjście:
// finally: obliczenie zakończone
// Wynik: 5
```

---

## Klauzula `on` — filtrowanie wyjątków po typie

Klauzula `on` pozwala przechwytywać wyłącznie wyjątki określonego typu. Można łączyć `on <Typ>` z `catch (e)` (aby uzyskać dostęp do obiektu błędu) lub z `catch (e, s)` (aby dodatkowo uzyskać `StackTrace`). Wiele klauzul `on` układa się od najbardziej szczegółowej do najbardziej ogólnej.

Poniższy przykład parsuje prostą linię konfiguracji `klucz=wartość` i rozróżnia dwa rodzaje problemów: brak znaku `=` (`FormatException`) oraz niedozwoloną wartość (`ArgumentError`).

```dart
// on <Typ> catch (e) — dopasowanie wielu typów wyjątków w kolejności od szczegółu
MapEntry<String, int> parsujWpis(String linia) {
  final czesci = linia.split('=');
  if (czesci.length != 2) {
    throw FormatException('Brak znaku "="', linia); // niepoprawny format
  }
  final wartosc = int.tryParse(czesci[1]);
  if (wartosc == null || wartosc < 0) {
    throw ArgumentError.value(czesci[1], 'wartość', 'musi być liczbą >= 0');
  }
  return MapEntry(czesci[0], wartosc);
}

void main() {
  for (final linia in ['limit=10', 'zepsuta-linia', 'limit=-3']) {
    try {
      final wpis = parsujWpis(linia);
      print('OK: ${wpis.key} -> ${wpis.value}');
    } on FormatException catch (e) {
      print('Błąd formatu: ${e.message} (źródło: ${e.source})');
    } on ArgumentError catch (e) {
      print('Błąd argumentu: ${e.message}');
    }
  }
}
// Oczekiwane wyjście:
// OK: limit -> 10
// Błąd formatu: Brak znaku "=" (źródło: zepsuta-linia)
// Błąd argumentu: musi być liczbą >= 0
```

Drugi przykład pokazuje ogólny `catch` bez `on`, który łapie **każdy** rzucony obiekt, oraz dostęp do `StackTrace` jako drugiego parametru.

```dart
// catch bez on łapie dowolny rzucony obiekt; drugi parametr wiąże StackTrace
void ryzykowna(int tryb) {
  switch (tryb) {
    case 0:
      throw 'zwykły string jako błąd'; // można rzucić dowolny obiekt
    default:
      throw Exception('nietypowy tryb: $tryb');
  }
}

void main() {
  for (final tryb in [0, 9]) {
    try {
      ryzykowna(tryb);
    } catch (e, stackTrace) {
      // e — rzucony obiekt, stackTrace — ślad stosu w momencie throw
      print('Złapano (${e.runtimeType}): $e');
      print('Ślad stosu dostępny: ${stackTrace.toString().isNotEmpty}');
    }
  }
}
// Oczekiwane wyjście:
// Złapano (String): zwykły string jako błąd
// Ślad stosu dostępny: true
// Złapano (_Exception): Exception: nietypowy tryb: 9
// Ślad stosu dostępny: true
```

---

## `rethrow` — przekazanie błędu dalej bez utraty śladu stosu

Czasem chcemy w bieżącej warstwie **częściowo** zareagować na błąd (np. zapisać go w logu lub zwiększyć licznik), a następnie przekazać go wyżej, aby zdecydowała o nim warstwa nadrzędna. Służy do tego `rethrow`, który przekazuje **ten sam** obiekt błędu wraz z **oryginalnym** `StackTrace`. Rzucenie `throw e` w tym miejscu zgubiłoby pierwotny ślad stosu.

Poniższy przykład zlicza próby i loguje błąd na poziomie pośrednim, po czym przekazuje go dalej za pomocą `rethrow`.

```dart
// rethrow zachowuje oryginalny stack trace przy przekazywaniu błędu w górę
int licznikProb = 0;

void operacjaBazowa(int wartosc) {
  if (wartosc.isNegative) {
    throw RangeError('Wartość nie może być ujemna: $wartosc');
  }
}

void operacjaZLogiem(int wartosc) {
  try {
    operacjaBazowa(wartosc);
  } on RangeError {
    licznikProb++; // częściowa reakcja: logowanie/telemetria
    print('Log: nieudana próba #$licznikProb');
    rethrow; // przekazujemy ten sam błąd wyżej, zachowując oryginalny ślad
  }
}

void main() {
  try {
    operacjaZLogiem(-5);
  } catch (e) {
    print('Ostateczna obsługa: $e');
  }
}
// Oczekiwane wyjście:
// Log: nieudana próba #1
// Ostateczna obsługa: RangeError: Wartość nie może być ujemna: -5
```

---

## Hierarchia `Exception` vs `Error`

Dart rozróżnia dwie koncepcyjne rodziny błędów:

- **`Exception`** — reprezentuje sytuacje **spodziewane i możliwe do obsłużenia** w czasie działania (np. błędne dane wejściowe, brak pliku, przekroczony limit). Kod aplikacji powinien je przechwytywać i sensownie na nie reagować.
- **`Error`** — reprezentuje **błędy programisty**, których wystąpienie oznacza, że kod jest niepoprawny (np. `RangeError`, `StateError`, `ArgumentError`, `AssertionError`, `UnimplementedError`). Zamiast je przechwytywać, należy naprawić kod. Łapanie `Error` jest zwykle antywzorcem.

Poniższa tabela porównuje obie rodziny:

| Cecha | `Exception` | `Error` |
|-------|-------------|---------|
| Przyczyna | Warunki środowiska / dane | Błąd w logice programu |
| Zalecane działanie | Przechwycić i obsłużyć | Naprawić kod |
| Typowi przedstawiciele | `FormatException`, `IOException` | `RangeError`, `StateError`, `ArgumentError` |
| Zawiera `stackTrace`? | Nie (dostarcza go `catch`) | Tak — pole `Error.stackTrace` |

Poniższy przykład pokazuje, jak w jednym bloku `catch` odróżnić obie rodziny za pomocą klauzul `on`.

```dart
// Exception -> sytuacja obsługiwalna; Error -> błąd programisty (napraw kod)
void pobierzElement(List<int> lista, int indeks) {
  if (lista.isEmpty) {
    // Sytuacja obsługiwalna — brak danych to spodziewany scenariusz
    throw const FormatException('Lista jest pusta');
  }
  print('Element: ${lista[indeks]}'); // dla złego indeksu rzuci RangeError
}

void main() {
  final przypadki = <(List<int>, int)>[
    ([10, 20, 30], 1),
    (<int>[], 0),
    ([10, 20, 30], 99),
  ];
  for (final (lista, indeks) in przypadki) {
    try {
      pobierzElement(lista, indeks);
    } on Exception catch (e) {
      print('Obsługiwalny wyjątek: $e');
    } on Error catch (e) {
      print('Błąd programisty (${e.runtimeType}) — napraw kod!');
    }
  }
}
// Oczekiwane wyjście:
// Element: 20
// Obsługiwalny wyjątek: FormatException: Lista jest pusta
// Błąd programisty (RangeError) — napraw kod!
```

---

## Odczytywanie `StackTrace`

Każdy przechwycony błąd może dostarczyć `StackTrace` — zapis kolejnych ramek wywołań w momencie zgłoszenia błędu. Ślad stosu jest kluczowy przy diagnozowaniu: pokazuje ścieżkę, którą podążył program aż do miejsca błędu. `StackTrace` uzyskujemy jako **drugi parametr** klauzuli `catch`.

Poniższy przykład wywołuje kilka zagnieżdżonych funkcji i sprawdza, że najświeższa ramka stosu dotyczy funkcji, która faktycznie rzuciła błąd.

```dart
// Drugi parametr catch (e, s) wiąże StackTrace z momentu rzucenia błędu
void poziomNajnizszy() => throw StateError('awaria na dole');
void poziomSrodkowy() => poziomNajnizszy();
void poziomGorny() => poziomSrodkowy();

void main() {
  try {
    poziomGorny();
  } catch (e, stackTrace) {
    print('Błąd: $e');
    final ramki = stackTrace.toString().split('\n');
    // Najświeższa ramka odpowiada funkcji, która rzuciła błąd
    print('Najświeższa ramka wskazuje poziomNajnizszy: '
        '${ramki.first.contains('poziomNajnizszy')}');
    print('Liczba ramek jest niezerowa: ${ramki.length > 1}');
  }
}
// Oczekiwane wyjście:
// Błąd: Bad state: awaria na dole
// Najświeższa ramka wskazuje poziomNajnizszy: true
// Liczba ramek jest niezerowa: true
```

---

## Własna hierarchia wyjątków (baza + 2 wyspecjalizowane podklasy)

W realnych aplikacjach warto definiować **własne typy wyjątków**, aby przenosić kontekst błędu (np. numer konta, nazwę operacji, brakujący klucz konfiguracji) i umożliwiać selektywne przechwytywanie po typie. Dobra hierarchia składa się z klasy bazowej oraz wyspecjalizowanych podklas, z których każda niesie własne pola kontekstowe i przesłania `toString`.

Poniższy przykład modeluje domenę **systemu bankowego**. Bazowy `WyjatekBankowy` definiuje wspólny kontrakt, a dwie podklasy dodają pola kontekstowe: `NiewystarczajaceSrodki` (kwota żądana i saldo) oraz `KontoZablokowane` (numer konta i powód blokady).

```dart
// Bazowa klasa hierarchii — wspólny kontrakt dla wszystkich wyjątków bankowych
abstract class WyjatekBankowy implements Exception {
  final String komunikat;
  const WyjatekBankowy(this.komunikat);

  @override
  String toString() => 'WyjatekBankowy: $komunikat';
}

// Podklasa 1 — pola kontekstowe: żądana kwota oraz dostępne saldo
class NiewystarczajaceSrodki extends WyjatekBankowy {
  final double zadanaKwota;
  final double dostepneSaldo;
  const NiewystarczajaceSrodki(this.zadanaKwota, this.dostepneSaldo)
      : super('Niewystarczające środki na koncie');

  double get brakujacaKwota => zadanaKwota - dostepneSaldo;

  @override
  String toString() => 'NiewystarczajaceSrodki: brakuje '
      '${brakujacaKwota.toStringAsFixed(2)} (żądano $zadanaKwota, '
      'saldo $dostepneSaldo)';
}

// Podklasa 2 — pola kontekstowe: numer konta oraz powód blokady
class KontoZablokowane extends WyjatekBankowy {
  final String numerKonta;
  final String powod;
  const KontoZablokowane(this.numerKonta, this.powod)
      : super('Konto jest zablokowane');

  @override
  String toString() =>
      'KontoZablokowane[konto=$numerKonta]: $powod';
}

void main() {
  final bledy = <WyjatekBankowy>[
    const NiewystarczajaceSrodki(500.0, 320.50),
    const KontoZablokowane('PL-001', 'podejrzenie oszustwa'),
  ];
  for (final b in bledy) {
    print(b); // korzysta z przesłoniętego toString każdej podklasy
  }
}
// Oczekiwane wyjście:
// NiewystarczajaceSrodki: brakuje 179.50 (żądano 500.0, saldo 320.5)
// KontoZablokowane[konto=PL-001]: podejrzenie oszustwa
```

Drugi przykład pokazuje, jak hierarchia umożliwia **selektywne przechwytywanie**: można złapać konkretną podklasę albo cały rodzaj przez typ bazowy. Pamiętaj o kolejności — najbardziej szczegółowa klauzula `on` musi być pierwsza.

```dart
abstract class WyjatekBankowy implements Exception {
  final String komunikat;
  const WyjatekBankowy(this.komunikat);
  @override
  String toString() => 'WyjatekBankowy: $komunikat';
}

class NiewystarczajaceSrodki extends WyjatekBankowy {
  final double zadanaKwota;
  final double dostepneSaldo;
  const NiewystarczajaceSrodki(this.zadanaKwota, this.dostepneSaldo)
      : super('Niewystarczające środki');
  @override
  String toString() => 'NiewystarczajaceSrodki(żądano $zadanaKwota)';
}

class KontoZablokowane extends WyjatekBankowy {
  final String numerKonta;
  final String powod;
  const KontoZablokowane(this.numerKonta, this.powod)
      : super('Konto zablokowane');
  @override
  String toString() => 'KontoZablokowane[$numerKonta]: $powod';
}

void wyplac(double saldo, double kwota, {bool zablokowane = false}) {
  if (zablokowane) {
    throw const KontoZablokowane('PL-777', 'zaległości');
  }
  if (kwota > saldo) {
    throw NiewystarczajaceSrodki(kwota, saldo);
  }
  print('Wypłacono $kwota, pozostało ${saldo - kwota}');
}

void bezpiecznaWyplata(double saldo, double kwota, {bool zablokowane = false}) {
  try {
    wyplac(saldo, kwota, zablokowane: zablokowane);
  } on NiewystarczajaceSrodki catch (e) {
    // Najbardziej szczegółowa klauzula on musi być pierwsza
    print('Odmowa (środki): $e');
  } on WyjatekBankowy catch (e) {
    // Łapie każdy inny wyjątek bankowy (w tym KontoZablokowane)
    print('Odmowa (ogólna): $e');
  }
}

void main() {
  bezpiecznaWyplata(1000, 200);
  bezpiecznaWyplata(100, 500);
  bezpiecznaWyplata(1000, 200, zablokowane: true);
}
// Oczekiwane wyjście:
// Wypłacono 200.0, pozostało 800.0
// Odmowa (środki): NiewystarczajaceSrodki(żądano 500.0)
// Odmowa (ogólna): KontoZablokowane[PL-777]: zaległości
```

---

## Obsługa synchroniczna vs asynchroniczna

### Obsługa synchroniczna

W kodzie synchronicznym błąd propaguje natychmiast w górę stosu wywołań, aż napotka pasujący `catch`. Poniższy przykład weryfikuje dane wejściowe konfiguracji i obsługuje błąd synchronicznie, dobierając wartość domyślną.

```dart
// Błąd synchroniczny propaguje natychmiast w bieżącym stosie wywołań
int odczytajTimeout(Map<String, String> konfiguracja) {
  final surowy = konfiguracja['timeout'];
  if (surowy == null) {
    throw ArgumentError('Brak klucza "timeout"'); // brak wymaganej wartości
  }
  final wartosc = int.parse(surowy); // może rzucić FormatException
  return wartosc;
}

void main() {
  final konfiguracje = <Map<String, String>>[
    {'timeout': '30'},
    <String, String>{},
    {'timeout': 'xxx'},
  ];
  for (final k in konfiguracje) {
    try {
      print('Timeout: ${odczytajTimeout(k)}');
    } on ArgumentError catch (e) {
      print('Użyto wartości domyślnej 60 (${e.message})');
    } on FormatException {
      print('Nieprawidłowa liczba — użyto wartości domyślnej 60');
    }
  }
}
// Oczekiwane wyjście:
// Timeout: 30
// Użyto wartości domyślnej 60 (Brak klucza "timeout")
// Nieprawidłowa liczba — użyto wartości domyślnej 60
```

### Obsługa asynchroniczna z `try`/`catch` wokół `await`

Gdy funkcja jest oznaczona `async`, a błędna operacja jest **oczekiwana** przez `await`, zwykły `try`/`catch` działa równie intuicyjnie jak w kodzie synchronicznym. To zalecany sposób obsługi błędów asynchronicznych. (Bardziej zaawansowane scenariusze — nieobsłużone błędy `Future`, `runZonedGuarded`, błędy w strumieniach — omawia moduł [Obsługa błędów w kodzie asynchronicznym](../07-concurrency/04-error-handling-async.md).)

Poniższy przykład symuluje odczyt pliku, który po krótkim opóźnieniu może zakończyć się błędem, a `try`/`catch` wokół `await` przechwytuje ten błąd.

```dart
import 'dart:async';

// Symulacja asynchronicznego odczytu pliku, który może zawieść
Future<String> wczytajPlik(String sciezka) async {
  await Future<void>.delayed(const Duration(milliseconds: 10));
  if (!sciezka.endsWith('.txt')) {
    throw const FormatException('Obsługiwane są tylko pliki .txt');
  }
  return 'zawartość $sciezka';
}

Future<void> main() async {
  for (final sciezka in ['raport.txt', 'obraz.png']) {
    try {
      // await sprawia, że błąd z Future jest rzucany w tym bloku try
      final tresc = await wczytajPlik(sciezka);
      print('Wczytano: $tresc');
    } on FormatException catch (e) {
      print('Pominięto $sciezka: ${e.message}');
    } finally {
      print('Zakończono próbę dla $sciezka');
    }
  }
}
// Oczekiwane wyjście:
// Wczytano: zawartość raport.txt
// Zakończono próbę dla raport.txt
// Pominięto obraz.png: Obsługiwane są tylko pliki .txt
// Zakończono próbę dla obraz.png
```

---

## Ćwiczenie 1: Strategia obsługi błędów dla aplikacji trójwarstwowej (advanced)

### Opis problemu

Zaprojektuj strategię obsługi błędów dla aplikacji o **trzech warstwach**: warstwie danych (data), warstwie usług (service) i warstwie prezentacji (presentation). Błędy z warstw niższych mają być **tłumaczone/opakowywane** na każdej granicy, aby wyższe warstwy nie zależały od szczegółów niższych.

W tym ćwiczeniu modelujemy prosty **katalog produktów** (inny scenariusz niż przykład z modułu 7.4). Zaimplementuj:

1. **Hierarchię wyjątków** złożoną z co najmniej 3 klas: bazowego `WyjatekKatalogu` oraz podklas `WyjatekRepozytorium` (z polem `tabela`) i `WyjatekLogikiBiznesowej` (z polem `regula` oraz opcjonalną przyczyną `przyczyna`). Każda klasa przesłania `toString`.
2. **Warstwę danych** — funkcja `String pobierzCene(String sku)`, która dla nieznanego `sku` (spoza mapy) rzuca `WyjatekRepozytorium('brak rekordu', tabela: 'ceny')`, a dla znanego zwraca cenę jako string (np. `'19.99'`).
3. **Warstwę usług** — funkcja `double obliczCeneBrutto(String sku)`, która wywołuje warstwę danych, parsuje cenę i mnoży ją przez 1.23 (VAT). **Każdy** `WyjatekRepozytorium` opakowuje w `WyjatekLogikiBiznesowej('nie można wycenić produktu', regula: 'wycena', przyczyna: e)`.
4. **Warstwę prezentacji** — funkcja `String pokazCene(String sku)`, która wywołuje warstwę usług i zwraca przyjazny komunikat: dla sukcesu `'Cena brutto: <kwota z dwoma miejscami>'`, a dla `WyjatekLogikiBiznesowej` — `'Produkt niedostępny (reguła: <regula>)'`.

Błąd z warstwy danych nie może „wyciec” w oryginalnej postaci do warstwy prezentacji — musi zostać przetłumaczony na `WyjatekLogikiBiznesowej`.

**Poziom trudności:** advanced

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `pokazCene('ABC')` (cena 100.00) | `Cena brutto: 123.00` |
| `pokazCene('NIEZNANY')` | `Produkt niedostępny (reguła: wycena)` |

### Wskazówki

1. W warstwie usług użyj `on WyjatekRepozytorium catch (e)`, a następnie rzuć nowy `WyjatekLogikiBiznesowej`, przekazując oryginalny wyjątek w polu `przyczyna`. Dzięki temu zachowasz kontekst do logowania.
2. Warstwa prezentacji powinna łapać wyłącznie `WyjatekLogikiBiznesowej` — nie powinna wiedzieć nic o `WyjatekRepozytorium`. To dowód poprawnego tłumaczenia błędów na granicy.
3. Do formatowania kwoty użyj `toStringAsFixed(2)`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// --- Hierarchia wyjątków (3 klasy) ---
abstract class WyjatekKatalogu implements Exception {
  final String komunikat;
  const WyjatekKatalogu(this.komunikat);
  @override
  String toString() => 'WyjatekKatalogu: $komunikat';
}

class WyjatekRepozytorium extends WyjatekKatalogu {
  final String tabela;
  const WyjatekRepozytorium(super.komunikat, {required this.tabela});
  @override
  String toString() => 'WyjatekRepozytorium[$tabela]: $komunikat';
}

class WyjatekLogikiBiznesowej extends WyjatekKatalogu {
  final String regula;
  final Object? przyczyna;
  const WyjatekLogikiBiznesowej(
    super.komunikat, {
    required this.regula,
    this.przyczyna,
  });
  @override
  String toString() =>
      'WyjatekLogikiBiznesowej[$regula]: $komunikat (przyczyna: $przyczyna)';
}

// --- Warstwa danych ---
const _ceny = <String, String>{'ABC': '100.00', 'XYZ': '19.99'};

String pobierzCene(String sku) {
  final cena = _ceny[sku];
  if (cena == null) {
    throw const WyjatekRepozytorium('brak rekordu', tabela: 'ceny');
  }
  return cena;
}

// --- Warstwa usług: tłumaczy WyjatekRepozytorium na WyjatekLogikiBiznesowej ---
double obliczCeneBrutto(String sku) {
  try {
    final netto = double.parse(pobierzCene(sku));
    return netto * 1.23;
  } on WyjatekRepozytorium catch (e) {
    // Opakowanie błędu warstwy niższej — zachowujemy oryginał jako przyczynę
    throw WyjatekLogikiBiznesowej(
      'nie można wycenić produktu',
      regula: 'wycena',
      przyczyna: e,
    );
  }
}

// --- Warstwa prezentacji: tłumaczy WyjatekLogikiBiznesowej na komunikat ---
String pokazCene(String sku) {
  try {
    final brutto = obliczCeneBrutto(sku);
    return 'Cena brutto: ${brutto.toStringAsFixed(2)}';
  } on WyjatekLogikiBiznesowej catch (e) {
    return 'Produkt niedostępny (reguła: ${e.regula})';
  }
}

void main() {
  print(pokazCene('ABC'));
  print(pokazCene('NIEZNANY'));
}
// Oczekiwane wyjście:
// Cena brutto: 123.00
// Produkt niedostępny (reguła: wycena)
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`try`/`catch`/`finally` to trzon obsługi błędów** — `finally` wykonuje się zawsze, także po `return` czy `throw`, i nadaje się do zwalniania zasobów.
2. **Klauzula `on` filtruje wyjątki po typie** — układaj klauzule od najbardziej szczegółowej do najbardziej ogólnej; `catch` bez `on` łapie dowolny obiekt.
3. **`rethrow` zachowuje oryginalny `StackTrace`** — używaj go zamiast `throw e`, gdy chcesz częściowo obsłużyć błąd i przekazać go wyżej.
4. **`Exception` to sytuacje obsługiwalne, `Error` to błędy programisty** — przechwytuj `Exception`, a `Error` naprawiaj w kodzie zamiast łapać.
5. **`StackTrace` (drugi parametr `catch`) wskazuje ścieżkę do miejsca błędu** — najświeższa ramka odpowiada funkcji, która rzuciła wyjątek.
6. **Własna hierarchia wyjątków** (baza + wyspecjalizowane podklasy z polami kontekstowymi i `toString`) umożliwia selektywne przechwytywanie i czytelną diagnostykę.
7. **Tłumacz błędy na granicach warstw** — opakowuj wyjątki niższych warstw w typy wyższego poziomu, aby warstwy nadrzędne nie zależały od szczegółów implementacyjnych.

---

**Poprzedni moduł:** [Adnotacje i widoczność bibliotek](../09-advanced/03-annotations-visibility.md)
**Spis treści:** [Powrót do spisu treści](../../README.md)
