Biblioteka googlemock
=====================

Podczas pisania testów często nie można, z różnych powodów, używać rzeczywistych obiektów. 
Warto wtedy użyć obiektu mock (mock object). Jest to obiekt który implementuje ten sam interfejs, co obiekt rzeczywisty, ale pozwala na 
sprawdzenie kolejności, parametrów i ilości wywołania metod interfejsu. 
Różni się więc od obiektów typu *fake*, które są działającą implementacją (uproszczoną) rzeczywistego obiektu. 

Biblioteka Google C++ Mocking Framework służy do tworzenia klas dostarczających mocków.

Korzystanie z mocków
--------------------

Praca z biblioteką *Google Mock* sprowadza się do trzech kroków:

1. Użycia prostych makr, aby opisać interfejs który chcemy pozorować.

   * dla interfejsu ``Foo``

   .. code-block:: c++
   
        class Foo 
        {
            virtual ~Foo();
            virtual int get_size() const = 0;
            virtual string describe(const char* name) = 0;
            virtual string describe(int type) = 0;
            virtual bool process(Bar elem, int count) = 0;
        };
    
   * tworzymy klasę pozorującą ``MockFoo``

   .. code-block:: c++
    
        class MockFoo : public Foo 
        {
            MOCK_METHOD(int, get_size, (), (const, override));
            MOCK_METHOD(string, describe, (const char* name), (override));
            MOCK_METHOD(string, describe, (int type), (override));
            MOCK_METHOD(bool, process, (Bar elem, int count), (override));
        };

2. Utworzenia obiektów wraz ze specyfikacją ich oczekiwań i zachowania z użyciem prostej składni.

   .. code-block:: c++
   
        MockFoo foo;

        // optional step
        ON_CALL(foo, get_size())
            .WillByDefault(Return(1));

        EXPECT_CALL(foo, describe(5)).
            .Times(3)
            .WillRepeatedly(Return("Id: 5");
        
3. Stworzenia testowego kodu, który będzie używał powyższych obiektów. Google Mock będzie odpowiedzialna za przestrzeganie wymagań zdefiniowanych w punkcie 2.

   .. code-block:: c++
   
        EXPECT_EQ("ok", MyProductionFunction(foo));


Konfiguracja domyślnego zachowania dla mocka
--------------------------------------------

Aby ustawić domyślną konfigurację dla metody i określić zwracaną wartość
należy użyć makra ``ON_CALL()``:

.. code-block:: cpp

    ON_CALL(mock, some_method(_)).WillByDefault(Return(42));

``ON_CALL()`` definiuje zachowanie w przypadku wywołania określonej metody, ale **nie definiuje oczekiwania**, że ta metoda zostanie wywołana.

Konfiguracja oczekiwań dla mocków
---------------------------------

Konfiguracja oczekiwań jest realizowana za pomocą makra ``EXPECT_CALL()``.
``EXPECT_CALL()`` wyraża konkretne oczekiwanie wywołania metody weryfikowane
w momencie wywołania destruktora obiektu pozorującego. 

.. code-block:: cpp

    EXPECT_CALL(mock, method(matchers) /*?*/)
        .With(multi_argument_matchers) // ?
        .Times(cardinality)            // ?
        .InSequence(S1..., SN)         // *
        .After(expectations)           // *
        .WillOnce(action)              // *
        .WillRepeatedly(action)        // ?
        .RetiresOnSaturation();        // ?

* Jeśli pominięta została sekcja ``(matchers)``, zachowanie jest równoważne ustawieniu dopasowań ogólnych (``_``) dla 
  wszystkich argumentów (np. ``(_, _, _, _)`` dla metody z czterema argumentami.

* Jeśli pominięta została sekcja ``Times()``, to ilość oczekiwanych wywołań metody jest ustawiona na:
  
  * ``Times(1)`` jeśli nie ustawiono ``WillOnce()`` lub ``WillRepeatedly()``
  * ``Times(n)`` jeśli ustawiono ``n`` razy ``WillOnce()``, ale nie ``WillRepeatedly()``
  * ``Times(AtLeast(n))`` jeśli ustawiono ``n`` razy ``WillOnce()`` a następnie ustawiono ``WillRepeatedly()``


ON_CALL vs. EXPECT_CALL
***********************

Dobre testy, powinny **weryfikować kontrakt** pomiędzy klientem (testowany kod), a obiektami zależnymi (pozorowanymi przez mocki).
Jeśli test zbyt szczegółowo specyfikuje wymagania dla mocków, to efektem jest brak możliwości swobodnej implementacji funkcjonalności.
W takiej sytuacji refaktoring kodu zwykle powoduje zgłoszenie błędów przez testy.

Dobrym zaleceniem jest weryfikacja tylko jednej właściwości (lub zachowania) w jednym teście. 

Stosując gMock należy domyślnie używać ``ON_CALL`` a używać ``EXPECT_CALL`` tylko kiedy chcemy zweryfikować określone zachowanie.

Dobrym rozwiązaniem jest:

* skonfigurowanie mocka w fiksturze za pomocą wielu wywołań ``ON_CALL`` - tak skonfigurowany mock może być łatwo współdzielony przez wiele testów w grupie
* a następnie definiować konkretne oczekiwanie dotyczące zachowania obiektu za pomocą ``EXPECT_CALL`` w konkretnym teście ``TEST_F``.

NiceMocks vs. StrictMocks
*************************

Domyślnie (bez konfiguracji oczekiwań) obiekt mocka stworzony przy pomocy framework'a *googlemock* akceptuje
wszystkie wywołania metod zgłaszając ostrzeżenie z opisem jaka metoda została wywołana.

.. code-block:: cpp

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


Aby uniknąć ostrzeżeń możemy zastosować ``ON_CALL`` lub opakować mocka szablonem ``NiceMock``:

.. code-block:: cpp

    TEST(NiceVsStrictMocks, NiceMockWorksInSilentMode)
    {
        NiceMock<MockLogger> mock_logger;

        run(mock_logger); // no warnings
    }

W sytuacji, gdy przypadkowe wywołania mogą być problemem możemy zastosować
wrapper ``StricMock``. Wtedy każde wywołanie bez wcześniejszej konfiguracji oczekiwań
jest zgłaszane jako błąd:

.. code-block:: cpp

    TEST(NiceVsStrictMocks, StrictMockReportsUnexpectedCallsAsErrors)
    {
        StrictMock<MockLogger> mock_logger;

        run(mock_logger); // reported as error - unexpected call
    }

    TEST(NiceVsStrictMocks, StrictMockRequiresConfigurationOfExpectation)
    {
        StrictMock<MockLogger> mock_logger;

        EXPECT_CALL(mock_logger, log(_)).Times(1);

        run(mock_logger); // reported as error - unexpected call
    }


Konfiguracja mocków
-------------------

Zwracane wartości domyślne
**************************

Domyślnie metody wywoływane na mocku zwracają domyślne wartości określonego typu (**default-constructed values**).

Możemy zmienić domyślną wartość dla określonego typu wykorzystując szablon ``DefaultValue<T>``:

.. code-block:: c++

    using ::testing::DefaultValue;

    // sets the default value to be returned. T must be CopyConstructible.
    DefaultValue<T>::Set(value);

    // sets a factory. Will be invoked on demand. T must be MoveConstructible.
    //  T make_T();
    DefaultValue<T>::SetFactory(&make_T);
    
    // ... use the mocks ...
    
    // resets the default value.
    DefaultValue<T>::Clear();

Akcje
*****

Akcje określają co powinno się stać, kiedy określona metoda mocka zostanie wywołana:

Zwracanie wartości
~~~~~~~~~~~~~~~~~~

.. tabularcolumns:: |\Y{0.35}|\Y{0.65}|

+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``Return()``                  | Zwraca ``void``                                                                                                                         |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``Return(value)``             | Zwraca ``value``. Jeśli typ `value` jest inny od typu zwracanego z funkcji, ``value`` jest konwertowane w czasie ustawiania oczekiwania |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``ReturnArg<N>()``            | Zwraca N-ty argument (indeksacja od 0)                                                                                                  |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``ReturnNew<T>(a1, ..., ak)`` | Zwraca ``new T(a1, ..., ak)``; za każdym wywołaniem tworzony jest nowy obiekt                                                           |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``ReturnNull()``              | Zwraca ``nullptr``                                                                                                                      |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``ReturnPointee(ptr)``        | Zwraca wartość wskazywaną przez wskaźnik ``ptr``                                                                                        |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``ReturnRef(variable)``       | Zwraca referencję do zmiennej ``variable``                                                                                              |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+
| ``ReturnRefOfCopy(value)``    | Zwraca referencję do kopii ``value``                                                                                                    |
+-------------------------------+-----------------------------------------------------------------------------------------------------------------------------------------+

Przykład:

.. code-block:: cpp

    EXPECT_CALL(fake, my_method()).WillOnce(Return(-1));
    ASSERT_EQ(fake.my_method(), -1);

    int local = 42;
    EXPECT_CALL(fake, my_method_returning_ref()).WillOnce(ReturnRef(local));

    EXPECT_CALL(fake, my_method_returning_ptr()).WillOnce(ReturnPointee(&local));
    ASSERT_EQ(*(fake.my_method_returning_ptr()), 42);


Definiowanie różnych zachowań w zależności od parametrów
********************************************************

.. code-block:: cpp

    EXPECT_CALL(mock, my_method(100)).WillOnce(Return(true));
    EXPECT_CALL(mock, my_method(200)).WillOnce(Return(false));

    EXPECT_CALL(mock, my_method(_)).WillRepeatedly(Return(false));
    EXPECT_CALL(mock, my_method(100)).WillRepeatedly(Return(true));


Konfiguracja efektów ubocznych
******************************

Czasami metoda wywoływana na obiekcie pozorującym daje efekt uboczny (np. ustawienie wartości dla zmiennej przekazanej jako parametr wywołania funkcji
lub wywołanie funkcji):

.. code-block:: cpp

    struct Mock
    {
        MOCK_METHOD(bool, some_method, (bool, int*));
    };

    EXPECT_CALL(mock, some_method(true, _)).WillOnce(SetArgPointee<1>(10));

    int x;

    bool result = mock.some_method(true, &x);
    ASSERT_EQ(result, false); // default value is returned from mocked method
    ASSERT_EQ(*x, 10);

Jeśli chcemy połączyć wiele efektów ubocznych możemy zastosować ``DoAll()``:

.. code-block:: cpp

    EXPECT_CALL(mock, some_method(true, _)).WillOnce(DoAll(SetArgPointee<1>(10), Return(true));

Do najczęściej wykorzystywanych efektów ubocznych należą:

.. tabularcolumns:: |\Y{0.35}|\Y{0.65}|

+--------------------------------+---------------------------------------------------------------------------+
| ``Assign(&variable, value)``   | Przypisuje wartość do zmiennej ``variable``                               |
+--------------------------------+---------------------------------------------------------------------------+
| ``SaveArg<N>(pointer)``        | Save the N-th (0-based) argument to ``*pointer``                          |
+--------------------------------+---------------------------------------------------------------------------+
| ``SaveArgPointee<N>(pointer)`` | Zapisuje wartość wskazywanę przez N-ty argument do ``*pointer``           |
+--------------------------------+---------------------------------------------------------------------------+
| ``SetArgReferee<N>(value)``    | Przypisuje wartość ``value`` do referencji przekazanej jako N-ty argument |
+--------------------------------+---------------------------------------------------------------------------+
| ``SetArgPointee<N>(value)``    | Przypisuje wartość ``value`` do zmiennej wskazwanej przez N-ty argument   |
+--------------------------------+---------------------------------------------------------------------------+
| ``Throw(exception)``           | Rzuca wyjątek (dowolną kopiowalną wartość)                                |
+--------------------------------+---------------------------------------------------------------------------+

Wywołania funkcji, funktorów lub lambd
**************************************

W poniższej tabeli ``f`` oznacza funkcję, ``std::function``, funktor lub lambdę.

.. tabularcolumns:: |\Y{0.45}|\Y{0.55}|

+-------------------------------------------------------+----------------------------------------------------------------------------------------+
| ``f``                                                 | Wywołuje ``f`` z argumentami przekazanymi do mockowanej funkcji                        |
+-------------------------------------------------------+----------------------------------------------------------------------------------------+
| ``Invoke(f)``                                         | Wywołuje ``f`` z argumentami przekazanymi do mockowanej funkcji                        |
+-------------------------------------------------------+----------------------------------------------------------------------------------------+
| ``Invoke(object_pointer, &class::method)``            | Wywołuje metodę na wskazanym obiekcie z argumentami przekazanymi do mockowanej funkcji |
+-------------------------------------------------------+----------------------------------------------------------------------------------------+
| ``InvokeWithoutArgs(f)``                              | Wywołuje ``f``; ``f`` nie przyjmuje żadnych argumentów                                 |
+-------------------------------------------------------+----------------------------------------------------------------------------------------+
| ``InvokeWithoutArgs(object_pointer, &class::method)`` | Wywołuje bezparametrową metodę na wskazanym obiekcie                                   |
+-------------------------------------------------------+----------------------------------------------------------------------------------------+

Wywołanie funkcji jako efekt uboczny możemy skonfigurować następująco:

.. code-block:: cpp

    EXPECT_CALL(mock, some_method(_, _))
        .WillOnce(InvokeWithoutArgs(other_function));

    EXPECT_CALL(mock, some_method(_, _))
        .WillOnce(InvokeWithoutArgs(IgnoreResult(another_function));

    EXPECT_CALL(mock, some_method_with_many_args(_, _, _, _))
        .WillOnce(WithArgs<0, 2, 3>(callback_function));

Jeśli metoda, którą mockujemy przyjmuje argument w postaci wskaźnika do funkcji, możemy użyć tego wskaźnika do wywołania funkcji z określonym argumentem:

.. code-block:: cpp


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


Konfiguracja rzucania wyjątków
******************************

Aby pozorować rzucanie wyjątków należy skonfigurować mocka przy pomocy opji ``Throw()``:

.. code-block:: cpp

    InvalidArgumentException my_exception("Error #13");

    EXPECT_CALL(mock, some_method(13)).WillOnce(Throw(my_exception));

Określanie ilości wywołań
*************************

Opcja ``Times()`` umożliwia konfigurację ilości wywołań metody dla mocka:

.. code-block:: cpp

    class Mock
    {
        MOCK_METHOD(int, my_function, ());
    };

    Mock mock;

    // setting expectation that my_function is called exactly 3 times returning default value
    EXPECT_CALL(mock, my_function()).Times(3);
    EXPECT_CALL(mock, my_function()).Times(Exactly(3));
    
    // setting expectation that my function is called exactly 3 times returning 10, 0, 0
    EXPECT_CALL(mock, my_function()).Times(3).WillOnce(Return(10));
    
    ASSERT_EQ(mock.my_function(), 10);
    ASSERT_EQ(mock.my_function(), 0);
    ASSERT_EQ(mock.my_function(), 0);

    // other examples of Times options
    EXPECT_CALL(mock, my_function()).Times(AtLeast(1));
    EXPECT_CALL(mock, my_function()).Times(AtMost(3));
    EXPECT_CALL(mock, my_function()).Times(Between(1, 5));
    EXPECT_CALL(mock, my_function()).Times(AnyNumber());

Oczekiwania są zapisywane na stosie (LIFO) co w rezultacie umożliwia 
przedefiniowanie skonfigurowanych wcześniej (np. w fiksturze) oczekiwań.
Zawsze sprawdzenie czy wywołanie metody pasuje do ustawionej konfiguracji
zaczyna się od ostatniego wywołania ``EXPECT_CALL()``:


.. code-block:: cpp

    TEST(OverridingExpectations, WhenLastConfigurationFitsRestIsInvisible)
    {
        Mock mock;

        EXPECT_CALL(mock, is_saturated())
            .WillOnce(Return(true)):
    
        EXPECT_CALL(mock, is_saturated())
            .Times(1)
            .WillOnce(Return(false));

        ASSERT_FALSE(mock.is_saturated());
        ASSERT_TRUE(mock.is_saturated()); // error - is_saturated() ivoked twice
    }

Aby ograniczyć czas działania określonej konfiguracji oczekiwań możemy użyć
opcji ``RetiresOnSaturation()``

.. code-block:: cpp

    TEST(OverridingExpectations, CanBeManagedWithRetiresOnSaturation)
    {
        Mock mock;

        EXPECT_CALL(mock, is_saturated())
            .WillOnce(Return(true)):
    
        EXPECT_CALL(mock, is_saturated())
            .Times(1)
            .WillOnce(Return(false))
            .RetiresOnSaturation();

        ASSERT_FALSE(mock.is_saturated());
        ASSERT_TRUE(mock.is_saturated()); // ok
    }

Konfiguracja kolejności wywołań metod
*************************************

Gdy chcemy określić kolejność wykonywanych na mocku operacji
możemy zastosować opcję ``After()``:

.. code-block:: cpp

    Expectation setup = EXPECT_CALL(mock, setup());
    Expectation validate = EXPECT_CALL(mock, validate());

    EXPECT_CALL(mock, run()).After(setup, validate);

Obiekt ``ExpectationSet`` umożliwia agregację oczekiwań w odpowiedniej kolejności:

.. code-block:: cpp

    ExpectationSet all_inits;

    for(int i = 0; i < devs_no; ++i)
        all_inits += EXPECT_CALL(mock, init_dev(i));

    EXPECT_CALL(mock, run()).After(all_inits);

Inną opcją określenia kolejności wywołań jest zastosowanie obiektu ``Sequence``:

.. code-block:: cpp

    TEST(SequencedCalls, AllCallAreInSequence)
    {
        Sequence s1, s2;

        EXPECT_CALL(mock, my_method(1)).InSequence(s1, s2);
        EXPECT_CALL(mock, my_method(2)).InSequence(s1);
        EXPECT_CALL(mock, other_method(_)).InSequence(s2);
    }

Matchers
--------

Obiekty dopasowujące (*matchers*) są wykorzystywane do sprawdzenia, czy 
metoda została wywołana z określonymi parametrami.

Dopasowanie dowolnej wartości
*****************************

Dowolna wartość jest akceptowana przy pomocy ``_`` lub szablonu ``A<T>`` lub ``An<T>``:

.. code-block:: cpp

    EXPECT_CALL(mock, some_method(_));
    EXPECT_CALL(mock, some_method(A<int>()); // usable for overloaded functions
    EXPECT_CALL(mock, some_method(An<int>());


Porównania
**********

Dopasowanie wykorzystujące operatory porównań może być zdefiniowane przy pomocy matcherów ``Eq()``, ``Ne()``, ``Lt()``, ``Gt()``:

.. code-block:: cpp

    EXPECT_CALL(mock, some_method(Eq(100));
    EXPECT_CALL(mock, some_method(Ne(100));
    EXPECT_CALL(mock, some_method(Lt(100));
    EXPECT_CALL(mock, some_method(Gt(100));

Dla wskaźników możemy wykorzystać obiekty ``IsNull()`` i ``NotNull()``:

.. code-block:: cpp

    EXPECT_CALL(mock, print(IsNull());
    EXPECT_CALL(mock, print(NotNull());

Porównania mogą być również ograniczane dla typów:

.. code-block:: cpp

    EXPECT_CALL(mock, some_method(TypedEq<int>(100));
    EXPECT_CALL(mock, some_method(Matcher<int>(Gt(50)));

Dopasowanie łańcuchów znaków
****************************

Łańcuchy znaków (C-strings i std::string) mogą być dopasowywane przy pomocy następujących obiektów:

* ``ContainsRegex(string)``
* ``EndsWith(suffix)``
* ``HasSubstr(string)``
* ``MatchesRegex(string)``
* ``StartsWith(prefix)``
* ``StrCaseEq(string)``
* ``StrCaseNe(string)``
* ``StrEq(string)``
* ``StrNe(string)``

Łączenie wielu porównań
***********************

Łączenie wielu obiektów porównujących może się odbywać przy pomocy obiektów:

* ``AllOf(m1, m2, ...)``
* ``AnyOf(m1, m2, ...)``
* ``Not(m)``

.. code-block:: cpp

    EXPECT_CALL(mock, some_method(AllOf(NotNull(), Not(StrEq("")), 5));

Dopasowanie dla pól i getterów obiektów
***************************************

* ``Field(&class::field, m)``
* ``Property(&class::property, m)``
* ``Key(v/m)`` - ``EXPECT_CALL(my_map, Contains(Key(42)))``
* ``Pair(m1, m2)``

.. code-block:: cpp

    struct Gadget
    {
        int id;
    };

    struct MockUser
    {
        MOCK_METHOD(void, use, (Gadget&));
    };

    MockUser user;

    EXPECT_CALL(user, use(Field(&Gadget::id, Gt(0))).Times(2);

    Gadget g1{10};    
    user.use(g1); // ok

    Gadget g2{-1};
    user.use(g2); // report error

Dopasowania dla kontenerów
**************************

Dopasowania dla całych kontenrów:

* ``ContainerEq(other)``
* ``IsEmpty()``
* ``Size(m)``
* ``Contains(e)``
* ``Each(e)``

Dopasowania dla kolekcji elementów:

* ``ElementsAre(e0, e1, ...)``
* ``ElementsAreArray({...})``
* ``Pointwise(m, container)``
* ``UnorderedElementsAre(...)``
* ``WhenSorted(m)``
* ``WhenSortedBy(comparator, m)``

Przykłady:

.. code-block:: cpp

    MOCK_METHOD1(save_data, void(const vector<int>& numbers));

    EXPECT_CALL(mock, save_data(UnorderedElementsAre(1, Gt(0), _, 5)));

    vector<int> data = { 1, 10, -100, 5 };
    mock.save_data(data); // ok

Dopsowania wieloargumentowe
***************************

Czasami wymagane zdefiniowanie wzajemnie zależnych wymagań dla argumentów wywołania mockowanej funkcji:

.. code-block:: cpp

    EXPECT_CALL(mock, some_method(_, _)).With(Eq());
    EXPECT_CALL(mock, some_method(_, _, _)).With(AllArgs(Eq())); // the same as above
    EXPECT_CALL(mock, some_method(_, _, _, _)).With(Args<0, 3>(Eq()));

Tworzenie własnych obiektów dopasowujących
******************************************

Tworzenie własnych obiektów dopasowujących jest możliwe na trzy sposoby:

#. Używając funkcji ``Truly(predicate)``

   .. code-block:: cpp

        int is_even(int n) { return (n % 2) == 0 ? 1 : 0; }

        // some_method() must be called with an even number.
        EXPECT_CALL(mock, some_method(Truly(is_even)));

#. Pisząc makro ``MATCHER()``

   .. code-block:: cpp
    
        MATCHER(IsEven, std::string(negation ? "isn't" : "is") + " even") { return arg % 2 == 0; }

        MATCHER_P(IsDivisible, value, std::string(negation ? "isn't" : "is")
            + " divisible by " + std::to_string(value)) { return arg % value == 0; }

        MATCHER_P2(InCloseRange, low, high, std::to_string(arg)  
            + std::string(negation ? " isn not" : " is") 
            + " in range [" + std::to_string(low) + ", " 
            + std::to_string(high) + "]") {
            return low <= arg && arg >= high;
        }

#. Pisząc własną klasę dziedziczącą po ``MatcherInterface``

Wykorzystanie obiektów dopasowujących w asercjach testów
********************************************************

Obiekty porównujące mogą być wykorzystane w asercjach testów jednostkowych:

.. code-block:: cpp

    ASSERT_THAT(result, AllOf(NotNull(), StrNe("")));

    EXPECT_THAT(result, AnyOf(Gt(100), Le(-100));

Mockowanie metod niewirtualnych
-------------------------------

Bibliotek gMock umożliwia mockowanie metod, które nie są wirtualne. Możemy to wykorzystać w testach kodu, który wykorzystuje *Hi-performance Dependency Injection*.

W takim przypadku klasa mocka nie dziedziczy po interfejsie (lub klasie z metodami wirtualnymi). 

.. code-block:: c++

    // A simple logger class.  None of its members is virtual.
    class Logger 
    {
    public:
        void log(const std::string& message);
        void warn(const std::string& warning);
        void error(const std::string& error_msg, std::error_code err_code);
    };

Klasa mocka dla klasy logger:

.. code-block:: c++

    class MockLogger 
    {
    public:
        MOCK_METHOD(void, log, (const std::string& message));
        MOCK_METHOD(void, warn, (const std::string& warning));
        MOCK_METHOD(void, error, (const std::string& error_msg, std::error_code err_code));
    };

Aby mieć możliwość podstawienia mocka dla potrzeb testów, musimy wykorzystać *static polymorphism* i szablony.

.. code-block:: c++

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
        {}

        void read_data(const std::string& cmd)
        {
            //...
            if (logger_)
                logger_->log("Command "s + cmd + " has been executed");
        }
    };

Powyższy kod daje możliwość zastosowania konkretnej implementacji logger'a w kodzie produkcyjnym (np: ``FileLogger``):

.. code-block:: c++

    FileLogger logger{"log.dat"};

    auto conn = create_connection("http://localhost:8000", &logger);

    DataReader<FileLogger> reader{&logger};
    reader.read_data("select * from table");

W teście możemy natomiast wykorzystać klasę mocka:

.. code-block:: c++

    MockLogger mock_logger;
    EXPECT_CALL(mock_logger, /*...*/);
    // set more expectations on mock_logger...
  
    DataReader<MockLogger> reader(&mock_logger);
    // exercise reader...