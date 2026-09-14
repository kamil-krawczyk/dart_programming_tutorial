---
id: "8.4"
title: "dart:io — pliki, procesy i sieć"
difficulty: "advanced"
section: "08-standard-library"
prerequisites:
  - "Futures i async/await"
  - "Streams"
  - "dart:async"
---

# 8.4 dart:io — pliki, procesy i sieć

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Futures i async/await](../07-concurrency/01-async-await.md), [Streams](../07-concurrency/02-streams.md), [dart:async](03-dart-async.md)
- **Cele nauki:**
  1. Wykonywać operacje na systemie plików (`File`, `Directory`, `Link`) w wariancie synchronicznym i asynchronicznym oraz poprawnie obsługiwać `FileSystemException`.
  2. Korzystać ze standardowych strumieni (`stdin`, `stdout`, `stderr`) oraz uruchamiać zewnętrzne procesy (`Process`), przechwytując ich wyjście i kod zakończenia.
  3. Budować aplikacje sieciowe: klienta i serwer HTTP (`HttpClient`, `HttpServer`), surowe połączenia TCP (`Socket`, `ServerSocket`) oraz wykrywać środowisko uruchomieniowe przez klasę `Platform`.

---

## 1. Biblioteka `dart:io` — do czego służy

Biblioteka `dart:io` udostępnia interfejs do systemu operacyjnego: plików, katalogów, procesów, gniazd sieciowych oraz serwerów i klientów HTTP. Jest dostępna wyłącznie w aplikacjach **konsolowych i serwerowych** (Dart VM, kompilacja AOT) — **nie** w przeglądarce (Flutter Web), gdzie odpowiednikiem są API przeglądarki.

Aby korzystać z tej biblioteki, importujemy ją na początku pliku:

```dart
import 'dart:io';
```

Charakterystyczną cechą `dart:io` jest to, że większość operacji ma **dwie wersje**: asynchroniczną (zwraca `Future`, nie blokuje pętli zdarzeń) oraz synchroniczną (z sufiksem `Sync`, blokuje wykonanie do zakończenia). W kodzie serwerowym niemal zawsze preferujemy wersje asynchroniczne, aby nie blokować obsługi innych żądań.

---

## 2. `File` — odczyt i zapis plików

Klasa `File` reprezentuje plik na dysku. Sam obiekt `File` jest tylko *uchwytem do ścieżki* — jego utworzenie nie dotyka dysku. Dopiero wywołanie metody (np. `readAsString`) wykonuje operację I/O.

### 2.1 Zapis i odczyt asynchroniczny

Wersje asynchroniczne zwracają `Future` i są zalecane w kodzie serwerowym. Poniżej zapisujemy tekst, a następnie go odczytujemy.

```dart
import 'dart:io';

Future<void> main() async {
  final plik = File('${Directory.systemTemp.path}/notatka.txt');

  // writeAsString zwraca Future<File> — czekamy na zakończenie zapisu.
  await plik.writeAsString('Pierwsza linia\nDruga linia\n');

  // readAsString wczytuje całą zawartość jako String (domyślnie UTF-8).
  final tresc = await plik.readAsString();
  print('Zawartość:\n$tresc');

  // readAsLines rozbija zawartość na listę linii.
  final linie = await plik.readAsLines();
  print('Liczba linii: ${linie.length}');

  await plik.delete(); // sprzątamy plik tymczasowy
}
// Oczekiwane wyjście:
// Zawartość:
// Pierwsza linia
// Druga linia
//
// Liczba linii: 2
```

### 2.2 Zapis i odczyt synchroniczny

Wersje z sufiksem `Sync` zwracają wynik bezpośrednio (bez `Future`), ale **blokują** isolate do czasu zakończenia operacji. Nadają się do prostych skryptów i narzędzi CLI, gdzie blokowanie nie stanowi problemu.

```dart
import 'dart:io';

void main() {
  final plik = File('${Directory.systemTemp.path}/dane.txt');

  // writeAsStringSync zapisuje synchronicznie — brak Future, brak await.
  plik.writeAsStringSync('linia A\nlinia B\nlinia C\n');

  // readAsLinesSync zwraca List<String> bezpośrednio.
  final linie = plik.readAsLinesSync();
  print('Wczytano ${linie.length} linii: $linie');

  plik.deleteSync();
}
// Oczekiwane wyjście:
// Wczytano 3 linii: [linia A, linia B, linia C]
```

### 2.3 Dopisywanie i praca na bajtach

Tryb `FileMode.append` dopisuje do istniejącego pliku zamiast go nadpisywać. Metody `writeAsBytes`/`readAsBytes` operują na surowych bajtach (`List<int>` / `Uint8List`), co przydaje się dla danych binarnych.

```dart
import 'dart:io';

Future<void> main() async {
  final plik = File('${Directory.systemTemp.path}/log.txt');

  await plik.writeAsString('start\n');
  // FileMode.append dopisuje na końcu, nie kasując wcześniejszej treści.
  await plik.writeAsString('kolejny wpis\n', mode: FileMode.append);

  // Operacje bajtowe — zapisujemy i czytamy surowe bajty.
  await plik.writeAsBytes([72, 105]); // nadpisuje: bajty 'H', 'i'
  final bajty = await plik.readAsBytes();
  print('Bajty: $bajty, jako tekst: ${String.fromCharCodes(bajty)}');

  await plik.delete();
}
// Oczekiwane wyjście:
// Bajty: [72, 105], jako tekst: Hi
```

---

## 3. Obsługa błędów — `FileSystemException`

Operacje na plikach mogą się nie powieść: plik nie istnieje, brak uprawnień, katalog jest zajęty. W takich przypadkach `dart:io` rzuca `FileSystemException`. Zawsze opakowujemy ryzykowne operacje w `try`/`catch`.

### 3.1 Obsługa błędów asynchronicznie

Poniższy przykład próbuje odczytać nieistniejący plik i przechwytuje `FileSystemException`. Pole `osError` zawiera szczegóły błędu z systemu operacyjnego (m.in. `errorCode`).

```dart
import 'dart:io';

Future<void> main() async {
  final plik = File('/sciezka/ktora/nie/istnieje.txt');

  try {
    // Odczyt nieistniejącego pliku rzuci FileSystemException.
    final tresc = await plik.readAsString();
    print(tresc);
  } on FileSystemException catch (e) {
    // e.message opisuje błąd, e.path wskazuje ścieżkę, e.osError szczegóły OS.
    print('Błąd systemu plików: ${e.message}');
    print('Ścieżka: ${e.path}');
    print('Kod OS: ${e.osError?.errorCode}');
  }
}
// Przykładowe wyjście (treść komunikatu zależy od systemu):
// Błąd systemu plików: Cannot open file, path = '/sciezka/ktora/nie/istnieje.txt' (OS Error: No such file or directory, errno = 2)
// Ścieżka: /sciezka/ktora/nie/istnieje.txt
// Kod OS: 2
```

### 3.2 Obsługa błędów synchronicznie

Wersje `Sync` również rzucają `FileSystemException`, którą łapiemy tym samym mechanizmem. Poniżej odróżniamy "plik nie istnieje" (sprawdzenie wstępne) od błędu odmowy dostępu (odczyt katalogu jak pliku).

```dart
import 'dart:io';

void main() {
  final plik = File('/root/tajne_dane.txt');

  try {
    // Wstępne sprawdzenie istnienia pliku bez rzucania wyjątku.
    if (!plik.existsSync()) {
      print('Plik nie istnieje — pomijam odczyt.');
      return;
    }
    // Jeśli plik istnieje, ale brak uprawnień, readAsStringSync rzuci wyjątek.
    final tresc = plik.readAsStringSync();
    print(tresc);
  } on FileSystemException catch (e) {
    // Typowe przyczyny: brak uprawnień (permission denied), plik zajęty.
    print('Nie udało się odczytać pliku: ${e.message}');
  }
}
// Przykładowe wyjście:
// Plik nie istnieje — pomijam odczyt.
```

---

## 4. `Directory` i `Link`

`Directory` reprezentuje katalog, a `Link` — dowiązanie symboliczne. Obie klasy dziedziczą wspólny interfejs `FileSystemEntity` (z metodami takimi jak `exists`, `delete`, `rename`, `stat`).

```dart
import 'dart:io';

Future<void> main() async {
  // Tworzymy tymczasowy katalog roboczy.
  final katalog = await Directory.systemTemp.createTemp('demo_');
  print('Utworzono katalog: ${katalog.path}');

  // Tworzymy plik wewnątrz katalogu.
  final plik = File('${katalog.path}/dane.txt');
  await plik.writeAsString('zawartość');

  // list() zwraca Stream<FileSystemEntity> — listujemy zawartość katalogu.
  await for (final wpis in katalog.list()) {
    // FileSystemEntity.typeSync rozróżnia plik, katalog i link.
    print('Wpis: ${wpis.path} (typ: ${FileSystemEntity.typeSync(wpis.path)})');
  }

  // recursive: true usuwa katalog wraz z zawartością.
  await katalog.delete(recursive: true);
  print('Katalog istnieje po usunięciu: ${await katalog.exists()}');
}
// Przykładowe wyjście (nazwa katalogu jest losowa):
// Utworzono katalog: /tmp/demo_XXXXXX
// Wpis: /tmp/demo_XXXXXX/dane.txt (typ: file)
// Katalog istnieje po usunięciu: false
```

Dowiązanie symboliczne tworzymy przez `Link.create`. Metoda `target()` zwraca ścieżkę, na którą wskazuje link.

```dart
import 'dart:io';

Future<void> main() async {
  final katalog = await Directory.systemTemp.createTemp('link_');
  final plik = File('${katalog.path}/oryginal.txt');
  await plik.writeAsString('treść oryginału');

  final link = Link('${katalog.path}/skrot.txt');
  // create tworzy dowiązanie symboliczne wskazujące na oryginalny plik.
  await link.create(plik.path);

  // target() zwraca ścieżkę docelową dowiązania.
  print('Link wskazuje na: ${await link.target()}');
  // Odczyt przez link czyta zawartość pliku docelowego.
  print('Treść przez link: ${await File(link.path).readAsString()}');

  await katalog.delete(recursive: true);
}
// Przykładowe wyjście:
// Link wskazuje na: /tmp/link_XXXXXX/oryginal.txt
// Treść przez link: treść oryginału
```

---

## 5. Standardowe strumienie: `stdin`, `stdout`, `stderr`

`dart:io` udostępnia trzy globalne strumienie:

- `stdout` — standardowe wyjście (odpowiednik `print`, ale z pełną kontrolą),
- `stderr` — standardowe wyjście błędów (komunikaty diagnostyczne),
- `stdin` — standardowe wejście (odczyt danych od użytkownika).

`stdout.write` nie dodaje znaku nowej linii (w przeciwieństwie do `stdout.writeln`), co pozwala pisać zachęty (prompty) w tej samej linii.

```dart
import 'dart:io';

void main() {
  // write nie dodaje nowej linii — kursor zostaje w tej samej linii.
  stdout.write('Podaj imię: ');
  // readLineSync czyta jedną linię z wejścia (zwraca null na końcu strumienia).
  final imie = stdin.readLineSync();

  // Komunikaty diagnostyczne kierujemy na stderr, nie na stdout.
  stderr.writeln('[debug] odczytano wejście');

  stdout.writeln('Cześć, ${imie ?? "nieznajomy"}!');
}
// Przykładowa sesja (dane po "Podaj imię:" wpisuje użytkownik):
// Podaj imię: Ala
// Cześć, Ala!
// (na stderr: [debug] odczytano wejście)
```

Aby czytać wejście linia po linii aż do końca (np. przy przekierowaniu pliku na wejście `dart program.dart < plik.txt`), transformujemy bajtowy strumień `stdin` dekoderem UTF-8 i `LineSplitter`.

```dart
import 'dart:convert';
import 'dart:io';

Future<void> main() async {
  var numer = 1;
  // stdin to Stream<List<int>> — dekodujemy bajty i dzielimy na linie.
  final linie = stdin.transform(utf8.decoder).transform(const LineSplitter());

  await for (final linia in linie) {
    stdout.writeln('$numer: $linia'); // numerujemy każdą wczytaną linię
    numer++;
  }
}
// Przykład użycia: echo -e "raz\ndwa" | dart program.dart
// Oczekiwane wyjście:
// 1: raz
// 2: dwa
```

---

## 6. `Process` — uruchamianie zewnętrznych programów

Klasa `Process` pozwala uruchamiać zewnętrzne programy systemowe. Dostępne są dwa główne warianty:

- `Process.run` — uruchamia program, **czeka na jego zakończenie** i zwraca `ProcessResult` z całym wyjściem (`stdout`, `stderr`) oraz kodem zakończenia (`exitCode`).
- `Process.start` — uruchamia program i zwraca uchwyt `Process`, przez który strumieniowo czytamy wyjście w czasie rzeczywistym (przydatne dla długo działających procesów).

### 6.1 `Process.run` — przechwytywanie wyjścia i kodu zakończenia

Poniższy przykład uruchamia `echo`, przechwytuje jego `stdout` i `stderr` oraz sprawdza kod zakończenia. Kod `0` oznacza sukces, wartości niezerowe — błąd.

```dart
import 'dart:io';

Future<void> main() async {
  // Uruchamiamy program 'echo' z argumentem. runInShell nie jest potrzebne.
  final wynik = await Process.run('echo', ['Witaj z procesu']);

  // ProcessResult zawiera exitCode, stdout i stderr.
  print('Kod zakończenia: ${wynik.exitCode}'); // 0 = sukces
  print('stdout: ${(wynik.stdout as String).trim()}');
  print('stderr: "${(wynik.stderr as String).trim()}"');

  // Sprawdzenie sukcesu na podstawie kodu zakończenia.
  if (wynik.exitCode == 0) {
    print('Proces zakończył się poprawnie.');
  } else {
    print('Proces zakończył się błędem.');
  }
}
// Oczekiwane wyjście:
// Kod zakończenia: 0
// stdout: Witaj z procesu
// stderr: ""
// Proces zakończył się poprawnie.
```

Poniższy przykład pokazuje proces kończący się **niezerowym** kodem oraz wypisujący na `stderr`. Uruchamiamy interpreter powłoki z komendą, która pisze na stderr i zwraca kod 3.

```dart
import 'dart:io';

Future<void> main() async {
  // sh -c pozwala uruchomić komendę powłoki; celowo zwracamy kod 3.
  final wynik = await Process.run('sh', ['-c', 'echo "błąd!" 1>&2; exit 3']);

  print('Kod zakończenia: ${wynik.exitCode}'); // 3 = błąd
  print('Treść stderr: ${(wynik.stderr as String).trim()}');

  if (wynik.exitCode != 0) {
    // Reagujemy na niezerowy kod zakończenia.
    print('Wykryto niepowodzenie procesu (kod ${wynik.exitCode}).');
  }
}
// Oczekiwane wyjście:
// Kod zakończenia: 3
// Treść stderr: błąd!
// Wykryto niepowodzenie procesu (kod 3).
```

### 6.2 `Process.start` — strumieniowanie wyjścia

Dla procesów, które produkują wyjście stopniowo, używamy `Process.start` i słuchamy strumieni `stdout`/`stderr` na bieżąco. `exitCode` jest tu `Future`, które kończy się, gdy proces zakończy działanie — **niekoniecznie** gdy cały jego `stdout` zostanie już odebrany i przetworzony. Aby zagwarantować kolejność wypisywanych linii, czekamy dodatkowo na zakończenie przetwarzania strumienia (np. przez `forEach`, które zwraca `Future` kończące się po jego wyczerpaniu).

```dart
import 'dart:convert';
import 'dart:io';

Future<void> main() async {
  // start zwraca uchwyt procesu; stdout/stderr są strumieniami bajtów.
  final proces = await Process.start('sh', ['-c', 'echo linia1; echo linia2']);

  // Dekodujemy i dzielimy strumień stdout na linie w czasie rzeczywistym.
  // forEach zwraca Future, które kończy się dopiero po wyczerpaniu strumienia.
  final przetworzoneWyjscie = proces.stdout
      .transform(utf8.decoder)
      .transform(const LineSplitter())
      .forEach((linia) => print('OUT> $linia'));

  // exitCode to Future — czekamy na zakończenie procesu.
  final kod = await proces.exitCode;
  // Dodatkowo czekamy, aż cały stdout zostanie odebrany i wypisany — inaczej
  // kolejność linii względem komunikatu końcowego nie byłaby gwarantowana.
  await przetworzoneWyjscie;
  print('Proces zakończony, kod: $kod');
}
// Oczekiwane wyjście (kolejność linii OUT> zachowana):
// OUT> linia1
// OUT> linia2
// Proces zakończony, kod: 0
```

---

## 7. `HttpServer` — prosty serwer HTTP

`HttpServer.bind` uruchamia serwer nasłuchujący na wskazanym adresie i porcie. Serwer jest `Stream<HttpRequest>` — każde przychodzące żądanie to jeden element strumienia. Dla każdego żądania badamy metodę (`request.method`) i ścieżkę (`request.uri.path`), a odpowiedź piszemy do `request.response`.

Poniższy serwer nasłuchuje na porcie `8080`, obsługuje żądania `GET` i zwraca odpowiedź tekstową. Po napisaniu odpowiedzi **musimy** zamknąć `response` przez `close()`.

```dart
import 'dart:io';

Future<void> main() async {
  // bind uruchamia serwer na localhost:8080.
  final serwer = await HttpServer.bind(InternetAddress.loopbackIPv4, 8080);
  print('Serwer nasłuchuje na http://${serwer.address.host}:${serwer.port}');

  await for (final zadanie in serwer) {
    final odpowiedz = zadanie.response;

    if (zadanie.method == 'GET' && zadanie.uri.path == '/') {
      // Ustawiamy typ zawartości i status odpowiedzi.
      odpowiedz
        ..statusCode = HttpStatus.ok
        ..headers.contentType = ContentType.text
        ..write('Witaj! Żądanie GET ${zadanie.uri.path} obsłużone.');
    } else {
      // Dla innych ścieżek/metod zwracamy 404.
      odpowiedz
        ..statusCode = HttpStatus.notFound
        ..write('Nie znaleziono: ${zadanie.uri.path}');
    }

    // close() wysyła odpowiedź do klienta i zwalnia połączenie.
    await odpowiedz.close();
  }
}
// Po uruchomieniu, w innym terminalu:
//   curl http://127.0.0.1:8080/
// Oczekiwana odpowiedź:
//   Witaj! Żądanie GET / obsłużone.
```

---

## 8. `HttpClient` — klient HTTP

`HttpClient` pozwala wykonywać żądania HTTP z poziomu aplikacji Dart. Sekwencja jest trzyetapowa: otwarcie żądania (`getUrl`) zwraca `HttpClientRequest`, zamknięcie żądania (`close`) zwraca `HttpClientResponse`, a odpowiedź czytamy jako strumień bajtów. Po zakończeniu zamykamy klienta metodą `close`.

```dart
import 'dart:convert';
import 'dart:io';

Future<void> main() async {
  final klient = HttpClient();
  try {
    // getUrl przygotowuje żądanie GET; można ustawić nagłówki przed close().
    final zadanie = await klient.getUrl(Uri.parse('http://example.com/'));
    // close() wysyła żądanie i zwraca odpowiedź.
    final odpowiedz = await zadanie.close();

    print('Status: ${odpowiedz.statusCode}');
    // Treść odpowiedzi to strumień bajtów — dekodujemy do String przez UTF-8.
    final tresc = await odpowiedz.transform(utf8.decoder).join();
    print('Długość odpowiedzi: ${tresc.length} znaków');
  } on SocketException catch (e) {
    // Brak sieci lub nieosiągalny host rzuca SocketException.
    print('Błąd połączenia: ${e.message}');
  } finally {
    klient.close(); // zwalniamy zasoby klienta
  }
}
// Przykładowe wyjście (wymaga dostępu do sieci):
// Status: 200
// Długość odpowiedzi: 1256 znaków
```

---

## 9. `Socket` i `ServerSocket` — surowe połączenia TCP

Gdy potrzebujemy protokołu niższego poziomu niż HTTP, używamy gniazd TCP. `ServerSocket.bind` uruchamia serwer nasłuchujący połączeń, a `Socket.connect` łączy się z serwerem. Każde połączenie to `Socket` — jednocześnie strumień przychodzących bajtów i miejsce do zapisu bajtów wychodzących.

Poniższy przykład uruchamia serwer echo (odsyła to, co otrzyma) i łączy się z nim klientem — wszystko w jednym programie dla ilustracji.

```dart
import 'dart:convert';
import 'dart:io';

Future<void> main() async {
  // Serwer TCP nasłuchujący na losowym wolnym porcie (0 => wybór systemu).
  final serwer = await ServerSocket.bind(InternetAddress.loopbackIPv4, 0);
  print('Serwer TCP na porcie ${serwer.port}');

  // Obsługa połączeń w tle: każdemu klientowi odsyłamy odebrane dane (echo).
  serwer.listen((klientSocket) {
    klientSocket.listen((bajty) {
      final tekst = utf8.decode(bajty);
      klientSocket.write('echo: $tekst'); // odsyłamy z prefiksem
      klientSocket.close();
    });
  });

  // Klient łączy się z serwerem i wysyła wiadomość.
  final klient = await Socket.connect(InternetAddress.loopbackIPv4, serwer.port);
  klient.write('ping');

  // Socket to Stream<Uint8List>; dekodujemy bajty przez utf8.decoder.bind.
  final odpowiedz = await utf8.decoder.bind(klient).join();
  print('Odpowiedź serwera: $odpowiedz');

  await klient.close();
  await serwer.close();
}
// Oczekiwane wyjście (numer portu jest losowy):
// Serwer TCP na porcie XXXXX
// Odpowiedź serwera: echo: ping
```

---

## 10. `Platform` — wykrywanie środowiska

Klasa `Platform` udostępnia informacje o środowisku uruchomieniowym: system operacyjny, zmienne środowiskowe, ścieżkę interpretera Dart, liczbę rdzeni procesora i argumenty wiersza poleceń. Wszystkie te właściwości są **statyczne**.

```dart
import 'dart:io';

void main() {
  // Wykrywanie systemu operacyjnego — przydatne dla kodu wieloplatformowego.
  print('System operacyjny: ${Platform.operatingSystem}'); // np. linux, macos, windows
  print('Wersja systemu: ${Platform.operatingSystemVersion}');
  print('Czy Linux? ${Platform.isLinux}');

  // Liczba rdzeni procesora — pomocne przy doborze liczby isolate'ów.
  print('Liczba rdzeni: ${Platform.numberOfProcessors}');

  // Ścieżka do interpretera Dart, który uruchomił program.
  print('Executable: ${Platform.resolvedExecutable}');

  // Separator ścieżek zależny od systemu ('/' na Unix, '\\' na Windows).
  print('Separator ścieżek: "${Platform.pathSeparator}"');

  // Odczyt zmiennej środowiskowej (mapa String -> String).
  final home = Platform.environment['HOME'] ?? Platform.environment['USERPROFILE'];
  print('Katalog domowy: $home');
}
// Przykładowe wyjście (wartości zależą od maszyny):
// System operacyjny: linux
// Wersja systemu: ...
// Czy Linux? true
// Liczba rdzeni: 8
// Executable: /usr/lib/dart/bin/dart
// Separator ścieżek: "/"
// Katalog domowy: /home/user
```

---

## Ćwiczenie 1: Narzędzie przetwarzające plik

### Opis problemu

Napisz funkcję `Future<void> przetworzPlik(String wejscie, String wyjscie)`, która wczyta plik tekstowy `wejscie`, przekształci jego zawartość i zapisze wynik do pliku `wyjscie`. Transformacja polega na:

1. usunięciu pustych linii,
2. zamianie każdej linii na wielkie litery (`toUpperCase`),
3. dodaniu przed każdą linią jej numeru w formacie `N: treść`.

Funkcja ma obsłużyć przypadek nieistniejącego pliku wejściowego — wtedy wypisuje komunikat o błędzie i nie tworzy pliku wyjściowego.

### Przykłady wejścia/wyjścia

| Wejście (zawartość pliku) | Oczekiwane wyjście (zawartość pliku wynikowego) |
|---------------------------|-------------------------------------------------|
| `"ala\n\nma kota\n"` | `"1: ALA\n2: MA KOTA\n"` |
| `"jeden\ndwa\ntrzy\n"` | `"1: JEDEN\n2: DWA\n3: TRZY\n"` |

### Wskazówki

1. Użyj `readAsLines()` do wczytania linii oraz `writeAsString` do zapisu wyniku.
2. Odfiltruj puste linie przez `where((l) => l.trim().isNotEmpty)`.
3. Do numeracji przydaje się `asMap().entries` lub ręczny licznik; pamiętaj, że numeracja zaczyna się od 1.
4. Opakuj odczyt w `try`/`catch` łapiący `FileSystemException`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:io';

Future<void> przetworzPlik(String wejscie, String wyjscie) async {
  final plikWe = File(wejscie);
  try {
    // Wczytujemy wszystkie linie (rzuci FileSystemException, gdy brak pliku).
    final linie = await plikWe.readAsLines();

    // 1. usuwamy puste linie, 2. wielkie litery, 3. numerujemy od 1.
    final przetworzone = linie
        .where((l) => l.trim().isNotEmpty)
        .map((l) => l.toUpperCase())
        .toList();

    final bufor = StringBuffer();
    for (var i = 0; i < przetworzone.length; i++) {
      bufor.writeln('${i + 1}: ${przetworzone[i]}');
    }

    // Zapisujemy wynik do pliku wyjściowego.
    await File(wyjscie).writeAsString(bufor.toString());
    print('Zapisano ${przetworzone.length} linii do $wyjscie');
  } on FileSystemException catch (e) {
    // Nie tworzymy pliku wyjściowego, gdy wejście jest niedostępne.
    print('Błąd: nie udało się przetworzyć pliku wejściowego: ${e.message}');
  }
}

Future<void> main() async {
  final we = '${Directory.systemTemp.path}/we.txt';
  final wy = '${Directory.systemTemp.path}/wy.txt';

  await File(we).writeAsString('ala\n\nma kota\n');
  await przetworzPlik(we, wy);
  print(await File(wy).readAsString());
  // 1: ALA
  // 2: MA KOTA

  // Przypadek błędu: nieistniejący plik.
  await przetworzPlik('/brak/pliku.txt', wy);

  await File(we).delete();
  await File(wy).delete();
}
```

</details>

---

## Ćwiczenie 2: Serwer HTTP z licznikiem odwiedzin

### Opis problemu

Zbuduj serwer HTTP nasłuchujący na porcie `8090`, który obsługuje żądania `GET`:

- ścieżka `/` — zwraca tekst `Witaj! Odwiedziny nr N`, gdzie `N` to numer kolejnego żądania (licznik rośnie z każdym żądaniem),
- ścieżka `/health` — zwraca tekst `OK` ze statusem `200`,
- każda inna ścieżka — zwraca status `404` i tekst `Nie znaleziono`.

Napisz funkcję `Future<HttpServer> uruchomSerwer(int port)`, która uruchamia serwer i zwraca jego uchwyt (aby dało się go potem zamknąć w teście).

### Przykłady wejścia/wyjścia

| Wejście (żądanie) | Oczekiwane wyjście (treść odpowiedzi) |
|-------------------|---------------------------------------|
| `GET /` (pierwsze) | `Witaj! Odwiedziny nr 1` |
| `GET /health` | `OK` |

### Wskazówki

1. `HttpServer.bind(InternetAddress.loopbackIPv4, port)` uruchamia serwer.
2. Trzymaj licznik w zmiennej poza pętlą `await for` i inkrementuj go dla ścieżki `/`.
3. Rozróżniaj ścieżki przez `request.uri.path` i metodę przez `request.method`.
4. Zawsze wywołuj `response.close()` po zapisaniu treści.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:convert';
import 'dart:io';

Future<HttpServer> uruchomSerwer(int port) async {
  final serwer = await HttpServer.bind(InternetAddress.loopbackIPv4, port);
  var licznik = 0; // stan współdzielony między żądaniami

  // Obsługa żądań w tle — nie blokujemy zwrócenia uchwytu serwera.
  serwer.listen((zadanie) async {
    final odpowiedz = zadanie.response;
    odpowiedz.headers.contentType = ContentType.text;

    if (zadanie.method == 'GET' && zadanie.uri.path == '/') {
      licznik++;
      odpowiedz
        ..statusCode = HttpStatus.ok
        ..write('Witaj! Odwiedziny nr $licznik');
    } else if (zadanie.method == 'GET' && zadanie.uri.path == '/health') {
      odpowiedz
        ..statusCode = HttpStatus.ok
        ..write('OK');
    } else {
      odpowiedz
        ..statusCode = HttpStatus.notFound
        ..write('Nie znaleziono');
    }

    await odpowiedz.close();
  });

  return serwer;
}

Future<void> main() async {
  final serwer = await uruchomSerwer(8090);
  final klient = HttpClient();

  // Test: dwa żądania do '/' oraz jedno do '/health'.
  for (final sciezka in ['/', '/', '/health']) {
    final zadanie = await klient.getUrl(Uri.parse('http://127.0.0.1:8090$sciezka'));
    final odpowiedz = await zadanie.close();
    final tresc = await odpowiedz.transform(utf8.decoder).join();
    print('$sciezka -> $tresc');
  }

  klient.close();
  await serwer.close();
}
// Oczekiwane wyjście:
// / -> Witaj! Odwiedziny nr 1
// / -> Witaj! Odwiedziny nr 2
// /health -> OK
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **`dart:io` łączy Dart z systemem operacyjnym** — plikami, procesami i siecią — i działa tylko w aplikacjach konsolowych/serwerowych, nie w przeglądarce.
2. **Większość operacji ma wariant asynchroniczny i synchroniczny (`Sync`).** W kodzie serwerowym preferuj asynchroniczne, aby nie blokować pętli zdarzeń; wersje `Sync` nadają się do prostych skryptów CLI.
3. **Operacje na plikach mogą zawieść — zawsze łap `FileSystemException`** (plik nie istnieje, brak uprawnień). Pole `osError` daje szczegóły z systemu operacyjnego.
4. **`Process.run` czeka na zakończenie i zwraca `ProcessResult`** z `stdout`, `stderr` i `exitCode`; `Process.start` pozwala strumieniowo czytać wyjście długo działających procesów. Kod `0` to sukces, wartości niezerowe to błąd.
5. **`HttpServer` jest strumieniem żądań, `Socket`/`ServerSocket` obsługują surowy TCP.** Dla żądań HTTP pamiętaj o zamknięciu odpowiedzi przez `response.close()`.
6. **Klasa `Platform` udostępnia informacje o środowisku** — system operacyjny, zmienne środowiskowe, ścieżkę interpretera i liczbę rdzeni — co pozwala pisać kod wieloplatformowy i dobierać liczbę isolate'ów.

---

**Następny moduł:** [dart:convert](05-dart-convert.md)
**Poprzedni moduł:** [dart:async](03-dart-async.md)
