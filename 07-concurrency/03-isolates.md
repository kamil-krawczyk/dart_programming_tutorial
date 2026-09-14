---
id: "7.3"
title: "Isolates i współbieżność"
difficulty: "advanced"
section: "07-concurrency"
prerequisites:
  - "Futures i async/await"
  - "Streams"
---

# 7.3 Isolates i współbieżność

## Informacje o module

- **Poziom trudności:** advanced
- **Wymagania wstępne:** [Futures i async/await](01-async-await.md), [Streams](02-streams.md)
- **Cele nauki:**
  1. Zrozumieć jednowątkowy model wykonania Dart oraz koncepcję izolacji pamięci pomiędzy isolate'ami.
  2. Uruchamiać zadania na osobnych isolate'ach za pomocą `Isolate.run` i `Isolate.spawn`, komunikować się przez `SendPort`/`ReceivePort` oraz zarządzać cyklem życia isolate'a.
  3. Projektować dwukierunkowe wzorce komunikacji (żądanie-odpowiedź, strumieniowanie, pula workerów), obsługiwać błędy isolate'ów oraz decydować, kiedy używać isolate'ów, a kiedy wystarczy `async`/`await`.

---

## 1. Jednowątkowy model wykonania Dart

Kod Dart wykonuje się w ramach **isolate'a**. Isolate to niezależna jednostka wykonawcza posiadająca **własną pamięć (stertę)** oraz **własną pętlę zdarzeń** (event loop). Każdy program Dart startuje z jednym isolate'em — nazywamy go *isolate'em głównym* (main isolate).

Kluczowa właściwość: w obrębie pojedynczego isolate'a kod jest **jednowątkowy**. W danej chwili wykonuje się dokładnie jedna operacja synchroniczna. Konstrukcje `async`/`await` oraz `Future` nie tworzą nowych wątków — jedynie przeplatają zadania w tej samej pętli zdarzeń. Oznacza to, że operacja I/O (np. odczyt z sieci) nie blokuje isolate'a, ale **ciężkie obliczenia CPU-bound blokują** całą pętlę zdarzeń, bo nie ma miejsca, w którym pętla mogłaby "wejść" między kroki obliczeń.

Isolate'y nie współdzielą pamięci. Zamiast tego komunikują się wyłącznie przez **przekazywanie wiadomości** (message passing). Dzięki temu w Dart nie występują klasyczne wyścigi (data races) czy potrzeba blokad (mutexów) na współdzielonej pamięci — bo współdzielonej pamięci po prostu nie ma.

Poniższy przykład pokazuje, że ciężkie obliczenie synchroniczne blokuje pętlę zdarzeń: zaplanowany `Future` nie ma szansy wykonać się w trakcie pętli `for`.

```dart
import 'dart:async';

// Ciężkie obliczenie CPU-bound wykonywane synchronicznie na głównym isolate'ie.
int sumaKwadratow(int n) {
  var suma = 0;
  for (var i = 1; i <= n; i++) {
    suma += i * i; // synchronicznie — bez punktu "await"
  }
  return suma;
}

void main() {
  // Future zaplanowany na kolejce zdarzeń — wykona się dopiero, gdy pętla zdarzeń
  // uzyska kontrolę z powrotem, czyli PO zakończeniu obliczeń poniżej.
  Future(() => print('Future: pętla zdarzeń jest znów wolna'));

  print('Start obliczeń');
  final wynik = sumaKwadratow(100000000);
  print('Wynik: $wynik'); // blokuje isolate do momentu zakończenia
}
// Oczekiwane wyjście (kolejność):
// Start obliczeń
// Wynik: 672921401752298880 (suma matematyczna 333333338333333350000000
// przepełnia 64-bitowy int i "zawija się" — Dart nie rzuca tu wyjątku)
// Future: pętla zdarzeń jest znów wolna
```

Drugi przykład pokazuje, że nawet oznaczenie funkcji jako `async` **nie pomaga** — dopóki nie ma realnego `await` na asynchronicznej operacji, ciężka pętla nadal blokuje isolate.

```dart
import 'dart:async';

// `async` samo w sobie nie tworzy wątku ani nie zwalnia pętli zdarzeń.
Future<int> policzAsync(int n) async {
  var suma = 0;
  for (var i = 1; i <= n; i++) {
    suma += i; // brak await w środku — pętla nadal blokuje isolate
  }
  return suma;
}

Future<void> main() async {
  Timer(Duration.zero, () => print('Timer: chciałem się wykonać wcześniej'));
  final wynik = await policzAsync(50000000);
  print('Suma: $wynik');
}
// Oczekiwane wyjście (kolejność):
// Suma: 1250000025000000
// Timer: chciałem się wykonać wcześniej
```

---

## 2. Izolacja pamięci — co można, a czego nie można przesłać

Ponieważ isolate'y mają rozdzielone sterty, wiadomość wysyłana między nimi jest **kopiowana** (dla większości typów). Dart pozwala przesyłać:

- typy proste: `null`, `bool`, `int`, `double`, `String`,
- kolekcje tych typów: `List`, `Map`, `Set` (rekurencyjnie, o ile ich zawartość jest przesyłalna),
- `SendPort` (uchwyt do portu innego isolate'a),
- `TransferableTypedData` (dane binarne przekazywane bez kopiowania),
- niektóre obiekty niemodyfikowalne (np. instancje typów oznaczonych jako głęboko niezmienne).

Czego **nie można** przesłać:

- obiektów zawierających stan natywny lub zamknięcia domknięć nad zmiennym stanem (`Function`/domknięcia w trybie spawn),
- `ReceivePort` (przesyłamy tylko powiązany z nim `SendPort`),
- obiektów typu `Socket`, `HttpClient` i podobnych zasobów systemowych.

Próba wysłania nieprzesyłalnego obiektu kończy się wyjątkiem `ArgumentError` w czasie wykonania.

Poniższy przykład pokazuje przesyłanie zwykłych, przesyłalnych danych oraz `SendPort`.

```dart
import 'dart:isolate';

void main() async {
  final odbiorczy = ReceivePort();
  // Przesyłalne: mapa z prostymi typami oraz SendPort.
  final wiadomosc = {
    'id': 1,
    'nazwa': 'zamówienie',
    'pozycje': ['A', 'B', 'C'],
    'odpowiedz': odbiorczy.sendPort, // SendPort JEST przesyłalny
  };
  // Wysyłamy wiadomość samą do siebie dla demonstracji.
  odbiorczy.sendPort.send(wiadomosc);

  final odebrane = await odbiorczy.first as Map;
  print('Odebrano: ${odebrane['nazwa']}, pozycje=${odebrane['pozycje']}');
  odbiorczy.close();
}
// Oczekiwane wyjście:
// Odebrano: zamówienie, pozycje=[A, B, C]
```

Dla dużych buforów binarnych `TransferableTypedData` przekazuje własność bufora **bez kopiowania** (zero-copy), co jest znacznie tańsze niż kopiowanie megabajtów danych. Po wywołaniu `materialize()` bufor "przenosi się" do odbiorcy, a nadawca nie powinien go już używać.

```dart
import 'dart:isolate';
import 'dart:typed_data';

void main() async {
  final port = ReceivePort();

  final bufor = Uint8List.fromList(List.filled(8, 42));
  // Owijamy bufor — przekazanie własności zamiast kopiowania bajtów.
  final transfer = TransferableTypedData.fromList([bufor]);
  port.sendPort.send(transfer);

  final odebrany = await port.first as TransferableTypedData;
  // materialize() zwraca ByteBuffer po stronie odbiorcy.
  final dane = odebrany.materialize().asUint8List();
  print('Odebrano ${dane.length} bajtów, pierwszy=${dane.first}');
  port.close();
}
// Oczekiwane wyjście:
// Odebrano 8 bajtów, pierwszy=42
```

---

## 3. Uruchamianie isolate'ów: `Isolate.run` i `Isolate.spawn`

### 3.1 `Isolate.run` — proste zadanie jednorazowe

`Isolate.run` (dodane w Dart 2.19) to najprostszy sposób na odciążenie CPU. Uruchamia funkcję na nowym isolate'ie, zwraca `Future` z jej wynikiem i automatycznie zamyka isolate po zakończeniu. Idealne do pojedynczego, kosztownego obliczenia.

```dart
import 'dart:isolate';

// Funkcja wykonywana na osobnym isolate'ie — musi być funkcją najwyższego poziomu
// lub statyczną, aby dało się ją przekazać do innego isolate'a.
int silnia(int n) {
  var wynik = 1;
  for (var i = 2; i <= n; i++) {
    wynik *= i;
  }
  return wynik;
}

Future<void> main() async {
  // Isolate.run uruchamia obliczenie poza głównym isolate'em i zwraca wynik.
  final wynik = await Isolate.run(() => silnia(10));
  print('10! = $wynik');
}
// Oczekiwane wyjście:
// 10! = 3628800
```

`Isolate.run` przyjmuje domknięcie, ale przechwycone w nim wartości muszą być przesyłalne. Poniżej przekazujemy dane wejściowe do isolate'a przez zmienną domknięcia.

```dart
import 'dart:isolate';

Future<void> main() async {
  final liczby = List<int>.generate(1000, (i) => i + 1);

  // Domknięcie przechwytuje `liczby` (przesyłalną listę int) i liczy sumę
  // na osobnym isolate'ie.
  final suma = await Isolate.run(() {
    return liczby.fold<int>(0, (acc, x) => acc + x);
  });

  print('Suma 1..1000 = $suma');
}
// Oczekiwane wyjście:
// Suma 1..1000 = 500500
```

### 3.2 `Isolate.spawn` — pełna kontrola

`Isolate.spawn` daje pełną kontrolę: uruchamia funkcję wejściową (`entry point`), której przekazujemy jeden argument (najczęściej `SendPort`). Isolate żyje tak długo, jak długo ma coś do zrobienia lub dopóki go nie zabijemy. Ten mechanizm jest podstawą długożyjących workerów.

```dart
import 'dart:isolate';

// Punkt wejścia isolate'a: odbiera SendPort, na który odeśle wynik.
void wejscie(SendPort doGlownego) {
  final wynik = List<int>.generate(20, (i) => i * i);
  doGlownego.send(wynik); // odsyłamy dane do isolate'a głównego
}

Future<void> main() async {
  final odbiorczy = ReceivePort();
  // spawn uruchamia funkcję `wejscie`, przekazując jej port do komunikacji.
  await Isolate.spawn(wejscie, odbiorczy.sendPort);

  final wynik = await odbiorczy.first as List<int>;
  print('Kwadraty: ${wynik.take(5).toList()} ...');
  odbiorczy.close(); // zamykamy port, gdy nie jest już potrzebny
}
// Oczekiwane wyjście:
// Kwadraty: [0, 1, 4, 9, 16] ...
```

---

## 4. Komunikacja przez `SendPort` / `ReceivePort`

`ReceivePort` to strumień przychodzących wiadomości; posiada powiązany `SendPort`, który przekazujemy drugiemu isolate'owi. `SendPort.send` jest jednokierunkowy. Aby uzyskać komunikację **dwukierunkową**, każdy isolate tworzy własny `ReceivePort` i wymienia się `SendPort`-ami — ten wzorzec nazywa się *handshake*.

```dart
import 'dart:async';
import 'dart:isolate';

void worker(SendPort doGlownego) {
  final wlasnyPort = ReceivePort();
  // Krok 1 handshake: worker odsyła swój SendPort isolate'owi głównemu.
  doGlownego.send(wlasnyPort.sendPort);

  wlasnyPort.listen((wiadomosc) {
    // Odpowiadamy na przychodzące wiadomości.
    doGlownego.send('echo: $wiadomosc');
  });
}

Future<void> main() async {
  final odGlownego = ReceivePort();
  await Isolate.spawn(worker, odGlownego.sendPort);

  // Przekształcamy strumień portu w kolejkę, aby wygodnie czytać po kolei.
  final kolejka = StreamIterator(odGlownego);

  await kolejka.moveNext();
  final doWorkera = kolejka.current as SendPort; // odebrany SendPort workera

  doWorkera.send('cześć');
  await kolejka.moveNext();
  print(kolejka.current); // echo: cześć

  await kolejka.cancel();
  odGlownego.close();
}
// Oczekiwane wyjście:
// echo: cześć
```

---

## 5. Wzorce komunikacji dwukierunkowej

### 5.1 Żądanie–odpowiedź (request-response)

Najczęstszy wzorzec: wysyłamy żądanie i czekamy na dokładnie jedną odpowiedź. Aby dopasować odpowiedzi do żądań, do każdego żądania dołączamy jednorazowy `SendPort` (utworzony z tymczasowego `ReceivePort`).

```dart
import 'dart:isolate';

// Worker liczy kwadrat i odsyła wynik na dołączony port odpowiedzi.
void kalkulator(SendPort doGlownego) {
  final port = ReceivePort();
  doGlownego.send(port.sendPort);

  port.listen((wiadomosc) {
    final (int liczba, SendPort odpowiedz) = wiadomosc as (int, SendPort);
    odpowiedz.send(liczba * liczba); // jedna odpowiedź na jedno żądanie
  });
}

Future<int> zapytaj(SendPort worker, int liczba) async {
  final odpowiedz = ReceivePort();
  worker.send((liczba, odpowiedz.sendPort)); // dołączamy port odpowiedzi
  final wynik = await odpowiedz.first as int;
  odpowiedz.close();
  return wynik;
}

Future<void> main() async {
  final start = ReceivePort();
  await Isolate.spawn(kalkulator, start.sendPort);
  final worker = await start.first as SendPort;
  start.close();

  print('3^2 = ${await zapytaj(worker, 3)}');
  print('7^2 = ${await zapytaj(worker, 7)}');
}
// Oczekiwane wyjście:
// 3^2 = 9
// 7^2 = 49
```

### 5.2 Ciągłe strumieniowanie (continuous streaming)

Worker może generować wiele wiadomości w czasie, a isolate główny konsumuje je jako strumień. `ReceivePort` sam jest `Stream`, więc możemy po prostu na nim `listen`.

```dart
import 'dart:isolate';

// Worker wysyła strumień wartości, a na końcu znacznik zakończenia.
void generator(SendPort doGlownego) {
  for (var i = 1; i <= 5; i++) {
    doGlownego.send(i * 10);
  }
  doGlownego.send('DONE'); // sygnał końca strumienia
}

Future<void> main() async {
  final port = ReceivePort();
  await Isolate.spawn(generator, port.sendPort);

  await for (final wiadomosc in port) {
    if (wiadomosc == 'DONE') break; // koniec — przerywamy nasłuch
    print('Otrzymano: $wiadomosc');
  }
  port.close();
}
// Oczekiwane wyjście:
// Otrzymano: 10
// Otrzymano: 20
// Otrzymano: 30
// Otrzymano: 40
// Otrzymano: 50
```

### 5.3 Koordynacja wielu workerów (multi-worker)

Możemy uruchomić kilka isolate'ów naraz, rozdzielić między nie pracę i zebrać wyniki. Poniżej rozdzielamy listę na fragmenty, każdy fragment liczy osobny worker, a wyniki agregujemy.

```dart
import 'dart:isolate';

// Każdy worker sumuje przydzielony mu fragment listy.
void sumator(List<Object> argumenty) {
  final dane = argumenty[0] as List<int>;
  final odpowiedz = argumenty[1] as SendPort;
  odpowiedz.send(dane.fold<int>(0, (a, b) => a + b));
}

Future<void> main() async {
  final dane = List<int>.generate(100, (i) => i + 1); // 1..100
  const liczbaWorkerow = 4;
  final rozmiar = dane.length ~/ liczbaWorkerow;

  final wyniki = <Future<int>>[];
  for (var w = 0; w < liczbaWorkerow; w++) {
    final port = ReceivePort();
    final fragment = dane.sublist(w * rozmiar, (w + 1) * rozmiar);
    await Isolate.spawn(sumator, [fragment, port.sendPort]);
    // Każdy port da jedną wartość — zbieramy Future z każdego workera.
    wyniki.add(port.first.then((v) {
      port.close();
      return v as int;
    }));
  }

  final czesciowe = await Future.wait(wyniki); // czekamy na wszystkich naraz
  final suma = czesciowe.fold<int>(0, (a, b) => a + b);
  print('Suma 1..100 = $suma (fragmenty: $czesciowe)');
}
// Oczekiwane wyjście:
// Suma 1..100 = 5050 (fragmenty: [325, 950, 1575, 2200])
```

---

## 6. Przykład wydajnościowy — pomiar czasu przed i po odciążeniu

Poniższy przykład mierzy, jak długo blokowany jest isolate główny podczas ciężkiego obliczenia wykonywanego **lokalnie**, a następnie to samo obliczenie odciąża na osobny isolate przez `Isolate.run`. Podczas wersji odciążonej isolate główny pozostaje responsywny (co pokazujemy licznikiem "tików").

```dart
import 'dart:async';
import 'dart:isolate';

// Celowo kosztowne obliczenie CPU-bound.
int ciezkieObliczenie(int n) {
  var suma = 0;
  for (var i = 0; i < n; i++) {
    suma += (i % 7) * (i % 13);
  }
  return suma;
}

Future<void> main() async {
  const n = 200000000;

  // --- Wariant 1: obliczenie na głównym isolate'ie (blokuje pętlę zdarzeń) ---
  final zegar1 = Stopwatch()..start();
  final wynik1 = ciezkieObliczenie(n);
  zegar1.stop();
  print('Lokalnie: wynik=$wynik1, czas=${zegar1.elapsedMilliseconds} ms '
      '(pętla zdarzeń była zablokowana)');

  // --- Wariant 2: odciążenie na osobny isolate (główny pozostaje wolny) ---
  var tiki = 0;
  final licznik = Timer.periodic(
    const Duration(milliseconds: 50),
    (_) => tiki++, // ten timer działa tylko, gdy pętla zdarzeń jest wolna
  );

  final zegar2 = Stopwatch()..start();
  final wynik2 = await Isolate.run(() => ciezkieObliczenie(n));
  zegar2.stop();
  licznik.cancel();

  print('Odciążone: wynik=$wynik2, czas=${zegar2.elapsedMilliseconds} ms, '
      'tików timera podczas pracy=$tiki (>0 oznacza responsywność)');
}
// Przykładowe wyjście (liczby czasu i tików zależą od maszyny):
// Lokalnie: wynik=..., czas=~900 ms (pętla zdarzeń była zablokowana)
// Odciążone: wynik=..., czas=~950 ms, tików timera podczas pracy=18 (>0 oznacza responsywność)
```

Wniosek: całkowity czas obliczenia jest podobny (a nawet nieco większy z powodu narzutu na uruchomienie isolate'a i kopiowanie wyniku), ale w wariancie odciążonym **isolate główny pozostaje responsywny** — timer się wykonuje, UI by się nie zaciął.

---

## 7. Zarządzanie cyklem życia isolate'a

`Isolate.spawn` zwraca obiekt `Isolate`, który pozwala kontrolować jego cykl życia:

- `pause()` / `resume(Capability)` — wstrzymanie i wznowienie przetwarzania,
- `kill(priority: Isolate.immediate)` — natychmiastowe zakończenie,
- `addOnExitListener(SendPort)` — powiadomienie o zakończeniu isolate'a,
- `addErrorListener(SendPort)` — powiadomienie o nieobsłużonym błędzie.

Poniższy przykład pokazuje `addOnExitListener` oraz `kill` — po zabiciu workera dostajemy komunikat o zakończeniu.

```dart
import 'dart:async';
import 'dart:isolate';

// Długożyjący worker: nasłuchuje w nieskończoność, aż zostanie zabity.
void worker(SendPort doGlownego) {
  final port = ReceivePort();
  doGlownego.send(port.sendPort);
  port.listen((w) => doGlownego.send('przetworzono: $w'));
}

Future<void> main() async {
  final odGlownego = ReceivePort();
  final exitPort = ReceivePort();

  final isolate = await Isolate.spawn(worker, odGlownego.sendPort);
  // Rejestrujemy nasłuch zakończenia — dostaniemy wiadomość, gdy worker zniknie.
  isolate.addOnExitListener(exitPort.sendPort);

  final kolejka = StreamIterator(odGlownego);
  await kolejka.moveNext();
  final doWorkera = kolejka.current as SendPort;

  doWorkera.send('A');
  await kolejka.moveNext();
  print(kolejka.current); // przetworzono: A

  // Kończymy isolate natychmiast — zwalniamy jego zasoby.
  isolate.kill(priority: Isolate.immediate);
  await exitPort.first; // czekamy na potwierdzenie zakończenia
  print('Isolate zakończony');

  await kolejka.cancel();
  odGlownego.close();
  exitPort.close();
}
// Oczekiwane wyjście:
// przetworzono: A
// Isolate zakończony
```

---

## 8. Obsługa błędów isolate'ów

Nieobsłużony wyjątek w spawnowanym isolate'ie domyślnie **nie** przerywa isolate'a głównego — po prostu ten worker kończy pracę. Aby dowiedzieć się o błędzie, rejestrujemy `addErrorListener`, który dostaje wiadomość `[opisBłędu, stackTrace]`. Domyślnie isolate umiera po nieobsłużonym błędzie; można to zmienić parametrem `errorsAreFatal` w `Isolate.spawn`.

```dart
import 'dart:isolate';

// Worker celowo rzuca wyjątek po odebraniu wiadomości.
void wadliwyWorker(SendPort doGlownego) {
  final port = ReceivePort();
  doGlownego.send(port.sendPort);
  port.listen((_) {
    throw StateError('Coś poszło nie tak w workerze');
  });
}

Future<void> main() async {
  final odGlownego = ReceivePort();
  final bledy = ReceivePort();

  final isolate = await Isolate.spawn(
    wadliwyWorker,
    odGlownego.sendPort,
    paused: true, // startujemy wstrzymani, aby zdążyć podłączyć nasłuch błędów
  );
  // Przekazanie błędów do isolate'a głównego zamiast cichego zakończenia.
  isolate.addErrorListener(bledy.sendPort);
  isolate.resume(isolate.pauseCapability!);

  final doWorkera = await odGlownego.first as SendPort;
  doWorkera.send('start');

  // Błąd przychodzi jako lista [komunikat, stackTrace].
  final blad = await bledy.first as List;
  print('Przechwycono błąd z workera: ${blad[0]}');

  isolate.kill(priority: Isolate.immediate);
  odGlownego.close();
  bledy.close();
}
// Oczekiwane wyjście:
// Przechwycono błąd z workera: Bad state: Coś poszło nie tak w workerze
```

Dla eleganckiego zamknięcia (graceful shutdown) łączymy nasłuch błędów z nasłuchem zakończenia: gdy przyjdzie błąd, sprzątamy zasoby i zamykamy porty, zamiast pozwolić aplikacji działać w niespójnym stanie.

```dart
import 'dart:async';
import 'dart:isolate';

void worker(SendPort doGlownego) {
  final port = ReceivePort();
  doGlownego.send(port.sendPort);
  port.listen((w) {
    if (w == 'awaria') throw Exception('awaria workera');
    doGlownego.send('ok: $w');
  });
}

Future<void> main() async {
  final odGlownego = ReceivePort();
  final bledy = ReceivePort();
  final wyjscie = ReceivePort();

  final isolate = await Isolate.spawn(worker, odGlownego.sendPort, paused: true);
  isolate.addErrorListener(bledy.sendPort);
  isolate.addOnExitListener(wyjscie.sendPort);
  isolate.resume(isolate.pauseCapability!);

  final kolejka = StreamIterator(odGlownego);
  await kolejka.moveNext();
  final doWorkera = kolejka.current as SendPort;

  // Nasłuchujemy błędu w tle i wykonujemy graceful shutdown.
  bledy.listen((blad) {
    print('Błąd -> sprzątanie zasobów: ${(blad as List)[0]}');
    isolate.kill(priority: Isolate.immediate);
  });

  doWorkera.send('zadanie1');
  await kolejka.moveNext();
  print(kolejka.current); // ok: zadanie1

  doWorkera.send('awaria'); // wywoła błąd i zamknięcie
  await wyjscie.first; // czekamy aż isolate faktycznie zniknie
  print('Zamknięto czysto');

  await kolejka.cancel();
  odGlownego.close();
  bledy.close();
  wyjscie.close();
}
// Oczekiwane wyjście (kolejność dwóch środkowych linii może się przeplatać):
// ok: zadanie1
// Błąd -> sprzątanie zasobów: Exception: awaria workera
// Zamknięto czysto
```

---

## 9. Porównanie: `Isolate.run` vs długożyjące isolate'y

Wybór między prostym zadaniem jednorazowym a długożyjącym workerem zależy od charakteru pracy.

| Kryterium | `Isolate.run` (jednorazowe) | Długożyjący isolate (`Isolate.spawn`) |
|-----------|-----------------------------|----------------------------------------|
| **Zastosowania** | Pojedyncze, kosztowne obliczenie (parsowanie dużego JSON, kompresja, obliczenie matematyczne) | Powtarzalna praca, pula workerów, serwer przetwarzający wiele żądań, ciągłe strumieniowanie |
| **Narzut startu** | Ponoszony przy każdym wywołaniu (start + zamknięcie isolate'a) | Ponoszony raz; kolejne żądania są tanie |
| **Koszt pamięci** | Isolate zwalniany od razu po zakończeniu | Isolate zajmuje pamięć przez cały czas życia |
| **Złożoność komunikacji** | Minimalna — argument wejściowy i wynik jako `Future` | Wyższa — ręczne porty, handshake, protokół wiadomości |
| **Zarządzanie cyklem życia** | Automatyczne — brak sprzątania | Ręczne — trzeba pamiętać o `kill`, zamykaniu portów, obsłudze błędów |

Zasada praktyczna: jeśli robisz coś **raz**, użyj `Isolate.run`. Jeśli to samo obliczenie wykonujesz **wielokrotnie** i narzut startu isolate'a zaczyna dominować, zbuduj długożyjącego workera lub pulę workerów.

---

## 10. Kryteria decyzji: isolate'y vs `async`/`await`

To najważniejsza decyzja projektowa w kodzie współbieżnym Dart.

**Zadania I/O-bound (sieć, dysk, baza danych) → `async`/`await`.**
Operacje I/O i tak są nieblokujące — czekanie na odpowiedź sieci nie zużywa CPU. Pętla zdarzeń w tym czasie obsługuje inne zadania. Tworzenie isolate'a dla operacji I/O to niepotrzebny narzut (kopiowanie danych + start isolate'a) bez żadnego zysku.

**Zadania CPU-bound (ciężkie obliczenia, parsowanie, przetwarzanie obrazów) → isolate.**
Takie zadania blokują pętlę zdarzeń, bo faktycznie zajmują procesor. Bez isolate'a zablokują UI lub obsługę innych żądań. Odciążenie na osobny isolate pozwala wykonać je równolegle na innym rdzeniu.

**Uwzględnij narzut uruchomienia.**
Start isolate'a i kopiowanie danych mają swój koszt. Dla bardzo krótkich obliczeń (kilka milisekund) narzut może przewyższyć zysk — wtedy `async`/`await` (a nawet kod synchroniczny) jest wystarczające. Isolate opłaca się, gdy praca jest na tyle duża, że jej czas znacząco przewyższa narzut startu.

**Kiedy `async`/`await` wystarcza:**
- czekasz na I/O (pliki, HTTP, DB),
- przeplatasz wiele operacji asynchronicznych,
- obliczenia są krótkie i nie blokują zauważalnie pętli zdarzeń.

**Kiedy potrzebujesz isolate'a:**
- ciężkie obliczenia CPU-bound zacinają UI lub opóźniają obsługę żądań,
- chcesz wykorzystać wiele rdzeni procesora do równoległości,
- masz długo działające zadanie w tle, które nie może blokować głównej pętli.

---

## Ćwiczenie 1: Równoległe obliczenie sumy kwadratów

### Opis problemu

Napisz funkcję `Future<int> sumaKwadratowRownolegle(int n)`, która obliczy sumę kwadratów liczb od `1` do `n`, dzieląc pracę na 2 isolate'y (pierwsza połowa zakresu i druga połowa), a następnie sumując wyniki cząstkowe. Użyj `Isolate.run`.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `n = 5` | `55` |
| `n = 10` | `385` |

### Wskazówki

1. Suma kwadratów `1²+2²+...+k²`. Podziel zakres na `[1, n~/2]` oraz `[n~/2 + 1, n]`.
2. Uruchom dwa `Isolate.run` i poczekaj na oba przez `Future.wait`, potem dodaj wyniki.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:isolate';

// Liczy sumę kwadratów w domkniętym zakresie [od, do].
int sumaKwadratowZakres(int od, int do_) {
  var suma = 0;
  for (var i = od; i <= do_; i++) {
    suma += i * i;
  }
  return suma;
}

Future<int> sumaKwadratowRownolegle(int n) async {
  if (n < 1) return 0;
  final srodek = n ~/ 2;

  // Dwa niezależne isolate'y liczące połowy zakresu.
  final wyniki = await Future.wait([
    Isolate.run(() => sumaKwadratowZakres(1, srodek)),
    Isolate.run(() => sumaKwadratowZakres(srodek + 1, n)),
  ]);

  return wyniki[0] + wyniki[1];
}

Future<void> main() async {
  print(await sumaKwadratowRownolegle(5)); // 55
  print(await sumaKwadratowRownolegle(10)); // 385
}
```

</details>

---

## Ćwiczenie 2: Pula workerów (worker pool)

### Opis problemu

Zaimplementuj prostą pulę workerów. Utwórz `N` długożyjących isolate'ów, które przyjmują liczbę i odsyłają jej sześcian. Klasa `PulaWorkerow` powinna udostępniać metodę `Future<int> policz(int x)`, która wybiera dowolnego wolnego workera (dla uproszczenia — cyklicznie, round-robin) i zwraca wynik. Na koniec metoda `zamknij()` kończy wszystkie isolate'y.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `policz(3)` | `27` |
| `policz(4)` | `64` |

### Wskazówki

1. Każdy worker: `ReceivePort` po swojej stronie, handshake `SendPort`, potem `listen` z odpowiedzią na dołączony port odpowiedzi (jak w sekcji 5.1).
2. `PulaWorkerow` trzyma listę `SendPort` workerów i indeks round-robin inkrementowany modulo `N`.
3. `zamknij()` powinno wywołać `kill` na każdym isolate'ie i pozamykać porty.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:isolate';

// Punkt wejścia workera: odbiera żądania (liczba, portOdpowiedzi) i odsyła sześcian.
void _worker(SendPort doGlownego) {
  final port = ReceivePort();
  doGlownego.send(port.sendPort);
  port.listen((wiadomosc) {
    final (int x, SendPort odpowiedz) = wiadomosc as (int, SendPort);
    odpowiedz.send(x * x * x);
  });
}

class PulaWorkerow {
  final List<Isolate> _isolaty = [];
  final List<SendPort> _porty = [];
  var _nastepny = 0;

  // Fabryka asynchroniczna tworzy i inicjalizuje pulę.
  static Future<PulaWorkerow> utworz(int rozmiar) async {
    final pula = PulaWorkerow();
    for (var i = 0; i < rozmiar; i++) {
      final odbiorczy = ReceivePort();
      final isolate = await Isolate.spawn(_worker, odbiorczy.sendPort);
      final port = await odbiorczy.first as SendPort;
      odbiorczy.close();
      pula._isolaty.add(isolate);
      pula._porty.add(port);
    }
    return pula;
  }

  Future<int> policz(int x) async {
    // Wybór workera metodą round-robin.
    final worker = _porty[_nastepny];
    _nastepny = (_nastepny + 1) % _porty.length;

    final odpowiedz = ReceivePort();
    worker.send((x, odpowiedz.sendPort));
    final wynik = await odpowiedz.first as int;
    odpowiedz.close();
    return wynik;
  }

  void zamknij() {
    for (final isolate in _isolaty) {
      isolate.kill(priority: Isolate.immediate);
    }
    _isolaty.clear();
    _porty.clear();
  }
}

Future<void> main() async {
  final pula = await PulaWorkerow.utworz(3);
  print(await pula.policz(3)); // 27
  print(await pula.policz(4)); // 64
  print(await Future.wait([pula.policz(2), pula.policz(5)])); // [8, 125]
  pula.zamknij();
}
```

</details>

---

## Ćwiczenie 3: Obsługa błędów isolate'a z bezpiecznym wynikiem

### Opis problemu

Napisz funkcję `Future<int> bezpieczneDzielenie(int a, int b)`, która wykona dzielenie `a ~/ b` na osobnym isolate'ie. Jeśli isolate rzuci błąd (np. dzielenie przez zero), funkcja ma przechwycić ten błąd i zwrócić `-1` zamiast przerwać program. Wykorzystaj `Isolate.spawn` z `addErrorListener` (lub przechwyt błędu z `Isolate.run`).

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| `a = 10, b = 2` | `5` |
| `a = 10, b = 0` | `-1` |

### Wskazówki

1. Najprościej: `Isolate.run` opakuj w `try`/`catch` — błędy z uruchomionego isolate'a są przekazywane jako wyjątek `Future`.
2. Dzielenie całkowite przez zero (`~/ 0`) rzuca `UnsupportedError` / `IntegerDivisionByZeroException`.
3. W bloku `catch` zwróć `-1`.

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
import 'dart:isolate';

Future<int> bezpieczneDzielenie(int a, int b) async {
  try {
    // Błąd rzucony w isolate'ie propaguje się jako wyjątek tego Future.
    return await Isolate.run(() => a ~/ b);
  } catch (e) {
    // Przechwytujemy dowolny błąd (np. dzielenie przez zero) i zwracamy wartość awaryjną.
    return -1;
  }
}

Future<void> main() async {
  print(await bezpieczneDzielenie(10, 2)); // 5
  print(await bezpieczneDzielenie(10, 0)); // -1
}
```

</details>

---

## Podsumowanie

### Kluczowe wnioski

1. **Kod Dart jest jednowątkowy w obrębie isolate'a.** `async`/`await` przeplata zadania w jednej pętli zdarzeń, ale nie tworzy wątków — ciężkie obliczenia CPU-bound blokują całą pętlę.
2. **Isolate'y nie współdzielą pamięci** i komunikują się wyłącznie przez przekazywanie wiadomości. Większość wiadomości jest kopiowana; `TransferableTypedData` pozwala przenieść duże bufory binarne bez kopiowania.
3. **`Isolate.run` to najprostsza opcja** dla pojedynczego kosztownego obliczenia — automatycznie zwraca wynik i sprząta isolate. `Isolate.spawn` daje pełną kontrolę i jest podstawą długożyjących workerów.
4. **Komunikacja dwukierunkowa opiera się na wymianie `SendPort`-ów** (handshake). Popularne wzorce to żądanie-odpowiedź, ciągłe strumieniowanie oraz koordynacja wielu workerów.
5. **Zarządzaj cyklem życia i błędami jawnie:** używaj `kill`, `addOnExitListener`, `addErrorListener`, aby sprzątać zasoby i wykonywać graceful shutdown po awarii.
6. **Wybieraj isolate'y dla zadań CPU-bound, a `async`/`await` dla I/O-bound.** Uwzględnij narzut startu isolate'a — dla krótkich zadań może przewyższyć zysk z równoległości.

---

**Następny moduł:** [Obsługa błędów async](04-error-handling-async.md)
**Poprzedni moduł:** [Streams](02-streams.md)
