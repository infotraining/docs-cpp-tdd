# Wzorce testow jednostkowych

* Wzorce do tworzenia asercji
* Wzorce do budowania fikstur
* Wzorce do klas przypadkow testowych

## Four-Phase Test

Kazdy test sklada sie z czterech wykonywanych sekwencyjnie faz.

1. **Setup**. W fazie pierwszej ustawiana jest fikstura testu. Ta faza jest wymagana do tego, aby testowany system - SUT (*System Under Test*) wykazal sie oczekiwanym zachowaniem. W fazie inicjalizacyjnej konfigurowane sa obiekty pozorujace (mock).
2. **Exercise**. W fazie drugiej dokonujemy interakcji z testowanym obiektem (SUT).
3. **Verify**. W fazie trzeciej dokonujemy weryfikacji oczekiwanego stanu lub zachowania testowanego systemu.
4. **Teardown**. W fazie czwartej przywracamy testowane srodowisko do stanu z przed testu.

## State Verification

Po wykonaniu testu (fazie *Exercise*) dokonywana jest inspekcja stanu testowanego systemu (SUT). Otrzymane wartosci
porownywane sa z wartosciami oczekiwanymi dla poprawnego dzialania systemu.

Najczesciej weryfikowany stan przechowywany jest w obiekcie SUT, ale zdarza sie rowniez, ze wymagane jest sprawdzenie
stanu innego obiektu bioracego udzial w tescie.

*State Verification* powinien byc uzywany, gdy interesuje nas tylko stan koncowy systemu, a nie w jaki sposob system
znalazl sie w takim stanie.

```cpp
TEST(ListTest, SizeOfListReflectsItemsAddedToIt)
{
    List<string> list;
    list.add("Nazwisko");

    ASSERT_EQ(1, list.size());
}
```

Prezentowany test przejdzie z implementacja `size()`, ktora zawsze zwraca `1`. W takim przypadku test jest wystarczajaco prosty, aby odczytac intencje jego autora.

## Guard Assertion

Czasami jednak warto zweryfikowac nie tylko stan przed, ale rowniez stan po tescie, a wiec uzyc wzorca Guard Assertion. Wzorzec Guard Assertion polega na ujawnieniu zalozen zrobionych
dla fikstury przed wywolaniem funkcjonalnosci, ktora chcemy testowac.

```cpp
TEST(ListTest, ListIsNoLongerEmptyAfterAddingAnItemToIt)
{
    List<String> list;
    ASSERT_TRUE(list.is_empty()); // guard assertion
    list.add("Tekst");
    ASSERT_FALSE(list.is_empty()); // weryfikacja stanu
}
```

*Guard Assertion* gwarantuje, ze metoda `is_empty()` zwroci poprawnie wartosc `true` dla
pustej listy zanim wywolamy metode `add()`, ktora testujemy.

Wzorzec *Guard Assertion* jest czesto uzywamy z wzorcem *Resulting State Assertion*.
Te dwa wzorce sa laczone w sekwencje, w ktorej test najpierw weryfikuje, czy stan przed
odpowiada oczekiwaniom autora testu, a potem przechodzi do wywolania funkcjonalnosci i zweryfikowania
stanu wynikowego.

Zdarza sie jednak, ze celem dodania *Guard Assertion* jest upewnienie sie, czy zalozenie dotyczace stanu poczatkowego fikstury jest prawidlowe. W takich przypadkach sensownie jest przesunac *Guard Assertion(s)* na koniec metody inicjalizujacej fiksture.

## Delta Assertion

Jesli w testach istnieje fikstura, ktorej stanu nie mozemy zakodowac na sztywno, to nie
nalezy weryfikowac stanu absolutnego po wywolaniu kodu w tescie. Lepiej sprawdzic, czy roznica
(delta) pomiedzy stanem poczatkowym a koncowym jest zgodna z naszymi oczekiwaniami.

```cpp
TEST(ListTest, SizeOfListReflectsItemsAddedToIt)
{
    List<string> list = create_list(); // create_list() dostarcza wypelniony obiekt listy
    int size_before = list.size(); // zapamietanie stanu poczatkowego

    list.add("Tekst");

    ASSERT_EQ(size_before + 1, list.size()); // weryfikacja przyrostowa
}
```

Przewaga wzorca *Delta Assertions* nad wzorcem *Resulting State Assertions* polega na tym, ze test skupia sie na tym, co jest
testowane, zamiast zwracac pozornie dowolne wartosci.

## Custom Assertion

Zdarza sie, ze dlugosc kodu weryfikujacego nasze oczekiwania przekracza dlugosc kodu
wymagana do wywolania kodu w tescie. W takim przypadku zaleca sie wyodrebnienie
metody *Custom Assertion* z testu w celu hermetyzacji zlozonej logiki weryfikacji w prostej metodzie, ktora mozemy wywolac z testu.

*Custom Assertion* jest stosowane ze wzgledu na mozliwosc wykonywania roznych typow rozmytego dopasowywania.
Na przyklad, jesli chcemy porownac dwa obiekty jedynie w oparciu o zestaw wybranych wlasciwosci.

Innym zastosowaniem jest sytuacja, w ktorej obiekty nie implementuja operatora porownania w odpowiedni sposob.

Uzywanie *Custom Assertion* pozwala rowniez zdefiniowac bardziej znaczace komunikaty bledu, ktore zostana wyswietlone w razie niepowodzenia testu.

```cpp
TEST(MeetingCalendarTest, GetsNextAppointmentDate)
{
    MeetingCalendar calendar;
    // pominieto: ustaw spotkanie w kalendarzu

    Date time = calendar.get_next_appointment();

    AssertDate(time);
}

void AssertDate(Date time)
{
    // jesli obiekt czasu nie jest prawidlowy
    FAIL() << "Invalid time";
}
```

## Behavior Verification

Wzorzec *Behavior Verification* nie weryfikuje poprawnosci zwracanych wartosci, lecz sprawdza, czy nasz kod wspoldziala
z obiektami wspolpracownikami w oczekiwany przez nas sposob. Kazdy test weryfikuje jakie metody, i w jaki sposob, sa
wywolywane na wspolpracownikach przez testowany obiekt (SUT).

Typowym zastosowaniem wzorca *Behavior Verification* jest rozwijanie aplikacji w stylu "outside-in".
W takim przypadku weryfikujemy wywolania na obiektach, ktore nie maja jeszcze implementacji (obiektach Mock).

```{important}
W jednym tescie mozemy uzywac zarowno stylu asercji opartego na stanie, jak i stylu opartego na interakcji.
```

Przyklad:

```cpp
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
```

## Object Mother

**Object Mother** jest specjalizowana fabryka, ktorej rola jest dostarczenie istotnych dla testu danych. Dane te sa
przez fabryke odpowiednio skonfigurowane.

```cpp
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
```

Test jednostkowy wykorzystuje obiekt `OrderObjectMother` do przejrzystego utworzenia obiektu potrzebnego w logice testu.

```cpp
TEST_F(FlightServiceTests, CanAddReservationToRepository)
{
    auto reservation_request = Mother::create_reservation_request();

    EXPECT_CALL(flight_repository_, add(reservation_request.flight)).Times(1);

    sut_.make_reservation(reservation_request);
}
```

## Builder Object

Wzorzec *Object Mother* nie najlepiej sprawdza sie w sytuacji, kiedy musimy uwzglednic wiele wariacji danych testowych.
Lepszym rozwiazaniem jest dynamiczne budowanie takich danych na zadanie z wykorzystaniem wzorca **Test Builder**.
Obiekty **Test Builder** sa implementacja klasycznego wzorca Budowniczy (GOF), ktory umozliwia zbudowanie zlozonego obiektu poprzez
kolejne wywolania metod budowniczego i odebranie finalnego obiektu przez metode `get()`.

```cpp
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
```

Test korzystajacy z budowniczego wyglada nastepujaco:

```cpp
TEST_F(FlightServiceTests, ThrowsWhenTimestampInInvalidFormat)
{
    ReservationRequestBuilder reservation_request_builder;
    reservation_request_builder.with_timestamp("2017|01|01 1:45am");
    auto reservation_request = reservation_request_builder.get_reservation_request();

    EXPECT_THROW(sut_.make_reservation(reservation_request), std::invalid_argument);
}
```
