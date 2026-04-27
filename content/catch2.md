# Catch2

## Proste testy

Podstwowym makrem definiującym noty test w Catch2 jest `TEST_CASE`. 

Pierwszym argumentem makra `TEST_CASE` jest nazwa testu jest dowolnym stringiem, który opisuje test. Drugi (opcjonalny) argument to tag, który pozwala na grupowanie testów.

 Do sprawdzenia warunków używamy makr `REQUIRE` i `CHECK`. Różnica między nimi polega na tym, że `REQUIRE` kończy test, jeśli warunek nie jest spełniony, a `CHECK` kontynuuje test.

```cpp
unsigned int Factorial(unsigned int number) {
    return number <= 1 ? number : Factorial(number-1)*number;
}

TEST_CASE("Factorials are computed", "[factorial, math]") 
{
    REQUIRE( Factorial(0) == 1 );
    REQUIRE( Factorial(1) == 1 );
    REQUIRE( Factorial(2) == 2 );
    REQUIRE( Factorial(3) == 6 );
    REQUIRE( Factorial(10) == 3628800 );
}
```

## Testy i sekcje

Testy mogą być zagnieżdżone w sekcje. Sekcje pozwalają na grupowanie testów i wykonywanie ich w określonej kolejności. Odpowiada to mechanizmowi fikstur w innych frameworkach testowych.

```cpp
TEST_CASE("vectors can be sized and resized", "[vector]") 
{
    std::vector<int> v(5); // This setup will be done 4 times in total, once for each section

    REQUIRE(v.size() == 5);
    REQUIRE(v.capacity() >= 5);

    SECTION("resizing bigger changes size and capacity") 
    {
        v.resize(10);

        REQUIRE(v.size() == 10);
        REQUIRE(v.capacity() >= 10);
    }

    SECTION("resizing smaller changes size but not capacity") 
    {
        v.resize(0);

        REQUIRE(v.size() == 0);
        REQUIRE(v.capacity() >= 5);
    }

    SECTION("reserving bigger changes capacity but not size") 
    {
        v.reserve(10);

        REQUIRE(v.size() == 5);
        REQUIRE(v.capacity() >= 10);
    }

    SECTION("reserving smaller does not change size or capacity") 
    {
        v.reserve(0);

        REQUIRE(v.size() == 5);
        REQUIRE(v.capacity() >= 5);
    }
}
```

Dla każdej sekcji `SECTION` test jest wykonywany od początku. Oznacza to, że każda sekcja jest uruchamiana z nowo skonstruowanym wektorem `v`.

Sekcje mogą być zagnieżdżane, co pozwala na tworzenie bardziej złożonych struktur testów.

```cpp
    SECTION( "reserving bigger changes capacity but not size" ) 
    {
        v.reserve(10);

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 10 );

        SECTION( "reserving down unused capacity does not change capacity" ) 
        {
            v.reserve( 7 );
        
            REQUIRE( v.size() == 5 );
            REQUIRE( v.capacity() >= 10 );
        }
    }
```

## Testy w stylu BDD

Catch2 pozwala na pisanie testów w stylu BDD (Behavior-Driven Development). W tym stylu testy są pisane w sposób bardziej czytelny dla osób niebędących programistami. Poprzez odpowiednie aliasy dla makr `TEST_CASE` i `SECTION` można tworzyć testy wykorzystujące styl nazewnictwa *Given-When-Then*.

```cpp
#include <catch2/catch_test_macros.hpp>

SCENARIO( "vectors can be sized and resized", "[vector]" ) {

    GIVEN( "A vector with some items" ) {
        std::vector<int> v( 5 );

        REQUIRE( v.size() == 5 );
        REQUIRE( v.capacity() >= 5 );

        WHEN( "the size is increased" ) {
            v.resize( 10 );

            THEN( "the size and capacity change" ) {
                REQUIRE( v.size() == 10 );
                REQUIRE( v.capacity() >= 10 );
            }
        }

        WHEN( "the size is reduced" ) {
            v.resize( 0 );

            THEN( "the size changes but not capacity" ) {
                REQUIRE( v.size() == 0 );
                REQUIRE( v.capacity() >= 5 );
            }
        }
        
        WHEN( "more capacity is reserved" ) {
            v.reserve( 10 );

            THEN( "the capacity changes but not the size" ) {
                REQUIRE( v.size() == 5 );
                REQUIRE( v.capacity() >= 10 );
            }
        }
        
        WHEN( "less capacity is reserved" ) {
            v.reserve( 0 );

            THEN( "neither size nor capacity are changed" ) {
                REQUIRE( v.size() == 5 );
                REQUIRE( v.capacity() >= 5 );
            }
        }
    }
}
```

## Asercje

Catch2 dostarcza kilka makr do sprawdzania warunków.

### REQUIRE i CHECK

Makra `REQUIRE` testują wyrażenie i przerywają test jeśli wyrażenie nie jest ewaluowane do `true`. Rodzina makr `CHECK` testuje wyrażenie i kontynuuje test, nawet jeśli wyrażenie nie jest prawdziwe.

```cpp
CHECK( str == "string value" );
CHECK( thisReturnsTrue() );
REQUIRE( i == 42 );
```

Wyrażenia logiczne zawierające operatory `&&` oraz `||` nie mogą być zdekomponowane przez Catch2 i ich kompilacja zwróci błąd. Aby użyć takich wyrażeń, należy:

1. Zastosować nawiasy, aby wyrażenie było ewaluowane jako pojedyncza wartość logiczna przed dekompozycją.
   
   ```cpp
   REQUIRE( (a == 1 && b == 2) );
   ```

2. Przepisać wyrażenie `REQUIRE(a == 1 && b == 2)` do postaci:
   
   ```cpp
   REQUIRE( a == 1 );
   REQUIRE( b == 2 );
   ```

### REQUIRE_FALSE

Wyrażenia poprzedzone operatorem `!` nie mogą być zdekomponowane przez Catch2. W takim przypadku można użyć makra `REQUIRE_FALSE`.

```cpp
Status result = someFunction();
REQUIRE_FALSE(result); // result must evaluate to false, and Catch2 will print
                    // out the value of ret if possibly
```

### Asercje wyjątków

Catch2 pozwala na testowanie wyjątków. 

* `REQUIRE_THROWS` sprawdza, czy wyjątek został rzucony

* makro `REQUIRE_THROWS_AS` sprawdza, czy wyjątek został rzucony i jest odpowiedniego typu

* `REQUIRE_THROWS_WITH` sprawdza, czy wyjątek został rzucony i ma odpowiedni komunikat

* `REQUIRE_NOTHROW` sprawdza, czy wyjątek nie został rzucony

```cpp
std::vector<int> v = {1, 2, 3};

REQUIRE_THROWS( v.at(3) );
REQUIRE_THROWS_AS( v.at(3), std::out_of_range );
REQUIRE_THROWS_WITH( v.at(3), "out of range" );
REQUIRE_NOTHROW( v.at(2) );
```

### Asercje liczb zmiennoprzecinkowych

Catch2 dostarcza makra do porównywania liczb zmiennoprzecinkowych. Zalecanym sposobem porównania liczb zmiennoprzecinkowych jest wykorzystanie makr dopasowujących tzw. *matchers*.

```cpp
#include <catch2/matchers/catch_matchers_floating_point.hpp>
```

Biblioteka dostarcza trzy makra:

* `WithinAbs(double target, double margin)` - sprawdza, czy wartość mieści się w określonym przez `margin` przedziale

  ```cpp
  REQUIRE_THAT(1.0, WithinAbs(1.2, 0.2));
  ```

  
* `WithinRel(FloatingPoint target, FloatingPoint eps)` - akceptuje porównanie, jeżeli wartość jest w przybliżeniu równa wartości oczekiwanej z tolerancją `eps`. Sprawdzany jest warunek `|arg - target| <= eps * max(|arg|, |target|)`. Jeżeli nie podajemy `eps`, to domyślnie jest to `std::numeric_limits<FloatingPoint>::epsilon * 100`

  ```cpp
  REQUIRE_THAT(1.0, WithinRel(1.0, 0.0001));
  ```

* `WithinULP(FloatingPoint target, uint64_t maxUlpDiff)` - tworzy porównanie, które akceptuje wartość, jeżeli różnica między wartością oczekiwaną a wartością testowaną jest mniejsza niż `maxUlpDiff` jednostek [ULP](https://en.wikipedia.org/wiki/Unit_in_the_last_place). 

  ```cpp
  REQUIRE_THAT( -0.f, WithinULP( 0.f, 0 ) );
  ```

## Testy parametryzowane

### Generatory

Testy parametryzowane pozwalają na przetestowanie kodu z różnymi zestawami danych wejściowych. Catch2 wykorzystuje generatory, które pozwalają przekazać zestaw danych do testu - respektowane są zagnieżdżenia makr `TEST_CASE` i `SECTION`.

```cpp
TEST_CASE("is_odd") 
{
    auto n = GENERATE(1, 3, 5);
    REQUIRE(is_odd(n));
}
```

Jeśli chcemy przekazać do testu parametryzowanego kilka wartości jednocześnie należy użyć makra `GENERATE` w połączeniu z generatorem `table`.  
Aby utworzyć sekcję z dynamiczną nazwą odwołującą się do parametrów, należy użyć makra `DYNAMIC_SECTION`.

```cpp
TEST_CASE("table generators", "[generators]")
{
    auto [text, length] = GENERATE(table<std::string, size_t>({
        {"a", 1},
        {"bb", 2},
        {"ccc", 3}
    }));

    DYNAMIC_SECTION("length for " << text << " is " << length)
    {
        REQUIRE(text.length() == length);
    }
}
```

W przypadku testów w stylu BDD parametryzacja może wyglądać następująco:

```cpp
SCENARIO("Eating cucumbers", "[approvals]")
{
    auto [start, eat, left] = GENERATE(table<int, int, int>({
        {12, 5, 7},
        {20, 2, 18},
        {3, 1, 2}
    }));

    auto eat_cucumber = [](int start, int eat) { return start - eat; };

    GIVEN("there are " << start << " cucumbers")
    WHEN("I eat " << eat << " cucumbers")
    THEN("I should have " << left << " cucumbers")
    {
        REQUIRE(eat_cucumber(start, eat) == left);
    }
}
```

### Parametryzacja testów typami

Catch2 pozwala na parametryzację testów typami. W tym celu należy użyć makra `TEMPLATE_TEST_CASE`.

```cpp
TEMPLATE_TEST_CASE("vector can be resized", "[template]", int, std::string, float)
{
    std::vector<TestType> vec; // This setup will be done 3 times in total, once for each type
    
    CHECK(vec.size() == 0);
    vec.resize(10);
    REQUIRE(vec.size() == 10);
}
```

## Benchmarki

Catch2 pozwala na tworzenie benchmarków. Aby utworzyć benchmark, należy użyć makra `BENCHMARK`.

* Najprostszy benchmark wykorzystuje makro `BENCHMARK`:

```cpp
TEST_CASE("Fibonacci") 
{
    CHECK(Fibonacci(0) == 1);
    // some more asserts..

    CHECK(Fibonacci(5) == 8);
    // some more asserts..

    // now let's benchmark:
    BENCHMARK("Fibonacci 20") {
        return Fibonacci(20);
    }; 

    BENCHMARK("Fibonacci 25") {
        return Fibonacci(25);
    };

    BENCHMARK("Fibonacci 30") {
        return Fibonacci(30);
    };

    BENCHMARK("Fibonacci 35") {
        return Fibonacci(35);
    };
}
```

* Zaawansowane benchmarki wykorzystują makro `BENCHMARK_ADVANCED`:

```cpp
std::vector<int> generate_data()
{
    size_t size = 1000;

    std::random_device rd{};
    std::mt19937 rnd_engine{rd()};
    std::uniform_int_distribution<> rnd_gen(0, 10'000);

    std::vector<int> vec(size);
    std::generate_n(begin(vec), size, [&] { return rnd_gen(rnd_engine); });

    return vec;
}

TEST_CASE("sorting", "[benchmarks]")
{    
    BENCHMARK_ADVANCED("sort - stl")(Catch::Benchmark::Chronometer meter) {
        auto data = generate_data(); // setup data for benchmark

        meter.measure([&data] { std::sort(data.begin(), data.end()); }); // this will be benchmarked
    };

    BENCHMARK_ADVANCED("sort - ranges")(Catch::Benchmark::Chronometer meter) {
        auto data = generate_data(); // setup data for benchmark
        
        meter.measure([&data] { std::ranges::sort(data); }); // this will be benchmarked
    };    
}
```