# GoogleTest

## Podstawowe koncepcje

W frameworkach do TDD, takich jak *Google Test* pracę zaczynamy od pisania asercji, czyli wyrażeń, które sprawdzają, czy warunek jest spełniony.

Rezultatem asercji może być:

* sukces
* błąd niekrytyczny
* błąd krytyczny

## Podstawowe asercje

| Krytyczne                 | Niekrytyczne              | Sprawdza                   |
|---------------------------|---------------------------|----------------------------|
| `ASSERT_TRUE(warunek)`    | `EXPECT_TRUE(warunek)`    | Czy warunek jest prawdziwy |
| `ASSERT_FALSE(warunek)`   | `EXPECT_FALSE(warunek)`   | Czy warunek jest fałszywy  |

`ASSERT_*` powoduje, że zostaje przerwane wykonywanie bieżącego testu, a `EXPECT_*` kontynuuje jego wykonywanie.

## Porównywanie

| Krytyczne                         | Niekrytyczne                     | Sprawdza           |
|-----------------------------------|----------------------------------|--------------------|
| `ASSERT_EQ(expexcted, actual)`    | `EXPECT_EQ(expected, actual)`    | expected == actual |
| `ASSERT_NE(val1, val2)`           | `EXPECT_NE(val1, val2)`          | val1 != val2       |
| `ASSERT_LT(val1, val2)`           | `EXPECT_LT(val1, val2)`          | val1 < val2        |
| `ASSERT_LE(val1, val2)`           | `EXPECT_LE(val1, val2)`          | val1 <= val2       |
| `ASSERT_GT(val1, val2)`           | `EXPECT_GT(val1, val2)`          | val1 > val2        |
| `ASSERT_GE(val1, val2)`           | `EXPECT_GE(val1, val2)`          | val1 >= val2       |

Argumenty przekazane do asercji muszą umożliwiać odpowiednie porównanie poprzez dostarczenie odpowiadających operatorów.

Argumenty mogą również dostarczyć `operator <<` do komunikacji przy pomocy `std::ostream`. Taki operator zostanie wykorzystany przez Google Test do prezentacji argumentu.

## Proste testy

Aby utworzyć test, należy użyć makra `TEST()` w celu zdefiniowania nazwy testu. `TEST()` jest zwykłą funkcją C++ niezwracającą żadnej wartości. Wewnątrz ciała funkcji można umieszczać dowolne wyrażenia języka.

```cpp
TEST(IsPrimeTest, When_NumberIsPrime_ReturnsTrue)
{
    ASSERT_TRUE(is_prime(2));
    ASSERT_FALSE(is_prime(10));
}

TEST(ArrayEqualityTest, EqualityOfItems)
{
    int arr1[5] = {1, 2, 3, 4, 5};
    int arr2[5] = {1, 2, 3, 2, 5};
    for (int i = 0; i < 5; ++i)
    {       
        EXPECT_EQ(arr1[i], arr2[i]) << "for index: " << i;
    }
}
```

Za pomocą `operatora <<` można przekazać do asercji dodatkowe informacje, które zostaną wypisane obok ewentualnego błędu.

Aby uruchomić test, można skorzystać z gotowej funkcji `main` znajdującej się w `gtest_main` lub napisać własną funkcję.

```cpp
int main(int argc, char** argv) 
{
    ::testing::InitGoogleTest(&argc, argv);
    return RUN_ALL_TESTS();
}
```

Rezultatem będzie raport o przeprowadzonych testach.

```bash
[==========] Running 2 tests from 2 test cases.
[----------] Global test environment set-up.
[----------] 1 test from PrimeTest
[ RUN      ] IsPrimeTest.When_NumberIsPrime_ReturnsTrue
[       OK ] IsPrimeTest.When_NumberIsPrime_ReturnsTrue (1 ms)
[----------] 1 test from IsPrimeTest (7 ms total)

[----------] 1 test from ArrayEqualityTest
[ RUN      ] ArrayEqualityTest.EqualityOfItems
..\simple.cpp:27: Failure
Value of: arr2[i]
  Actual: 2
Expected: arr1[i]
Which is: 4
for index: 3
[  FAILED  ] ArrayEqualityTest.EqualityOfItems (9 ms)
[----------] 1 test from ArrayEquality (15 ms total)

[----------] Global test environment tear-down
[==========] 2 tests from 2 test cases ran. (51 ms total)
[  PASSED  ] 1 test.
[  FAILED  ] 1 test, listed below:
[  FAILED  ] ArrayEqualityTest.EqualityOfItems

 1 FAILED TEST
```

## Fikstury

Jeśli tworzymy testy operujące na tym samym zestawie danych, można użyć klasy dziedziczącej po `::testing::Test`, która będzie umożliwiała ponowne użycie kodu:

* Wewnątrz klasy definiujemy wszystkie obiekty, których użycie jest planowane w testach
  * pola klasy powinny być zadeklarowane jako `protected`
* Można również metod pomocniczych
* Przygotowanie obiektu fikstury odbywa się albo w konstruktorze, albo przy pomocy specjalnej metody `SetUp()`
* Jeśli zachodzi potrzeba zwalniana zasobów, można użyć destruktora lub metody `TearDown()`, która w przeciwieństwie do destruktora może wygenerować wyjątek

```{important}
Jeśli używamy fikstury, to należy użyć makra `TEST_F(…)` zamiast `TEST(…)`
```

```cpp
class VectorTests : public ::testing::Test 
{
protected:
    std::vector<std::string> vec;
public:
    VectorTests() 
    {
        // you can set-up work for each test             
        for (int i = 1; i <= 4; ++i) 
        {
            vec.push_back("Item#" + std::to_string(i));
        }
    }

    ~VectorTests() override
    {
        // you can clean-up work that does not THROW exceptions
    }

    void SetUp() override
    {
        // Code here will be called immediately after the constructor (right
        // before each test).
        vec.push_back("Item#5");

    }

    void TearDown() override
    {
        // Code here will be called immediately after each test (right
        // before the destructor).
    }
};

TEST_F(VectorTests, VectorEquality) 
{
    std::vector<std::string> expected = { "Item#1", "Item#2", "Item#3", "Item#4", "Item#5" };
    
    ASSERT_EQ(vec, expected);
}
```

## Dzielenie zasobów pomiędzy testami

Ponieważ konstruktor `SetUp()` fikstury jest wywoływany przed rozpoczęciem każdego następnego testu, może mieć to duży wpływ na wydajność procesu testowania. Jeśli zasób jest używany tylko do odczytu to bezpieczne jest dzielenie go pomiędzy testami.

Przykład:

```cpp
class SharedArrayTests : public ::testing::Test 
{
protected:
    static std::vector<int> vec;

    static void SetUpTestCase() 
    {
        std::cout << "Inside static fixture constructor" << std::endl;
        for (int i = 1; i <= 5; ++i) 
        {
            vec.push_back(i);
        }
    }
};

std::vector<int> SharedArrayTests::vec;

TEST_F(SharedArrayTests, ArrayTestFirst) 
{
    EXPECT_EQ(vec[0], 1);
}
```

## Testowanie wyjątków

Jeśli nasz kod posługuje się wyjątkami, to ważne jest sprawdzenie czy testowany kod poprawnie je obsługuje. Zapewniają to poniższe asercje:

| Krytyczne                       | Niekrytyczne                    | Sprawdza                                      |
|---------------------------------|---------------------------------|-----------------------------------------------|
| `ASSERT_THROW(expr, type)`      | `EXPECT_THROW(expr, type)`      | Wyrażenie wygenerowało wyjątek określonego typu |
| `ASSERT_ANY_THROW(expr)`        | `EXPECT_ANY_THROW(expr)`        | Wyrażenie wygenerowało wyjątek dowolnego typu |
| `ASSERT_NO_THROW(expr)`         | `EXPECT_NO_THROW(expr)`         | Wyrażenie nie wygenerowało żadnego wyjątku    |

Przykładowo:

```cpp
void simple_crash()
{
    throw std::runtime_error("ERROR");
}

TEST(ExceptionTests, SimpleCrashTrowsException) 
{
    EXPECT_THROW(simple_crash(), std::runtime_error);
}
```

## Porównywanie liczb zmiennoprzecinkowych

Porównywanie liczb zmiennoprzecinkowych bywa trudne, dlatego Google Test dostarcza specjalnych testów.

| Krytyczne                       | Niekrytyczne                    | Sprawdza                                    |
|---------------------------------|---------------------------------|---------------------------------------------|
| `ASSERT_FLOAT_EQ(a, b)`         | `EXPECT_FLOAT_EQ(a, b)`         | Czy liczby są niemal równe                  |
| `ASSERT_NEAR(a, b, margin)`     | `EXPECT_NEAR(a, b, margin)`     | Czy liczby są równe z określonym marginesem |

Niemal równe oznacza błąd mniejszy niż cztery ULP – *Units in the Last Place*, czyli ostatnich znaczących bitów.

```cpp
TEST(FloatEquality, SimpleError)
{
    float a = 1.0;
    float b = 1.0 + 1e-7;
    EXPECT_FLOAT_EQ(a, b) << a << "=" << b;
}

TEST(FloatEquality, MarginError)
{
    float a = 1;
    float b = 1.1;
    EXPECT_NEAR(a, b, 0.2);
}
```

## Death Tests

Są to testy sprawdzające poprawne zakończenie się pracy programu.

| Krytyczne                                      | Niekrytyczne                               | Sprawdza                                            |
|------------------------------------------------|--------------------------------------------|-----------------------------------------------------|
| `ASSERT_DEATH(statement, regex)`               | `EXPECT_DEATH(statement, regex)`           | statement kończy pracę programu z podanym błędem    |
| `ASSERT_EXIT(statement, predicate, regex)`     | `EXPECT_EXIT(statement, predicate, regex)` | statement kończy pracę programu z podanym błędem, a kod błędu spełnia predykat predicate |

A predykaty to:

* `::testing::ExitedWithCode(exit_code)`
* `::testing::KilledBySignal(signal_number)  // Not available on Windows`

```cpp
bool is_prime(long n) 
{
    if (n > 0)
    {
        // some implementation
    }
    else 
    {
        std::cerr << "Error: Negative or zero input\n";
        exit(-1);
    }
}

TEST(PrimeTest, PrimesForPositiveNumbers) 
{
    ASSERT_EXIT(
        is_prime(-1), ::testing::ExitedWithCode(-1),
        "Error: Negative or zero input"
    );
}
```

## Bezpośrednie wywoływanie sukcesu lub porażki testu

Można wymusić wygenerowanie „sukcesu” poprzez użycie makra `SUCCEED()`;

Nie oznacza to że cały test się powiódł. Sukces testu występuje gdy każda jego asercja zakończyła się powodzeniem.

* `FAIL()` generuje błąd krytyczny testu
* `ADD_FAILURE()` generuje błąd niekrytyczny

```cpp
switch(expression) 
{
  case 1:
      break;
  case 2:
      break;
  default: 
    FAIL() << "Nie powinno nas tu byc";
}
```

## Testy parametryzowane

Framework GoogleTest umożliwia tworzenie testów, które są parametryzowane zestawami danych.

Aby przygotować parametryzowane testy dla funkcji:

```cpp
int sum(int a, int b)
{
    return a + b;
}
```

Należy przygotować:

* strukturę przechowującą zestaw parametrów testu:

```cpp
struct SumTestParams
{
    int a, b, expected;	

    SumTestParams(int a, int b, int expected)
        : a{a}, b{b}, expected{expected}
    {
    }
};
```

* fiksturę testów parametrycznych

```cpp
struct ParametrizedTest : public testing::TestWithParam<SumTestParams>
{
};
```

Kolejnym krokiem jest przygotowanie testu za pomocą makra `TEST_P`:

```cpp
TEST_P(ParametrizedTest, AddingTwoNumbers)
{
    SumTestParams params = GetParam();
    ASSERT_EQ(sum(params.a, params.b), params.expected);
}
```

Funkcja `GetParam()` umożliwia zwrócenie obiektu agregującego parametry testów - instancji `SumTestParams`.

Ostatnim krokiem jest utworzenie tablicy parametrów i zainicjowanie nią zestawu testów:

```cpp
SumTestParams params[] = { {1, 2, 3}, {5, 6, 11}, {665, 1, 666} };

INSTANTIATE_TEST_CASE_P(PackOfTests, ParametrizedTest, testing::ValuesIn(params));
```

Wynik wykonania testów parametrycznych:

```bash
[----------] 3 tests from PackOfTests/ParametrizedTest
[ RUN      ] PackOfTests/ParametrizedTest.AddingTwoNumbers/0
[       OK ] PackOfTests/ParametrizedTest.AddingTwoNumbers/0 (0 ms)
[ RUN      ] PackOfTests/ParametrizedTest.AddingTwoNumbers/1
[       OK ] PackOfTests/ParametrizedTest.AddingTwoNumbers/1 (0 ms)
[ RUN      ] PackOfTests/ParametrizedTest.AddingTwoNumbers/2
[       OK ] PackOfTests/ParametrizedTest.AddingTwoNumbers/2 (0 ms)
[----------] 3 tests from PackOfTests/ParametrizedTest (0 ms total)
```

## Testy dla list typów

W przypadku testów pisanych dla klas generycznych możemy uniknąć duplikacji logiki testów stosując tzw. testy typizowane (**typed tests**).
W takim przypadku zamiast makr `TEST` lub `TEST_F` należy zastosować makro `TYPED_TEST`. 

Pierwszym krokiem jest zdefiniowanie fikstury jako szablonu klasy sparametryzowanej typem:

```cpp
template <typename T>
class VectorTest : public ::testing::Test
{
public:
    using VectorType = std::vector<T>;
    inline static T value_{};

    std::vector<T> vec_{ T{}, T{}, value_ };
};
```

Następnie definiujemy listę typów, którą chcemy wykorzystać w testach:

```cpp
using MyListOfTypes = ::testing::Types<char, int, uint64_t>;
TYPED_TEST_SUITE(VectorTest, MyListOfTypes);
```

Następnie stosujemy makro `TYPED_TEST()` zamiast `TEST_F()`:

```cpp
TYPED_TEST(VectorTest, push_back_increases_size)
{
    TypeParam default_value = value_; // TypeParam allows to get the parameter type of a test

    auto size_before = vec_.size();

    vec_.push_back(default_value);

    ASSERT_EQ(size_before + 1, vec_.size());
}
```

## Selekcja testów

### Wyłączanie pojedynczego testu

Aby wyłączyć pojedynczy test, wystarczy nazwę tego testu poprzedzić prefiksem `DISABLED_`.

```cpp
TEST(TestCase, DISABLED_SimpleTest)
```

### Uruchomienie wybranych testów

Domyślnie Google Test uruchamia wszystkie testy zdefiniowane przez użytkownika.

Jeśli chcemy uruchomić jedynie ich podzbiór, to możemy użyć zmiennej środowiskowej `GTEST_FILTER`, lub ustawić w przed wywołaniem `InitGoogleTest` tzw. "filter string".

```cpp
int main(int argc, char** argv) 
{
    ::testing::GTEST_FLAG(filter) = "*Simple*";
    ::testing::InitGoogleTest(&argc, argv);

    return RUN_ALL_TESTS();
}
```

Trzecią możliwością jest użycie przełączników linii poleceń.

Przykłady wzorów dopasowania:

```bash
./foo_test Has no flag, and thus runs all its tests.
./foo_test --gtest_filter=* Also runs everything, due to the single match-everything * value.
./foo_test --gtest_filter=FooTest.* Runs everything in test case FooTest.
./foo_test --gtest_filter=*Null*:*Constructor* Runs any test whose full name contains either "Null" or "Constructor".
./foo_test --gtest_filter=-*DeathTest.* Runs all non-death tests.
./foo_test --gtest_filter=FooTest.*-FooTest.Bar Runs everything in test case FooTest except FooTest.Bar
```
````