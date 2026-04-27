Wzorce testów jednostkowych
===========================

* Wzorce do tworzenia asercji
* Wzorce do budowania fikstur
* Wzorce do klas przypadków testowych

Four-Phase Test
---------------

Każdy test składa się z czterech wykonywanych sekwencyjnie faz.

1. **Setup**. W fazie pierwszej ustawiana jest fikstura testu. Ta faza jest wymagana do tego, aby testowany system  - SUT (*System Under Test*) wykazał się oczekiwanym zachowaniem. W fazie inicjalizacyjnej konfigurowane są obiekty pozorujące (mock).
2. **Exercise**. W fazie drugiej dokonujemy interakcji z testowanym obiektem (SUT).
3. **Verify**. W fazie trzeciej dokonujemy weryfikacji oczekiwanego stanu lub zachowania testowanego systemu.
4. **Teardown**. W fazie czwartej przywracamy testowane środowisko do stanu z przed testu.


State Verification
------------------

Po wykonaniu testu (fazie *Exercise*) dokonywana jest inspekcja stanu testowanego systemu (SUT). Otrzymane wartości
porównywane są z wartościami oczekiwanymi dla poprawnego działania systemu.

Najczęściej weryfikowany stan przechowywany jest w obiekcie SUT, ale zdarza się również, że wymagane jest sprawdzenie
stanu innego obiektu biorącego udział w teście.

*State Verification* powinien być używany, gdy interesuje nas tylko stan końcowy systemu, a nie w jaki sposób system
znalazł się w takim stanie.


.. code-block:: cpp

    TEST(ListTest, SizeOfListReflectsItemsAddedToIt)
    {
        List<string> list;
        list.add("Nazwisko");

        ASSERT_EQ(1, list.size());
    }

Prezentowany test przejdzie z implementacją ``size()``, która zawsze zwraca ``1``. W takim przypadku test jest wystarczająco prosty, aby odczytać intencję jego autora.


Guard Assertion
---------------

Czasami jednak warto zweryfikować nie tylko stan przed, ale również stan po teście, a więc użyć wzorca Guard Assertion. Wzorzec Guard Assertion polega na ujawnieniu założeń zrobionych
dla fikstury przed wywołaniem funkcjonalności, którą chcemy testować.

.. code-block:: cpp

    TEST(ListTest, ListIsNoLongerEmptyAfterAddingAnItemToIt)
    {
        List<String> list;
        ASSERT_TRUE(list.is_empty()); // guard assertion 
        list.add("Tekst");
        ASSERT_FALSE(list.is_empty()); // weryfikacja stanu
    }

*Guard Assertion* gwarantuje, że metoda ``is_empty()`` zwróci poprawnie wartość ``true`` dla
pustej listy zanim wywołamy metodę ``add()``, którą testujemy.

Wzorzec *Guard Assertion* jest często używamy z wzorcem *Resulting State Assertion*.
Te dwa wzorce są łączone w sekwencję, w której test najpierw weryfikuje, czy stan przed 
odpowiada oczekiwaniom autora testu, a potem przechodzi do wywołania funkcjonalności i zweryfikowania
stanu wynikowego. 

Zdarza się jednak, że celem dodania *Guard Assertion* jest upewnienie się, czy założenie dotyczące stanu początkowego fikstury jest prawidłowe. W takich przypadkach sensownie jest przesunąć *Guard Assertion(s)* na koniec metody inicjalizującej fiksturę.

Delta Assertion
---------------

Jeśli w testach istnieje fikstura, której stanu nie możemy zakodować na sztywno, to nie 
należy weryfikować stanu absolutnego po wywołaniu kodu w teście. Lepiej sprawdzić, czy różnica
(delta) pomiędzy stanem początkowym a końcowym jest zgodna z naszymi oczekiwaniami.

.. code-block:: cpp

    TEST(ListTest, SizeOfListReflectsItemsAddedToIt)
    {
        List<string> list = create_list(); // create_list() dostarcza wypełniony obiekt listy
        int size_before = list.size(); // zapamiętanie stanu początkowego
        
        list.add("Tekst");
        
        ASSERT_EQ(size_before + 1, list.size()); // weryfikacja przyrostowa
    }

Przewaga wzorca *Delta Assertions* nad wzorcem *Resulting State Assertions* polega na tym, że test skupia się na tym, co jest 
testowane, zamiast zwracać pozornie dowolne wartości.

Custom Assertion
----------------

Zdarza się, że długość kod weryfikującego nasze oczekiwania przekracza długość kodu 
wymaganą do wywołania kodu w teście. W takim przypadku zaleca się wyodrębnienie 
metody *Custom Assertion* z testu w celu hermetyzacji złożonej logiki weryfikacji w prostej metodzie, którą możemy wywołać z testu.

*Custom Assertion* jest stosowane ze względu na możliwość wykonywania różnych typów rozmytego dopasowywania. 
Na przykład, jeśli chcemy porównać dwa obiekty jedynie w oparciu o zestaw wybranych właściwości.

Innym zastosowaniem jest sytuacja, w której obiekty nie implementują operatora porównania w odpowiedni sposób. 

Używanie *Custom Assertion* pozwala również zdefiniować bardziej znaczące komunikaty błędu, które zostaną wyświetlone w razie niepowodzenia testu.


.. code-block:: cpp

    TEST(MeetingCalendarTest, GetsNextAppointmentDate)
    {
        MeetingCalendar calendar;
        // pominięto: ustaw spotkanie w kalendarzu
        
        Date time = calendar.get_next_appointment();
        
        AssertDate(time);
    }

    void AssertDate(Date time) 
    {
        // jeśli obiekt czasu nie jest prawidłowy
        FAIL() << "Invalid time";
    }

Behavior Verification
---------------------

Wzorzec *Behavior Verification* nie weryfikuje poprawności zwracanych wartości, lecz sprawdza, czy nasz kod współdziała
z obiektami współpracownikami w oczekiwany przez nas sposób. Każdy test weryfikuje jakie metody, i w jaki sposób, są
wywoływane na współpracownikach przez testowany obiekt (SUT).

Typowym zastosowaniem wzorca *Behavior Verification* jest rozwijanie aplikacji w stylu "outside-in".
W takim przypadku weryfikujemy wywołania na obiektach, które nie mają jeszcze implementacji (obiektach Mock).

.. important:: W jednym teście możemy używać zarówno stylu asercji opartego na stanie, jak i stylu opartego na interakcji.

Przykład:

.. code:: cpp

    class FlightRepository
    {
    public:
        virtual ~FlightRepository() = default;
        virtual void add(const Flight& flight) = 0;
    };

    class MockFlightRepository : public FlightRepository
    {
    public:
        MOCK_METHOD1(add, void (const Flight&));
    };

    class FlightServiceTests : public ::testing::Test
    {
    protected:
        MockFlightRepository flight_repository_;
        FlightReservationService sut_;

    public:
        FlightServiceTests() : sut_{flight_repository_}
        {}
    };

    TEST_F(FlightServiceTests, CanAddReservationToRepository)
    {
        auto reservation_request = Mother::create_reservation_request();

        EXPECT_CALL(flight_repository_, add(reservation_request.flight)).Times(1);

        sut_.make_reservation(reservation_request);
    }

Object Mother
-------------

**Object Mother** jest specjalizowaną fabryką, której rolą jest dostarczenie istotnych dla testu danych. Dane te są
przez fabrykę odpowiednio skonfigurowane.

.. code:: c++

    struct Mother
    {
        constexpr static const char* flight_no = "LOT101";
        constexpr static const char* client = "John Newman";
        constexpr static const char* timestamp = "2017/01/01 1:45am";

        static ReservationRequest create_reservation_request()
        {
            return ReservationRequest{Flight{flight_no, 100.0}, client, timestamp};
        }
    };


Test jednostkowy wykorzystuje obiekt ``OrderObjectMother`` do przejrzystego utworzenia obiektu potrzebnego w logice testu.

.. code:: c++

    TEST_F(FlightServiceTests, CanAddReservationToRepository)
    {
        auto reservation_request = Mother::create_reservation_request();

        EXPECT_CALL(flight_repository_, add(reservation_request.flight)).Times(1);

        sut_.make_reservation(reservation_request);
    }


Builder Object
--------------

Wzorzec *Object Mother* nie najlepiej sprawdza się w sytuacji, kiedy musimy uwzględnić wiele wariacji danych testowych.
Lepszym rozwiązaniem jest dynamiczne budowanie takich danych na żądanie z wykorzystaniem wzorca **Test Builder**.
Obiekty **Test Builder** są implementacją klasycznego wzorca Budowniczy (GOF), który umożliwia zbudowanie złożonego obiektu poprzez
kolejne wywołania metod budowniczego i odebranie finalnego obiektu przez metodę ``get()``.

.. code:: c++

    class ReservationRequestBuilder
    {
        constexpr static const char* flight_no = "LOT101";
        constexpr static const char* client = "John Newman";
        constexpr static const char* timestamp = "2017/01/01 1:45am";

        ReservationRequest reservation_request_{Flight{flight_no, 100.0}, client, timestamp};

    public:
        ReservationRequestBuilder() = default;

        ReservationRequestBuilder& with_client(const string& client)
        {
            reservation_request_.client = client;

            return *this;
        }

        ReservationRequestBuilder& with_timestamp(const string& timestamp)
        {
            reservation_request_.timestamp = timestamp;

            return *this;
        }

        ReservationRequestBuilder& with_flight(const Flight& flight)
        {
            reservation_request_.flight = flight;

            return *this;
        }

        ReservationRequest get_reservation_request() const
        {
            return reservation_request_;
        }
    };

Test korzystający z budowniczego wygląda następująco:

.. code:: csharp

    TEST_F(FlightServiceTests, ThrowsWhenTimestampInInvalidFormat)
    {
        ReservationRequestBuilder reservation_request_builder;
        reservation_request_builder.with_timestamp("2017|01|01 1:45am");
        auto reservation_request = reservation_request_builder.get_reservation_request();

        EXPECT_THROW(sut_.make_reservation(reservation_request), std::invalid_argument);
    }