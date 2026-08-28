# Samouczek języka Dart

Kompleksowy samouczek języka Dart oparty na **Dart SDK 3.13**, obejmujący pełną
specyfikację językową, programowanie asynchroniczne, Isolates, Streams oraz
najważniejsze elementy biblioteki standardowej. Każdy moduł zawiera wyjaśnienia
teoretyczne, przykłady kodu oraz ćwiczenia praktyczne.

## Spis Treści

### 1. Podstawy
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 1.1 | [Zmienne i typy danych](01-basics/01-variables-types.md) | beginner | Rozróżnianie deklaracji `var`, `final`, `const` i `late` oraz wbudowanych typów danych i mechanizmu wnioskowania typów. |
| 1.2 | [Operatory](01-basics/02-operators.md) | beginner | Przegląd wszystkich kategorii operatorów w Dart, ich zachowania i priorytetów. |
| 1.3 | [Null safety](01-basics/03-null-safety.md) | beginner | System sound null safety, typy nullable i operatory null-aware oraz zmienne `late`. |

### 2. Typy
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 2.1 | [Typy generyczne](02-types/01-generics.md) | intermediate | Wprowadzenie do typów generycznych zapewniających bezpieczeństwo typów i reużywalność kodu. |

### 3. Przepływ sterowania
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 3.1 | [Instrukcje warunkowe i pętle](03-control-flow/01-conditionals-loops.md) | beginner | Sterowanie przebiegiem programu za pomocą `if`/`else`, pętli `for`, `while` i `do-while`. |
| 3.2 | [Switch i pattern matching](03-control-flow/02-switch-patterns.md) | beginner | Tradycyjny `switch/case` oraz wyrażenia `switch` z dopasowywaniem wzorców w Dart 3. |
| 3.3 | [Assert — asercje debugowe](03-control-flow/03-assert.md) | beginner | Rola instrukcji `assert` w weryfikacji założeń oraz różnica między trybem debug a produkcyjnym. |

### 4. Funkcje
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 4.1 | [Deklaracje funkcji i parametry](04-functions/01-declarations-params.md) | intermediate | Deklarowanie funkcji ze składnią strzałkową oraz parametrami nazwanymi i pozycyjnymi. |
| 4.2 | [Lambdy i typedef](04-functions/02-lambdas-typedef.md) | intermediate | Tworzenie funkcji anonimowych (lambd), aliasów typów `typedef` i użycie typu `Function`. |
| 4.3 | [Domknięcia i funkcje wyższego rzędu](04-functions/03-closures-hof.md) | intermediate | Mechanizm domknięć, zasięg leksykalny oraz funkcje wyższego rzędu. |
| 4.4 | [Generatory (sync* i async*)](04-functions/04-generators.md) | intermediate | Różnica między generatorami synchronicznymi (`sync*`) a asynchronicznymi (`async*`). |

### 5. Programowanie obiektowe
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 5.1 | [Klasy i konstruktory](05-oop/01-classes-constructors.md) | intermediate | Deklarowanie klas z polami, metodami, getterami/setterami oraz różnymi typami konstruktorów. |
| 5.2 | [Dziedziczenie i interfejsy](05-oop/02-inheritance.md) | intermediate | Dziedziczenie przez `extends`, implementacja interfejsów oraz nadpisywanie metod. |
| 5.3 | [Mixiny](05-oop/03-mixins.md) | intermediate | Mixiny jako mechanizm ponownego wykorzystania kodu bez klasycznego dziedziczenia. |
| 5.4 | [Modyfikatory klas Dart 3 i rekordy](05-oop/04-dart3-modifiers.md) | intermediate | Modyfikatory klas Dart 3 (`sealed`, `base`, `interface`, `final`, `mixin`) oraz rekordy. |

### 6. Generyki
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 6.1 | [Klasy generyczne](06-generics/01-generic-classes.md) | intermediate | Typy generyczne rozwiązujące problem bezpieczeństwa typów i reużywalności bez duplikacji kodu. |
| 6.2 | [Metody generyczne i ograniczenia typów](06-generics/02-methods-constraints.md) | intermediate | Definiowanie metod i funkcji generycznych z własnymi parametrami typowymi i ograniczeniami. |

### 7. Współbieżność
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 7.1 | [Event loop, Futures i async/await](07-concurrency/01-async-await.md) | advanced | Model pętli zdarzeń, kolejność przetwarzania kolejek mikrozadań i zdarzeń oraz `async`/`await`. |
| 7.2 | [Streams](07-concurrency/02-streams.md) | advanced | `Stream` jako asynchroniczna sekwencja zdarzeń oraz różnica między strumieniami single-subscription a broadcast. |
| 7.3 | [Isolates i współbieżność](07-concurrency/03-isolates.md) | advanced | Jednowątkowy model wykonania Dart oraz koncepcja izolacji pamięci między isolate'ami. |
| 7.4 | [Obsługa błędów w kodzie asynchronicznym](07-concurrency/04-error-handling-async.md) | advanced | Obsługa błędów w kodzie asynchronicznym oraz propagacja błędów przez granice `async`. |

### 8. Biblioteka standardowa
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 8.1 | [Biblioteka dart:core](08-standard-library/01-dart-core.md) | intermediate | Manipulacja napisami, typy liczbowe, `DateTime`/`Duration`, `Uri` oraz kolekcje z `dart:core`. |
| 8.2 | [Biblioteka standardowa — dart:collection](08-standard-library/02-dart-collection.md) | intermediate | Zaawansowane struktury danych z `dart:collection` wraz z ich złożonością czasową. |
| 8.3 | [dart:async](08-standard-library/03-dart-async.md) | advanced | Zaawansowane narzędzia asynchroniczne: `Completer`, `StreamController`, `Timer` i `Zone`. |
| 8.4 | [dart:io — pliki, procesy i sieć](08-standard-library/04-dart-io.md) | advanced | Operacje na plikach, procesach i sieci z użyciem biblioteki `dart:io`. |
| 8.5 | [Biblioteka standardowa — dart:convert](08-standard-library/05-dart-convert.md) | intermediate | Kodowanie i dekodowanie danych JSON, UTF-8, Base64 oraz tworzenie własnych koderów. |
| 8.6 | [Biblioteka standardowa — dart:math i dart:typed_data](08-standard-library/06-dart-math-typed.md) | intermediate | Narzędzia matematyczne (`Random`, geometria) oraz praca z danymi binarnymi (`typed_data`). |

### 9. Zaawansowane cechy języka
| Nr | Moduł | Poziom | Opis |
|----|-------|--------|------|
| 9.1 | [Rozszerzenia — extension methods i extension types](09-advanced/01-extensions.md) | advanced | Metody rozszerzające i typy rozszerzające (Dart 3) dodające funkcje do istniejących typów. |
| 9.2 | [Rekordy i pattern matching](09-advanced/02-records-patterns.md) | advanced | Rekordy z polami pozycyjnymi i nazwanymi oraz dopasowywanie wzorców w Dart 3. |
| 9.3 | [Adnotacje metadanych, widoczność bibliotek i importy warunkowe](09-advanced/03-annotations-visibility.md) | advanced | Adnotacje metadanych, widoczność bibliotek (`show`, `hide`, `part`, `export`) oraz importy warunkowe. |
| 9.4 | [Obsługa błędów i wyjątków](09-advanced/04-error-handling.md) | intermediate | Konstrukcje `throw`, `rethrow`, `try`/`catch`/`finally` oraz hierarchie `Exception` i `Error`. |
