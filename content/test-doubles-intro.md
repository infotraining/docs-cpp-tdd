# Izolowanie zależności

Załóżmy, na przykład, że istnieje metoda, która potrzebuje instancji `PricingService` w celu wykonania obliczenia
sumy zamówienia. 
Jeśli `PricingService` korzysta z bazy danych, to wykonanie metody `get_discount_percentage()` może zająć dużo czasu. 
Nie chcemy używać w testach prawdziwej klasy, jeśli wykonanie każdego testu będzie wymagać połączenia z bazę danych.

Mogą pojawić się kolejne problemy, jeśli klasa `PricingService`:

* będzie wymagać podania wielu parametrów do konfiguracji w teście
* nie została jeszcze zaimplementowana

Problemy z testowalnością obiektów mogą również wynikać z niedeterministycznego zachowania, np:

* braku dostępu do zasobów 
* implementacji funkcjonalności zależnej od czasu

Często trudno jest spowodować wyjątki z użyciem prawdziwych obiektów w testach. 
Na przykład, wyłączenie i włączenie kabla sieciowego z testu jednostkowego w celu wywołania błędu dostępu do sieci.

```{important}
Obiekt pozorujący (*test double*) jest kontrolowalnym zamiennikiem dla istniejącej zależności w systemie.
```

Aby odizolować klasę `OrderProcessor` od zależności od bazy danych możemy podstawić do testu obiekt pozorujący:

Dla klasy:

```cpp
class PricingService
{
public:
    virtual ~PricingService() = default;
    virtual double get_discount_percentage(const Customer& c, const Product& p) 
    {
        // real implementation
    }
};
```

wprowadzamy implementację obiektu pozorującego:

```cpp
struct PricingServiceTestDouble : PricingService
{
    double discount;

    explicit PricingServiceTestDouble(double discount) : discount(discount)
    {}

    double get_discount_percentage(const Customer& c, const Product& p) override
    {
        return discount;
    }
};
```

i wykorzystujemy go w teście:

```cpp
TEST(OrderProcessor, WhenProcessingOrderDiscountForCustomerIsCaclulated)
{
    // arrange
    double initial_balance = 100.0;
    double list_price = 30.0;
    double discount = 10.0;
    double expected_balance = initial_balance - list_price * (1 - discount/100.0);

    Customer customer{1, initial_balance};
    Product product("TDD", list_price);

    PricingServiceTestDouble service(discount); // creating a test double
    OrderProcessor processor(service); // sut
    Order new_order(customer, product);

    // act
    auto processed_order = processor.process(new_order);

    // assert
    ASSERT_EQ(processed_order.customer.balance, expected_balance);
}
```

W przykładzie pokazano, jak można podstawić naśladującą implementację klasy abstrakcyjnej `PricingService` do testowanej klasy `OrderProcessor`.
Dzięki temu pomijamy kosztowny krok połączenia z bazą danych oraz unikamy wykonywania długotrwałych obliczeń związanych z zachowaniem klienta, które nie są istotne z punktu widzenia testu.