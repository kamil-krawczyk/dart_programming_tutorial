---
id: "4.3"
title: "Domknięcia i funkcje wyższego rzędu"
difficulty: "intermediate"
section: "04-functions"
prerequisites:
  - "Deklaracje funkcji i parametry"
  - "Lambdy i typedef"
---

# 4.3 Domknięcia i funkcje wyższego rzędu

## Informacje o module

- **Poziom trudności:** intermediate
- **Wymagania wstępne:** [Deklaracje funkcji i parametry](01-declarations-params.md), [Lambdy i typedef](02-lambdas-typedef.md)
- **Cele nauki:**
  1. Rozumieć mechanizm domknięć (closures) i leksykalnego zasięgu zmiennych w Dart
  2. Tworzyć i wykorzystywać funkcje wyższego rzędu (higher-order functions) do budowania elastycznego kodu
  3. Stosować domknięcia do enkapsulacji stanu mutowalnego i tworzenia fabryk funkcji

---

## Funkcje jako obiekty pierwszej klasy

W Dart funkcje są obiektami pierwszej klasy (first-class citizens). Oznacza to, że można je przypisywać do zmiennych, przekazywać jako argumenty i zwracać z innych funkcji — dokładnie tak samo jak wartości typu `int` czy `String`.

Poniższy przykład ilustruje trzy podstawowe sposoby traktowania funkcji jako wartości:

```dart
// Nazwana funkcja — może być przypisana do zmiennej
int dodaj(int a, int b) => a + b;
int pomnoz(int a, int b) => a * b;

void main() {
  // 1. Przypisanie funkcji do zmiennej
  var operacja = dodaj;
  print(operacja(3, 4)); // 7

  // 2. Zmiana przypisania na inną funkcję o tym samym typie
  operacja = pomnoz;
  print(operacja(3, 4)); // 12

  // 3. Przechowywanie w kolekcji
  var funkcje = <String, int Function(int, int)>{
    '+': dodaj,
    '*': pomnoz,
    '-': (a, b) => a - b, // lambda również jest wartością
  };

  print(funkcje['-']!(10, 3)); // 7
}
// Oczekiwane wyjście:
// 7
// 12
// 7
```

Funkcje mogą być przekazywane jako argumenty do innych funkcji — jest to kluczowy mechanizm pozwalający na tworzenie elastycznych abstakcji:

```dart
void wykonajDwaRazy(void Function(String) akcja, String wiadomosc) {
  akcja(wiadomosc);
  akcja(wiadomosc);
}

void krzycz(String tekst) => print(tekst.toUpperCase());
void szeptaj(String tekst) => print(tekst.toLowerCase());

void main() {
  // Przekazanie nazwanej funkcji
  wykonajDwaRazy(krzycz, 'Hej');

  // Przekazanie lambdy
  wykonajDwaRazy((msg) => print('>>> $msg <<<'), 'Dart');
}
// Oczekiwane wyjście:
// HEJ
// HEJ
// >>> Dart <<<
// >>> Dart <<<
```

---

## Domknięcia (closures)

Domknięcie (closure) to funkcja, która „przechwytuje" zmienne z otaczającego ją zakresu leksykalnego (lexical scope). Nawet po zakończeniu funkcji zewnętrznej, domknięcie zachowuje dostęp do przechwyconych zmiennych.

### Zasięg leksykalny (lexical scoping)

Dart używa zasięgu leksykalnego — funkcja widzi zmienne zdefiniowane w zakresie, w którym została utworzona, niezależnie od tego, gdzie zostanie później wywołana:

```dart
// Funkcja zwraca lambdę, więc jej typ zwracany to String Function(String)
String Function(String) stworzPowitanie(String powitanie) {
  // Zmienna 'powitanie' jest przechwycona przez zwróconą lambdę
  return (String imie) => '$powitanie, $imie!';
}

void main() {
  // Każde wywołanie stworzPowitanie tworzy nowe domknięcie
  // z własną kopią zmiennej 'powitanie'
  var czesc = stworzPowitanie('Cześć');
  var hej = stworzPowitanie('Hej');

  // Domknięcia pamiętają 'powitanie' z momentu utworzenia
  print(czesc('Anna'));   // Cześć, Anna!
  print(czesc('Jan'));    // Cześć, Jan!
  print(hej('Anna'));     // Hej, Anna!
}
// Oczekiwane wyjście:
// Cześć, Anna!
// Cześć, Jan!
// Hej, Anna!
```

Bardziej rozbudowany przykład zagnieżdżonego zasięgu leksykalnego — każdy poziom zagnieżdżenia widzi zmienne wszystkich poziomów nadrzędnych:

```dart
void main() {
  var zewnetrzna = 'Poziom 1';

  var funkcjaA = () {
    var wewnetrznaA = 'Poziom 2A';

    var funkcjaB = () {
      var wewnetrznaB = 'Poziom 3';
      // Widzi zmienne ze wszystkich nadrzędnych zakresów
      print('$zewnetrzna > $wewnetrznaA > $wewnetrznaB');
    };

    funkcjaB();
    // print(wewnetrznaB); // Błąd — wewnetrznaB nie jest widoczna tutaj
  };

  funkcjaA();
  // print(wewnetrznaA); // Błąd — wewnetrznaA nie jest widoczna tutaj
}
// Oczekiwane wyjście:
// Poziom 1 > Poziom 2A > Poziom 3
```

### Przechwytywanie zmiennych mutowalnych (mutable state capture)

Domknięcia przechwytują **referencje** do zmiennych, nie ich kopie. Oznacza to, że jeśli zmienna jest modyfikowana, domknięcie widzi nową wartość — i odwrotnie, domknięcie może modyfikować przechwyconą zmienną:

```dart
Function stworzLicznik(int startOd) {
  var wartosc = startOd; // zmienna mutowalna przechwycona przez domknięcie

  return () {
    wartosc++; // domknięcie modyfikuje przechwyconą zmienną
    return wartosc;
  };
}

void main() {
  var licznik1 = stworzLicznik(0);
  var licznik2 = stworzLicznik(100);

  // Każde domknięcie ma własną niezależną kopię 'wartosc'
  print(licznik1()); // 1
  print(licznik1()); // 2
  print(licznik1()); // 3

  print(licznik2()); // 101 — niezależny stan
  print(licznik2()); // 102
}
// Oczekiwane wyjście:
// 1
// 2
// 3
// 101
// 102
```

Praktyczny przykład enkapsulacji stanu — stworzenie prostego systemu limitów wywołań:

```dart
typedef AkcjaLimitowana = bool Function();

AkcjaLimitowana stworzZLimitem(int maxWywolan, void Function() akcja) {
  var pozostalo = maxWywolan; // mutowalny stan przechwycony przez domknięcie

  return () {
    if (pozostalo > 0) {
      pozostalo--;
      akcja();
      return true; // akcja wykonana
    }
    return false; // limit wyczerpany
  };
}

void main() {
  var limitowana = stworzZLimitem(3, () => print('Wykonano akcję!'));

  print(limitowana()); // true — zostało 2
  print(limitowana()); // true — zostało 1
  print(limitowana()); // true — zostało 0
  print(limitowana()); // false — limit wyczerpany
  print(limitowana()); // false
}
// Oczekiwane wyjście:
// Wykonano akcję!
// true
// Wykonano akcję!
// true
// Wykonano akcję!
// true
// false
// false
```

### Pułapka: domknięcia w pętlach

Częsty błąd polega na tworzeniu domknięć w pętli, które przechwytują zmienną iteracyjną — w Dart `for(var i...)` tworzy nową zmienną w każdej iteracji, więc problem ten nie występuje tak jak w niektórych innych językach:

```dart
void main() {
  var funkcje = <Function>[];

  // W Dart każda iteracja for tworzy nową zmienną 'i'
  for (var i = 0; i < 3; i++) {
    funkcje.add(() => print('i = $i'));
  }

  // Każde domknięcie przechwytuje swoją własną kopię 'i'
  for (var f in funkcje) {
    f();
  }
}
// Oczekiwane wyjście:
// i = 0
// i = 1
// i = 2
```

Porównanie z sytuacją, gdy zmienna jest współdzielona (zadeklarowana poza pętlą):

```dart
void main() {
  var funkcje = <Function>[];
  var j = 0; // jedna zmienna współdzielona przez wszystkie domknięcia

  while (j < 3) {
    // Wszystkie domknięcia przechwytują tę samą zmienną 'j'
    var kopia = j; // rozwiązanie: lokalna kopia
    funkcje.add(() => print('kopia=$kopia, j=$j'));
    j++;
  }

  for (var f in funkcje) {
    f(); // j ma teraz wartość 3 we wszystkich domknięciach
  }
}
// Oczekiwane wyjście:
// kopia=0, j=3
// kopia=1, j=3
// kopia=2, j=3
```

---

## Funkcje wyższego rzędu (higher-order functions)

Funkcja wyższego rzędu (higher-order function, HOF) to funkcja, która przyjmuje inną funkcję jako argument lub zwraca funkcję jako wynik. HOF są fundamentalnym narzędziem programowania funkcyjnego.

### HOF przyjmujące funkcje jako argumenty

Najczęściej spotykane HOF to metody kolekcji takie jak `map`, `where`, `reduce`, `fold` i `forEach`:

```dart
void main() {
  var liczby = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

  // where — filtruje elementy spełniające warunek
  var parzyste = liczby.where((n) => n % 2 == 0).toList();
  print('Parzyste: $parzyste');

  // map — transformuje każdy element
  var kwadraty = liczby.map((n) => n * n).toList();
  print('Kwadraty: $kwadraty');

  // reduce — redukuje listę do jednej wartości
  var suma = liczby.reduce((akumulator, element) => akumulator + element);
  print('Suma: $suma');

  // fold — jak reduce, ale z wartością początkową
  var iloczyn = liczby.fold(1, (acc, el) => acc * el);
  print('Iloczyn: $iloczyn');
}
// Oczekiwane wyjście:
// Parzyste: [2, 4, 6, 8, 10]
// Kwadraty: [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
// Suma: 55
// Iloczyn: 3628800
```

Tworzenie własnych HOF daje możliwość budowania reużywalnych abstrakcji:

```dart
// HOF: wykonuje operację na liście i mierzy czas
List<T> zmierz<T>(String nazwa, List<T> Function() operacja) {
  var start = DateTime.now();
  var wynik = operacja();
  var czas = DateTime.now().difference(start);
  print('$nazwa: ${czas.inMicroseconds}μs (${wynik.length} elementów)');
  return wynik;
}

// HOF: tworzy filtr łączący wiele predykatów operatorem AND
bool Function(T) polaczFiltry<T>(List<bool Function(T)> filtry) {
  return (T element) => filtry.every((filtr) => filtr(element));
}

void main() {
  var dane = List.generate(1000, (i) => i);

  var filtr = polaczFiltry<int>([
    (n) => n % 2 == 0,    // parzyste
    (n) => n % 3 == 0,    // podzielne przez 3
    (n) => n > 100,        // większe niż 100
  ]);

  var wynik = zmierz('Filtrowanie', () => dane.where(filtr).toList());
  print('Pierwszych 5: ${wynik.take(5).toList()}');
}
// Oczekiwane wyjście:
// Filtrowanie: (czas w μs) (150 elementów)
// Pierwszych 5: [102, 108, 114, 120, 126]
```

### HOF zwracające funkcje (fabryki funkcji)

Funkcja wyższego rzędu może zwracać nową funkcję — ten wzorzec jest szczególnie przydatny do tworzenia wyspecjalizowanych wariantów ogólnej logiki:

```dart
// Fabryka walidatorów — zwraca funkcję walidującą
String? Function(String) stworzWalidatorDlugosci(int min, int max) {
  return (String wartosc) {
    if (wartosc.length < min) return 'Za krótkie (min: $min znaków)';
    if (wartosc.length > max) return 'Za długie (max: $max znaków)';
    return null; // null = brak błędu
  };
}

// Fabryka transformacji — komponuje listę transformacji w jedną
String Function(String) stworzPipeline(List<String Function(String)> kroki) {
  return (String wejscie) {
    var wynik = wejscie;
    for (var krok in kroki) {
      wynik = krok(wynik);
    }
    return wynik;
  };
}

void main() {
  // Tworzenie wyspecjalizowanych walidatorów
  var walidujHaslo = stworzWalidatorDlugosci(8, 32);
  var walidujNazwe = stworzWalidatorDlugosci(2, 50);

  print(walidujHaslo('abc'));          // Za krótkie (min: 8 znaków)
  print(walidujHaslo('bezpieczne1'));  // null — OK
  print(walidujNazwe('A'));            // Za krótkie (min: 2 znaków)

  // Tworzenie pipeline'u transformacji
  var normalizuj = stworzPipeline([
    (s) => s.trim(),
    (s) => s.toLowerCase(),
    (s) => s.replaceAll(RegExp(r'\s+'), ' '),
  ]);

  print(normalizuj('  Hello   WORLD  '));
}
// Oczekiwane wyjście:
// Za krótkie (min: 8 znaków)
// null
// Za krótkie (min: 2 znaków)
// hello world
```

### Kompozycja funkcji

Kompozycja funkcji pozwala łączyć proste funkcje w bardziej złożone operacje:

```dart
// Generyczna kompozycja dwóch funkcji: (f ∘ g)(x) = f(g(x))
B Function(A) skomponuj<A, B, C>(B Function(C) f, C Function(A) g) {
  return (A x) => f(g(x));
}

// Kompozycja listy funkcji tego samego typu
T Function(T) skomponujWszystkie<T>(List<T Function(T)> funkcje) {
  return (T x) {
    var wynik = x;
    for (var f in funkcje) {
      wynik = f(wynik);
    }
    return wynik;
  };
}

void main() {
  // Proste funkcje do kompozycji
  int podwoj(int n) => n * 2;
  String naString(int n) => 'Wynik: $n';

  // Kompozycja: najpierw podwoj, potem naString
  var podwojIWyswietl = skomponuj<int, String, int>(naString, podwoj);
  print(podwojIWyswietl(5)); // Wynik: 10

  // Kompozycja wielu transformacji int → int
  var transformuj = skomponujWszystkie<int>([
    (n) => n * 2,     // 5 → 10
    (n) => n + 3,     // 10 → 13
    (n) => n * n,     // 13 → 169
  ]);

  print(transformuj(5)); // 169
}
// Oczekiwane wyjście:
// Wynik: 10
// 169
```

---

## Praktyczne wzorce z domknięciami i HOF

### Memoizacja (cache wyników)

Domknięcia idealnie nadają się do implementacji wzorca memoizacji — zapamiętywania wyników kosztownych obliczeń:

```dart
// Generyczna memoizacja dla funkcji jednoargumentowych
R Function(T) memoize<T, R>(R Function(T) funkcja) {
  var cache = <T, R>{}; // domknięcie przechwytuje cache

  return (T argument) {
    if (cache.containsKey(argument)) {
      print('  [cache hit] $argument');
      return cache[argument] as R;
    }
    print('  [obliczam] $argument');
    var wynik = funkcja(argument);
    cache[argument] = wynik;
    return wynik;
  };
}

void main() {
  // Kosztowna operacja
  int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
  }

  var memoFib = memoize(fibonacci);

  print('fib(10) = ${memoFib(10)}');
  print('fib(10) = ${memoFib(10)}'); // drugie wywołanie z cache
  print('fib(8) = ${memoFib(8)}');
}
// Oczekiwane wyjście:
//   [obliczam] 10
// fib(10) = 55
//   [cache hit] 10
// fib(10) = 55
//   [obliczam] 8
// fib(8) = 21
```

### Middleware / dekoratory

HOF pozwalają na tworzenie dekoratorów (wrapper functions) dodających zachowanie do istniejących funkcji:

```dart
typedef Handler = String Function(String request);

// Dekorator: logowanie wywołań
Handler zLogowaniem(Handler handler) {
  return (String request) {
    print('[LOG] Żądanie: $request');
    var odpowiedz = handler(request);
    print('[LOG] Odpowiedź: $odpowiedz');
    return odpowiedz;
  };
}

// Dekorator: walidacja wejścia
Handler zWalidacja(Handler handler) {
  return (String request) {
    if (request.isEmpty) return 'BŁĄD: puste żądanie';
    return handler(request);
  };
}

String obsluz(String request) => 'OK: $request';

void main() {
  // Kompozycja dekoratorów — każdy dodaje warstwę zachowania
  var handler = zLogowaniem(zWalidacja(obsluz));

  print(handler('GET /users'));
  print('---');
  print(handler(''));
}
// Oczekiwane wyjście:
// [LOG] Żądanie: GET /users
// [LOG] Odpowiedź: OK: GET /users
// OK: GET /users
// ---
// [LOG] Żądanie: 
// [LOG] Odpowiedź: BŁĄD: puste żądanie
// BŁĄD: puste żądanie
```

---

## Ćwiczenie 1 (intermediate)

### Opis problemu

Zaimplementuj funkcję `stworzAkumulator`, która przyjmuje wartość początkową (`double`) i zwraca funkcję. Zwrócona funkcja przyjmuje liczbę (`double`), dodaje ją do wewnętrznego stanu i zwraca aktualną sumę. Dodatkowo zaimplementuj funkcję `stworzHistorie`, która opakowuje akumulator i przechowuje historię wszystkich wartości pośrednich. Funkcja `stworzHistorie` zwraca record z dwoma polami: funkcją akumulującą i funkcją zwracającą historię.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| akumulator(0): dodaj 5, dodaj 3, dodaj -2 | `5.0\n8.0\n6.0` |
| historia(10): dodaj 5, dodaj 2; pobierz historię | `15.0\n17.0\nHistoria: [10.0, 15.0, 17.0]` |

### Wskazówki

1. Domknięcie może przechwycić zmienną `var suma` i modyfikować ją przy każdym wywołaniu
2. Dla historii użyj listy `List<double>` przechwyconej przez domknięcie
3. Dart 3 records pozwalają zwrócić dwie funkcje: `({double Function(double) dodaj, List<double> Function() historia})`

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// Prosty akumulator z domknięciem
double Function(double) stworzAkumulator(double poczatkowa) {
  var suma = poczatkowa;
  return (double wartosc) {
    suma += wartosc;
    return suma;
  };
}

// Akumulator z historią — zwraca record z dwoma funkcjami
({double Function(double) dodaj, List<double> Function() historia})
    stworzHistorie(double poczatkowa) {
  var suma = poczatkowa;
  var log = <double>[poczatkowa];

  return (
    dodaj: (double wartosc) {
      suma += wartosc;
      log.add(suma);
      return suma;
    },
    historia: () => List.unmodifiable(log),
  );
}

void main() {
  // Prosty akumulator
  var akumulator = stworzAkumulator(0);
  print(akumulator(5));   // 5.0
  print(akumulator(3));   // 8.0
  print(akumulator(-2));  // 6.0

  // Akumulator z historią
  var h = stworzHistorie(10);
  print(h.dodaj(5));       // 15.0
  print(h.dodaj(2));       // 17.0
  print('Historia: ${h.historia()}'); // Historia: [10.0, 15.0, 17.0]
}
```

</details>

---

## Ćwiczenie 2 (advanced)

### Opis problemu

Zaimplementuj mini-framework do przetwarzania danych oparty na HOF. Stwórz:
1. Funkcję `pipeline<T>` przyjmującą listę transformacji `T Function(T)` i zwracającą jedną skompozowaną funkcję
2. Funkcję `retry<T>` przyjmującą funkcję `T Function()` i liczbę prób — zwraca funkcję, która ponawia wywołanie do N razy w przypadku wyjątku
3. Funkcję `batch<T, R>` przyjmującą funkcję `R Function(T)` i rozmiar porcji — zwraca funkcję przetwarzającą listę elementów w partiach

Przetestuj na przetwarzaniu listy stringów: normalizacja → filtrowanie → transformacja, z obsługą błędów i przetwarzaniem w partiach.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| pipeline: trim → uppercase; dane: `['  hello ', ' world  ']` | `[HELLO, WORLD]` |
| retry(3): rzuca 2x wyjątek, za 3. razem zwraca `'ok'` | `Próba 1: błąd\nPróba 2: błąd\nPróba 3: sukces\nok` |
| batch(2): przetwarzaj `[1,2,3,4,5]` w partiach po 2 | `Partia 1: [1, 2]\nPartia 2: [3, 4]\nPartia 3: [5]` |

### Wskazówki

1. `pipeline` to sekwencyjna kompozycja — zastosuj `fold` na liście transformacji
2. `retry` używa pętli `for` z `try-catch` — domknięcie przechowuje licznik prób
3. `batch` używa `sublist` do wycinania porcji z listy wejściowej
4. Użyj generics aby framework był reużywalny dla różnych typów

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
// Kompozycja listy transformacji w jedną funkcję
T Function(T) pipeline<T>(List<T Function(T)> transformacje) {
  return (T wejscie) => transformacje.fold(wejscie, (acc, t) => t(acc));
}

// Ponawianie funkcji w przypadku wyjątku
T Function() retry<T>(T Function() funkcja, int maxProb) {
  return () {
    for (var proba = 1; proba <= maxProb; proba++) {
      try {
        var wynik = funkcja();
        print('Próba $proba: sukces');
        return wynik;
      } catch (e) {
        print('Próba $proba: błąd');
        if (proba == maxProb) rethrow;
      }
    }
    throw StateError('Nieosiągalne');
  };
}

// Przetwarzanie listy w partiach
List<R> Function(List<T>) batch<T, R>(R Function(T) przetworz, int rozmiar) {
  return (List<T> dane) {
    var wyniki = <R>[];
    for (var i = 0; i < dane.length; i += rozmiar) {
      var koniec = (i + rozmiar > dane.length) ? dane.length : i + rozmiar;
      var partia = dane.sublist(i, koniec);
      print('Partia ${i ~/ rozmiar + 1}: $partia');
      for (var element in partia) {
        wyniki.add(przetworz(element));
      }
    }
    return wyniki;
  };
}

void main() {
  // 1. Pipeline
  var normalizuj = pipeline<String>([
    (s) => s.trim(),
    (s) => s.toUpperCase(),
  ]);
  var dane = ['  hello ', ' world  '];
  print(dane.map(normalizuj).toList());

  // 2. Retry
  var licznik = 0;
  var operacja = retry<String>(() {
    licznik++;
    if (licznik < 3) throw Exception('Błąd tymczasowy');
    return 'ok';
  }, 3);
  print(operacja());

  // 3. Batch
  var przetwarzaj = batch<int, int>((n) => n * 10, 2);
  var wynik = przetwarzaj([1, 2, 3, 4, 5]);
  print('Wynik: $wynik');
}
```

</details>

---

## Ćwiczenie 3 (intermediate)

### Opis problemu

Napisz system zdarzeń (event bus) oparty na domknięciach. Stwórz funkcję `stworzEventBus`, która zwraca record z trzema metodami:
- `on(String event, void Function(dynamic) handler)` — rejestruje handler dla danego zdarzenia
- `emit(String event, dynamic data)` — emituje zdarzenie do wszystkich zarejestrowanych handlerów
- `off(String event)` — usuwa wszystkie handlery dla danego zdarzenia

Przetestuj na scenariuszu z wieloma listenerami i różnymi zdarzeniami.

### Przykłady wejścia/wyjścia

| Wejście | Oczekiwane wyjście |
|---------|-------------------|
| on('login', print), on('login', upperPrint), emit('login', 'Anna') | `Zalogowano: Anna\nZALOGOWANO: ANNA` |
| on('msg', print), emit('msg', 'hi'), off('msg'), emit('msg', 'ho') | `Wiadomość: hi` (po off — brak wyjścia) |

### Wskazówki

1. Użyj `Map<String, List<void Function(dynamic)>>` jako wewnętrznego stanu przechwyconego przez domknięcie
2. `on` dodaje handler do listy pod kluczem zdarzenia
3. `emit` iteruje po handlerach i wywołuje każdy z przekazanymi danymi
4. `off` czyści listę handlerów dla danego klucza

<details>
<summary>Rozwiązanie referencyjne</summary>

```dart
({
  void Function(String, void Function(dynamic)) on,
  void Function(String, dynamic) emit,
  void Function(String) off,
}) stworzEventBus() {
  var handlers = <String, List<void Function(dynamic)>>{};

  return (
    on: (String event, void Function(dynamic) handler) {
      handlers.putIfAbsent(event, () => []);
      handlers[event]!.add(handler);
    },
    emit: (String event, dynamic data) {
      var lista = handlers[event];
      if (lista != null) {
        for (var handler in lista) {
          handler(data);
        }
      }
    },
    off: (String event) {
      handlers.remove(event);
    },
  );
}

void main() {
  var bus = stworzEventBus();

  bus.on('login', (data) => print('Zalogowano: $data'));
  bus.on('login', (data) => print('ZALOGOWANO: ${data.toString().toUpperCase()}'));
  bus.on('msg', (data) => print('Wiadomość: $data'));

  bus.emit('login', 'Anna');
  bus.emit('msg', 'hi');
  bus.off('msg');
  bus.emit('msg', 'ho'); // brak wyjścia — handlery usunięte
}
```

</details>

---

**Poprzedni moduł:** [Lambdy i typedef](02-lambdas-typedef.md)
**Następny moduł:** [Generatory](04-generators.md)
