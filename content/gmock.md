# Biblioteka googlemock

Podczas pisania testow czesto nie mozna, z roznych powodow, uzywac rzeczywistych obiektow.
Warto wtedy uzyc obiektu mock (mock object). Jest to obiekt ktory implementuje ten sam interfejs, co obiekt rzeczywisty, ale pozwala na
sprawdzenie kolejnosci, parametrow i ilosci wywolania metod interfejsu.
Rozni sie wiec od obiektow typu *fake*, ktore sa dzialajaca implementacja (uproszczona) rzeczywistego obiektu.

Biblioteka Google C++ Mocking Framework sluzy do tworzenia klas dostarczajacych mockow.

## Korzystanie z mockow

Praca z biblioteka *Google Mock* sprowadza sie do trzech krokow:

1. Uzycia prostych makr, aby opisac interfejs ktory chcemy pozorowac.

   * dla interfejsu `Foo`

   ```cpp
   class Foo
   {
       virtual ~Foo();
       virtual int get_size() const = 0;
       virtual string describe(const char* name) = 0;
       virtual string describe(int type) = 0;
       virtual bool process(Bar elem, int count) = 0;
   };
   ```

   * tworzymy klase pozorujaca `MockFoo`

   ```cpp
   class MockFoo : public Foo
   {
       MOCK_METHOD(int, get_size, (), (const, override));
       MOCK_METHOD(string, describe, (const char* name), (override));
       MOCK_METHOD(string, describe, (int type), (override));
       MOCK_METHOD(bool, process, (Bar elem, int count), (override));
   };
   ```

1. Utworzenia obiektow wraz ze specyfikacja ich oczekiwan i zachowania z uzyciem prostej skladni.

   ```cpp
   MockFoo foo;

   // optional step
   ON_CALL(foo, get_size())
       .WillByDefault(Return(1));

   EXPECT_CALL(foo, describe(5))
       .Times(3)
       .WillRepeatedly(Return("Id: 5"));
   ```

1. Stworzenia testowego kodu, ktory bedzie uzywal powyzszych obiektow. Google Mock bedzie odpowiedzialna za przestrzeganie wymagan zdefiniowanych w punkcie 2.

   ```cpp
   EXPECT_EQ("ok", MyProductionFunction(foo));
   ```

## Konfiguracja domyslnego zachowania dla mocka

Aby ustawic domyslna konfiguracje dla metody i okreslic zwracana wartosc
nalezy uzyc makra `ON_CALL()`:

```cpp
ON_CALL(mock, some_method(_)).WillByDefault(Return(42));
```

`ON_CALL()` definiuje zachowanie w przypadku wywolania okreslonej metody, ale **nie definiuje oczekiwania**, ze ta metoda zostanie wywolana.

## Konfiguracja oczekiwan dla mockow

Konfiguracja oczekiwan jest realizowana za pomoca makra `EXPECT_CALL()`.
`EXPECT_CALL()` wyraza konkretne oczekiwanie wywolania metody weryfikowane
w momencie wywolania destruktora obiektu pozorujacego.

```cpp
EXPECT_CALL(mock, method(matchers) /*?*/)
    .With(multi_argument_matchers) // ?
    .Times(cardinality)            // ?
    .InSequence(S1..., SN)         // *
    .After(expectations)           // *
    .WillOnce(action)              // *
    .WillRepeatedly(action)        // ?
    .RetiresOnSaturation();        // ?
```

* Jesli pominieta zostala sekcja `(matchers)`, zachowanie jest rownowazne ustawieniu dopasowan ogolnych (`_`) dla
  wszystkich argumentow (np. `(_, _, _, _)` dla metody z czterema argumentami).

* Jesli pominieta zostala sekcja `Times()`, to ilosc oczekiwanych wywolan metody jest ustawiona na:
  * `Times(1)` jesli nie ustawiono `WillOnce()` lub `WillRepeatedly()`
  * `Times(n)` jesli ustawiono `n` razy `WillOnce()`, ale nie `WillRepeatedly()`
  * `Times(AtLeast(n))` jesli ustawiono `n` razy `WillOnce()` a nastepnie ustawiono `WillRepeatedly()`

### ON_CALL vs. EXPECT_CALL

Dobre testy powinny **weryfikowac kontrakt** pomiedzy klientem (testowany kod), a obiektami zaleznymi (pozorowanymi przez mocki).
Jesli test zbyt szczegolowo specyfikuje wymagania dla mockow, to efektem jest brak mozliwosci swobodnej implementacji funkcjonalnosci.
W takiej sytuacji refaktoring kodu zwykle powoduje zgloszenie bledow przez testy.

Dobrym zaleceniem jest weryfikacja tylko jednej wlasciwosci (lub zachowania) w jednym tescie.

Stosujac gMock nalezy domyslnie uzywac `ON_CALL`, a `EXPECT_CALL` tylko kiedy chcemy zweryfikowac okreslone zachowanie.

Dobrym rozwiazaniem jest:

* skonfigurowanie mocka w fiksturze za pomoca wielu wywolan `ON_CALL` - tak skonfigurowany mock moze byc latwo wspoldzielony przez wiele testow w grupie
* a nastepnie definiowanie konkretnego oczekiwania dotyczacego zachowania obiektu za pomoca `EXPECT_CALL` w konkretnym tescie `TEST_F`.

### NiceMocks vs. StrictMocks

Domyslnie (bez konfiguracji oczekiwan) obiekt mocka stworzony przy pomocy frameworka *googlemock* akceptuje
wszystkie wywolania metod, zglaszajac ostrzezenie z opisem jaka metoda zostala wywolana.

```cpp
class Logger
{
public:
    virtual void log(const std::string& message) = 0;
    virtual ~Logger() = default;
};

struct MockLogger : Logger
{
    MOCK_METHOD(void, log, (const std::string&), (override));
};

void run(Logger& logger)
{
    logger.log("Started");
}

TEST(NiceVsStrictMocks, DefaultMockDisplaysWarningInOutput)
{
    MockLogger mock_logger; // default mock - no expectations

    run(mock_logger); // gives warning
}
```

Aby uniknac ostrzezen mozemy zastosowac `ON_CALL` lub opakowac mocka szablonem `NiceMock`:

```cpp
TEST(NiceVsStrictMocks, NiceMockWorksInSilentMode)
{
    NiceMock<MockLogger> mock_logger;

    run(mock_logger); // no warnings
}
```

W sytuacji, gdy przypadkowe wywolania moga byc problemem, mozemy zastosowac
wrapper `StrictMock`. Wtedy kazde wywolanie bez wczesniejszej konfiguracji oczekiwan
jest zglaszane jako blad:

```cpp
TEST(NiceVsStrictMocks, StrictMockReportsUnexpectedCallsAsErrors)
{
    StrictMock<MockLogger> mock_logger;

    run(mock_logger); // reported as error - unexpected call
}

TEST(NiceVsStrictMocks, StrictMockRequiresConfigurationOfExpectation)
{
    StrictMock<MockLogger> mock_logger;

    EXPECT_CALL(mock_logger, log(_)).Times(1);

    run(mock_logger); // configured expectation is verified
}
```

## Konfiguracja mockow

### Zwracane wartosci domyslne

Domyslnie metody wywolywane na mocku zwracaja domyslne wartosci okreslonego typu (**default-constructed values**).

Mozemy zmienic domyslna wartosc dla okreslonego typu, wykorzystujac szablon `DefaultValue<T>`:

```cpp
using ::testing::DefaultValue;

// sets the default value to be returned. T must be CopyConstructible.
DefaultValue<T>::Set(value);

// sets a factory. Will be invoked on demand. T must be MoveConstructible.
// T make_T();
DefaultValue<T>::SetFactory(&make_T);

// ... use the mocks ...

// resets the default value.
DefaultValue<T>::Clear();
```

### Akcje

Akcje okreslaja co powinno sie stac, kiedy okreslona metoda mocka zostanie wywolana.

#### Zwracanie wartosci

| Akcja                       | Opis                                                                                                                                |
|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| `Return()`                  | Zwraca `void`                                                                                                                       |
| `Return(value)`             | Zwraca `value`. Jesli typ `value` jest inny od typu zwracanego z funkcji, `value` jest konwertowane w czasie ustawiania oczekiwania |
| `ReturnArg<N>()`            | Zwraca N-ty argument (indeksacja od 0)                                                                                              |
| `ReturnNew<T>(a1, ..., ak)` | Zwraca `new T(a1, ..., ak)`; za kazdym wywolaniem tworzony jest nowy obiekt                                                         |
| `ReturnNull()`              | Zwraca `nullptr`                                                                                                                    |
| `ReturnPointee(ptr)`        | Zwraca wartosc wskazywana przez wskaznik `ptr`                                                                                      |
| `ReturnRef(variable)`       | Zwraca referencje do zmiennej `variable`                                                                                            |
| `ReturnRefOfCopy(value)`    | Zwraca referencje do kopii `value`                                                                                                  |

Przyklad:

```cpp
EXPECT_CALL(fake, my_method()).WillOnce(Return(-1));
ASSERT_EQ(fake.my_method(), -1);

int local = 42;
EXPECT_CALL(fake, my_method_returning_ref()).WillOnce(ReturnRef(local));

EXPECT_CALL(fake, my_method_returning_ptr()).WillOnce(ReturnPointee(&local));
ASSERT_EQ(*(fake.my_method_returning_ptr()), 42);
```

#### Definiowanie roznych zachowan w zaleznosci od parametrow

```cpp
EXPECT_CALL(mock, my_method(100)).WillOnce(Return(true));
EXPECT_CALL(mock, my_method(200)).WillOnce(Return(false));

EXPECT_CALL(mock, my_method(_)).WillRepeatedly(Return(false));
EXPECT_CALL(mock, my_method(100)).WillRepeatedly(Return(true));
```

#### Konfiguracja efektow ubocznych

Czasami metoda wywolywana na obiekcie pozorujacym daje efekt uboczny (np. ustawienie wartosci dla zmiennej przekazanej jako parametr wywolania funkcji
lub wywolanie funkcji):

```cpp
struct Mock
{
    MOCK_METHOD(bool, some_method, (bool, int*));
};

EXPECT_CALL(mock, some_method(true, _)).WillOnce(SetArgPointee<1>(10));

int x;

bool result = mock.some_method(true, &x);
ASSERT_EQ(result, false); // default value is returned from mocked method
ASSERT_EQ(x, 10);
```

Jesli chcemy polaczyc wiele efektow ubocznych, mozemy zastosowac `DoAll()`:

```cpp
EXPECT_CALL(mock, some_method(true, _))
    .WillOnce(DoAll(SetArgPointee<1>(10), Return(true)));
```

Do najczesciej wykorzystywanych efektow ubocznych naleza:

| Akcja                        | Opis                                                                    |
|------------------------------|-------------------------------------------------------------------------|
| `Assign(&variable, value)`   | Przypisuje wartosc do zmiennej `variable`                               |
| `SaveArg<N>(pointer)`        | Zapisuje N-ty (0-based) argument do `*pointer`                          |
| `SaveArgPointee<N>(pointer)` | Zapisuje wartosc wskazywana przez N-ty argument do `*pointer`           |
| `SetArgReferee<N>(value)`    | Przypisuje wartosc `value` do referencji przekazanej jako N-ty argument |
| `SetArgPointee<N>(value)`    | Przypisuje wartosc `value` do zmiennej wskazywanej przez N-ty argument  |
| `Throw(exception)`           | Rzuca wyjatek (dowolna kopiowalna wartosc)                              |

#### Wywolania funkcji, funktorow lub lambd

W ponizszej tabeli `f` oznacza funkcje, `std::function`, funktor lub lambde.

| Akcja                                               | Opis                                                                                   |
|-----------------------------------------------------|----------------------------------------------------------------------------------------|
| `f`                                                 | Wywoluje `f` z argumentami przekazanymi do mockowanej funkcji                          |
| `Invoke(f)`                                         | Wywoluje `f` z argumentami przekazanymi do mockowanej funkcji                          |
| `Invoke(object_pointer, &class::method)`            | Wywoluje metode na wskazanym obiekcie z argumentami przekazanymi do mockowanej funkcji |
| `InvokeWithoutArgs(f)`                              | Wywoluje `f`; `f` nie przyjmuje zadnych argumentow                                     |
| `InvokeWithoutArgs(object_pointer, &class::method)` | Wywoluje bezparametrowa metode na wskazanym obiekcie                                   |

Wywolanie funkcji jako efekt uboczny mozemy skonfigurowac nastepujaco:

```cpp
EXPECT_CALL(mock, some_method(_, _))
    .WillOnce(InvokeWithoutArgs(other_function));

EXPECT_CALL(mock, some_method(_, _))
    .WillOnce(InvokeWithoutArgs(IgnoreResult(another_function)));

EXPECT_CALL(mock, some_method_with_many_args(_, _, _, _))
    .WillOnce(WithArgs<0, 2, 3>(callback_function));
```

Jesli metoda, ktora mockujemy przyjmuje argument w postaci wskaznika do funkcji, mozemy uzyc tego wskaznika do wywolania funkcji z okreslonym argumentem:

```cpp
class Mock
{
    MOCK_METHOD(void, function_with_callback, (bool, void(*)(int)));
};

Mock mock;

void my_callback(int value)
{
    // ...
}

EXPECT_CALL(mock, function_with_callback(true, _)).WillOnce(InvokeArgument<1>(42));
```

#### Konfiguracja rzucania wyjatkow

Aby pozorowac rzucanie wyjatkow, nalezy skonfigurowac mocka przy pomocy opcji `Throw()`:

```cpp
InvalidArgumentException my_exception("Error #13");

EXPECT_CALL(mock, some_method(13)).WillOnce(Throw(my_exception));
```

#### Okreslanie ilosci wywolan

Opcja `Times()` umozliwia konfiguracje ilosci wywolan metody dla mocka:

```cpp
class Mock
{
    MOCK_METHOD(int, my_function, ());
};

Mock mock;

// setting expectation that my_function is called exactly 3 times returning default value
EXPECT_CALL(mock, my_function()).Times(3);
EXPECT_CALL(mock, my_function()).Times(Exactly(3));

// setting expectation that my_function is called exactly 3 times returning 10, 0, 0
EXPECT_CALL(mock, my_function()).Times(3).WillOnce(Return(10));

ASSERT_EQ(mock.my_function(), 10);
ASSERT_EQ(mock.my_function(), 0);
ASSERT_EQ(mock.my_function(), 0);

// other examples of Times options
EXPECT_CALL(mock, my_function()).Times(AtLeast(1));
EXPECT_CALL(mock, my_function()).Times(AtMost(3));
EXPECT_CALL(mock, my_function()).Times(Between(1, 5));
EXPECT_CALL(mock, my_function()).Times(AnyNumber());
```

Oczekiwania sa zapisywane na stosie (LIFO), co w rezultacie umozliwia
przedefiniowanie skonfigurowanych wczesniej (np. w fiksturze) oczekiwan.
Zawsze sprawdzenie czy wywolanie metody pasuje do ustawionej konfiguracji
zaczyna sie od ostatniego wywolania `EXPECT_CALL()`:

```cpp
TEST(OverridingExpectations, WhenLastConfigurationFitsRestIsInvisible)
{
    Mock mock;

    EXPECT_CALL(mock, is_saturated())
        .WillOnce(Return(true));

    EXPECT_CALL(mock, is_saturated())
        .Times(1)
        .WillOnce(Return(false));

    ASSERT_FALSE(mock.is_saturated());
    ASSERT_TRUE(mock.is_saturated()); // error - is_saturated() invoked twice
}
```

Aby ograniczyc czas dzialania okreslonej konfiguracji oczekiwan, mozemy uzyc
opcji `RetiresOnSaturation()`.

```cpp
TEST(OverridingExpectations, CanBeManagedWithRetiresOnSaturation)
{
    Mock mock;

    EXPECT_CALL(mock, is_saturated())
        .WillOnce(Return(true));

    EXPECT_CALL(mock, is_saturated())
        .Times(1)
        .WillOnce(Return(false))
        .RetiresOnSaturation();

    ASSERT_FALSE(mock.is_saturated());
    ASSERT_TRUE(mock.is_saturated()); // ok
}
```

#### Konfiguracja kolejnosci wywolan metod

Gdy chcemy okreslic kolejnosc wykonywanych na mocku operacji,
mozemy zastosowac opcje `After()`:

```cpp
Expectation setup = EXPECT_CALL(mock, setup());
Expectation validate = EXPECT_CALL(mock, validate());

EXPECT_CALL(mock, run()).After(setup, validate);
```

Obiekt `ExpectationSet` umozliwia agregacje oczekiwan w odpowiedniej kolejnosci:

```cpp
ExpectationSet all_inits;

for (int i = 0; i < devs_no; ++i)
{
    all_inits += EXPECT_CALL(mock, init_dev(i));
}

EXPECT_CALL(mock, run()).After(all_inits);
```

Inna opcja okreslenia kolejnosci wywolan jest zastosowanie obiektu `Sequence`:

```cpp
TEST(SequencedCalls, AllCallAreInSequence)
{
    Sequence s1, s2;

    EXPECT_CALL(mock, my_method(1)).InSequence(s1, s2);
    EXPECT_CALL(mock, my_method(2)).InSequence(s1);
    EXPECT_CALL(mock, other_method(_)).InSequence(s2);
}
```

## Matchers

Obiekty dopasowujace (*matchers*) sa wykorzystywane do sprawdzenia, czy
metoda zostala wywolana z okreslonymi parametrami.

### Dopasowanie dowolnej wartosci

Dowolna wartosc jest akceptowana przy pomocy `_` lub szablonu `A<T>` lub `An<T>`:

```cpp
EXPECT_CALL(mock, some_method(_));
EXPECT_CALL(mock, some_method(A<int>())); // usable for overloaded functions
EXPECT_CALL(mock, some_method(An<int>()));
```

### Porownania

Dopasowanie wykorzystujace operatory porownan moze byc zdefiniowane przy pomocy matcherow `Eq()`, `Ne()`, `Lt()`, `Gt()`:

```cpp
EXPECT_CALL(mock, some_method(Eq(100)));
EXPECT_CALL(mock, some_method(Ne(100)));
EXPECT_CALL(mock, some_method(Lt(100)));
EXPECT_CALL(mock, some_method(Gt(100)));
```

Dla wskaznikow mozemy wykorzystac obiekty `IsNull()` i `NotNull()`:

```cpp
EXPECT_CALL(mock, print(IsNull()));
EXPECT_CALL(mock, print(NotNull()));
```

Porownania moga byc rowniez ograniczane dla typow:

```cpp
EXPECT_CALL(mock, some_method(TypedEq<int>(100)));
EXPECT_CALL(mock, some_method(Matcher<int>(Gt(50))));
```

### Dopasowanie lancuchow znakow

Lancuchy znakow (C-strings i std::string) moga byc dopasowywane przy pomocy nastepujacych obiektow:

* `ContainsRegex(string)`
* `EndsWith(suffix)`
* `HasSubstr(string)`
* `MatchesRegex(string)`
* `StartsWith(prefix)`
* `StrCaseEq(string)`
* `StrCaseNe(string)`
* `StrEq(string)`
* `StrNe(string)`

### Laczenie wielu porownan

Laczenie wielu obiektow porownujacych moze sie odbywac przy pomocy obiektow:

* `AllOf(m1, m2, ...)`
* `AnyOf(m1, m2, ...)`
* `Not(m)`

```cpp
EXPECT_CALL(mock, some_method(AllOf(NotNull(), Not(StrEq("")), Gt(5))));
```

### Dopasowanie dla pol i getterow obiektow

* `Field(&class::field, m)`
* `Property(&class::property, m)`
* `Key(v/m)` - `EXPECT_CALL(my_map, Contains(Key(42)))`
* `Pair(m1, m2)`

```cpp
struct Gadget
{
    int id;
};

struct MockUser
{
    MOCK_METHOD(void, use, (Gadget&));
};

MockUser user;

EXPECT_CALL(user, use(Field(&Gadget::id, Gt(0)))).Times(2);

Gadget g1{10};
user.use(g1); // ok

Gadget g2{-1};
user.use(g2); // report error
```

### Dopasowania dla kontenerow

Dopasowania dla calych kontenerow:

* `ContainerEq(other)`
* `IsEmpty()`
* `Size(m)`
* `Contains(e)`
* `Each(e)`

Dopasowania dla kolekcji elementow:

* `ElementsAre(e0, e1, ...)`
* `ElementsAreArray({...})`
* `Pointwise(m, container)`
* `UnorderedElementsAre(...)`
* `WhenSorted(m)`
* `WhenSortedBy(comparator, m)`

Przyklady:

```cpp
MOCK_METHOD1(save_data, void(const vector<int>& numbers));

EXPECT_CALL(mock, save_data(UnorderedElementsAre(1, Gt(0), _, 5)));

vector<int> data = {1, 10, -100, 5};
mock.save_data(data); // ok
```

### Dopasowania wieloargumentowe

Czasami wymagane jest zdefiniowanie wzajemnie zaleznych wymagan dla argumentow wywolania mockowanej funkcji:

```cpp
EXPECT_CALL(mock, some_method(_, _)).With(Eq());
EXPECT_CALL(mock, some_method(_, _, _)).With(AllArgs(Eq())); // the same as above
EXPECT_CALL(mock, some_method(_, _, _, _)).With(Args<0, 3>(Eq()));
```

### Tworzenie wlasnych obiektow dopasowujacych

Tworzenie wlasnych obiektow dopasowujacych jest mozliwe na trzy sposoby:

1. Uzywajac funkcji `Truly(predicate)`

   ```cpp
   int is_even(int n) { return (n % 2) == 0 ? 1 : 0; }

   // some_method() must be called with an even number.
   EXPECT_CALL(mock, some_method(Truly(is_even)));
   ```

1. Piszac makro `MATCHER()`

   ```cpp
   MATCHER(IsEven, std::string(negation ? "isn't" : "is") + " even")
   {
       return arg % 2 == 0;
   }

   MATCHER_P(IsDivisible, value, std::string(negation ? "isn't" : "is") + " divisible by " + std::to_string(value))
   {
       return arg % value == 0;
   }

   MATCHER_P2(InCloseRange, low, high, std::to_string(arg)
       + std::string(negation ? " isn't" : " is")
       + " in range [" + std::to_string(low) + ", "
       + std::to_string(high) + "]")
   {
       return low <= arg && arg <= high;
   }
   ```

1. Piszac wlasna klase dziedziczaca po `MatcherInterface`

### Wykorzystanie obiektow dopasowujacych w asercjach testow

Obiekty porownujace moga byc wykorzystane w asercjach testow jednostkowych:

```cpp
ASSERT_THAT(result, AllOf(NotNull(), StrNe("")));

EXPECT_THAT(result, AnyOf(Gt(100), Le(-100)));
```

## Mockowanie metod niewirtualnych

Biblioteka gMock umozliwia mockowanie metod, ktore nie sa wirtualne. Mozemy to wykorzystac w testach kodu, ktory wykorzystuje *Hi-performance Dependency Injection*.

W takim przypadku klasa mocka nie dziedziczy po interfejsie (lub klasie z metodami wirtualnymi).

```cpp
// A simple logger class. None of its members is virtual.
class Logger
{
public:
    void log(const std::string& message);
    void warn(const std::string& warning);
    void error(const std::string& error_msg, std::error_code err_code);
};
```

Klasa mocka dla klasy logger:

```cpp
class MockLogger
{
public:
    MOCK_METHOD(void, log, (const std::string& message));
    MOCK_METHOD(void, warn, (const std::string& warning));
    MOCK_METHOD(void, error, (const std::string& error_msg, std::error_code err_code));
};
```

Aby miec mozliwosc podstawienia mocka dla potrzeb testow, musimy wykorzystac *static polymorphism* i szablony.

```cpp
template <class LoggerImpl>
Connection create_connection(const std::string& connection_str, LoggerImpl* logger)
{
    // ...

    if (logger)
        logger->log("Established connection to " + connection_str);

    // ...
}

template <class LoggerImpl>
class DataReader
{
    LoggerImpl* logger_;

public:
    explicit DataReader(LoggerImpl* logger) : logger_{logger}
    {
    }

    void read_data(const std::string& cmd)
    {
        // ...
        if (logger_)
            logger_->log("Command "s + cmd + " has been executed");
    }
};
```

Powyzszy kod daje mozliwosc zastosowania konkretnej implementacji loggera w kodzie produkcyjnym (np. `FileLogger`):

```cpp
FileLogger logger{"log.dat"};

auto conn = create_connection("http://localhost:8000", &logger);

DataReader<FileLogger> reader{&logger};
reader.read_data("select * from table");
```

W tescie mozemy natomiast wykorzystac klase mocka:

```cpp
MockLogger mock_logger;
EXPECT_CALL(mock_logger, /*...*/);
// set more expectations on mock_logger...

DataReader<MockLogger> reader(&mock_logger);
// exercise reader...
```
