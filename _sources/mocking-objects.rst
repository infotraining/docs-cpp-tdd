Obiekty pozorujące
==================

.. important::
    Zewnętrzna zależność to obiekt w systemie, z którym komunikuje się testowany obiekt. 
    Takie zależności zwykle trudno kontrolować (np. zależność od systemu plików, wątki, czas, połączenia sieciowe).

.. important::
    Obiekt pozorujący (*test double*) jest kontrolowalnym zamiennikiem dla istniejącej zależności w systemie.


Obiekty pozorujące są alternatywnymi implementacjami interfejsów lub klas, których nie chcemy używać w teście, ponieważ:

* Są zbyt wolne
* Nie są dostępne (lub jeszcze nie istnieją)
* Zależą od zasobów, które w chwili uruchamiania testów nie są dostępne - np. połączenie sieciowe
* Trudno jest stworzyć ich instancję lub je skonfigurować dla celów testu

Załóżmy, na przykład, że istnieje metoda, która potrzebuje instancji ``PricingService`` w celu wykonania obliczenia
sumy zamówienia. 
Jeśli ``PricingService`` korzysta z bazy danych, to wykonanie metody ``get_discount_percentage()`` może zająć dużo czasu. 
Nie chcemy używać w testach prawdziwej klasy, jeśli wykonanie każdego testu będzie wymagać połączenia z bazę danych.

Mogą pojawić się kolejne problemy, jeśli klasa ``PricingService``:

* będzie wymagać podania wielu parametrów do konfiguracji w teście
* nie została jeszcze zaimplementowana

Problemy z testowalnością obiektów mogą również wynikać z niedeterministycznego zachowania, np:

* braku dostępu do zasobów 
* implementacji funkcjonalności zależnej od czasu

Często trudno jest spowodować wyjątki z użyciem prawdziwych obiektów w testach. 
Na przykład, wyłączenie i włączenie kabla sieciowego z testu jednostkowego w celu wywołania błędu dostępu do sieci.

Aby odizolować klasę ``OrderProcessor`` od zależności od bazy danych możemy podstawić do testu obiekt pozorujący:

Dla klasy:

.. code-block:: cpp

    class PricingService
    {
    public:
        virtual ~PricingService() = default;
        virtual double get_discount_percentage(const Customer& c, const Product& p) 
        {
            // real implementation
        }
    };

wprowadzamy implementację obiektu pozorującego:

.. code-block:: cpp

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

i wykorzystujemy go w teście:

.. code-block:: cpp

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

W przykładzie pokazano, jak można podstawić naśladującą implementację klasy abstrakcyjnej ``PricingService`` do testowanej klasy ``OrderProcessor``.
Dzięki temu pomijamy kosztowny krok połączenia z bazą danych oraz unikamy wykonywania długotrwałych obliczeń związanych z zachowaniem klienta, które nie są istotne z punktu widzenia testu.

Rodzaje obiektów pozorujących
-----------------------------

W testach jednostkowych rozróżniamy kilka rodzajów obiektów pozorujących:

.. tabularcolumns:: |\Y{0.3}|\Y{0.7}|

+-------------+---------------------------------------------------------------------------------------------------------------+
| Typ obiektu |                                                     Opis                                                      |
+=============+===============================================================================================================+
| Stub        | Najprostsza implementacja interfejsu. Metody stub’a zwykle zwracają zakodowane na sztywno wartości.           |
+-------------+---------------------------------------------------------------------------------------------------------------+
| Fake        | Bardziej zaawansowana konstrukcja niż stub. Alternatywna implementacja interfejsu.                            |
|             | Fake wygląda i działa jak prawdziwy obiekt, a stub tylko wygląda.                                             |
+-------------+---------------------------------------------------------------------------------------------------------------+
| Mock        | Najbardziej zaawansowany obiekt pozorujący. Mock używa asercji do sprawdzenia oczekiwanej współpracy z innymi |
|             | obiektami w czasie testu. W zależności od implementacji, może zwracać zakodowane na sztywno wartości lub      |
|             | dostarczać naśladujące implementacje logiki. Zwykle jest generowany za pomocą odpowiednich frameworków        |
|             | i bibliotek takich jak gmock, ale może być również implementowany ręcznie.                                    |
+-------------+---------------------------------------------------------------------------------------------------------------+

