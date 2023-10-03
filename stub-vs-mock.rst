Stub Vs. Mock
=============

Stub - weryfikacja stanu
------------------------

Załóżmy, że chcemy złożyć zamówienie i zrealizować je pobierając obiekty z 
repozytorium ``Warehouse``. 
Klasa ``Order`` posiada składowe ``product()`` i ``quantity()``. Obiekt ``Warehouse`` przechowuje informacje o ilości dostępnych produktów. 
Jeśli złożymy zamówienie, możliwe są dwa scenariusze:

1. Jeśli ilość produktów na stanie jest wystarczająca, zamówienie jest wypełniane i następuje aktualizacja liczby dostępnych produktów na stanie.
2. Jeśli ilość produktów na stanie nie jest wystarczająca, zamówienie nie jest wypełniane i nie zmienia się liczba dostępnych produktów na stanie.


Te dwa przypadki są testowane poniżej:

.. code:: cpp
    
    class OrderStateTests : public ::testing::Test
    {
    protected:
        const string talisker = "Talisker";
        const string highland_park = "Highland Park";

        WarehouseImpl warehouse_;
        Order order_;

    public:
        OrderStateTests()
        {
            warehouse_.add(talisker, 50);
            warehouse_.add(highland_park, 25);
        }
    };

    TEST_F(OrderStateTests, OrderIsFilledIfEnoughInWarehouse)
    {
        Order order{talisker, 50};

        order.fill(warehouse_);

        ASSERT_TRUE(order.is_filled());
    }

    TEST_F(OrderStateTests, OrderIsNotFilledIfNotEnoughInWarehouse)
    {
        Order order{talisker, 51u};

        order.fill(warehouse_);

        ASSERT_FALSE(order.is_filled());
    }

    TEST_F(OrderStateTests, IfEnoughInStockItemsAreTransferedFromWarehouse)
    {
        Order order{talisker, 50u};

        order.fill(warehouse_);

        ASSERT_EQ(warehouse_.get_inventory(talisker), 0u);
    }

    TEST_F(OrderStateTests, IfNotEnoughInStockItemsAreNotTransferedFromWarehouse)
    {
        Order order{ talisker, 51 };

        order.fill(warehouse_);

        ASSERT_EQ(warehouse_.get_inventory(talisker), 50u);
    }


Do testów potrzebujemy obiektu ``Order`` (SUT) i obiektu współpracującego implementującego interfejs ``Warehouse``. 
Obiekt pozorujący ``WarehouseImpl`` jest wymagany, ponieważ ``Order`` wywołuje na nim metody oraz będziemy potrzebowali go do weryfikacji stanu.

Taki styl testu używa techniki weryfikacji stanu, tzn. sprawdzając stan obiektu, który jest testowany oraz jego współpracowników określamy, czy wywołane metody zadziałały prawidłowo.

Mock - weryfikacja zachowania
-----------------------------

.. code:: cpp

    struct MockWarehouse : Warehouse
    {
        MOCK_CONST_METHOD2(has_inventory, bool (const std::string&, size_t));
        MOCK_METHOD2(add, void (const std::string&, size_t ));
        MOCK_METHOD2(remove, void (const std::string&, size_t));
        MOCK_CONST_METHOD1(get_inventory, size_t (const std::string&));
    };

    using ::testing::Return;
    using ::testing::_;

    class OrderInteractionsTests : public ::testing::Test
    {
    protected:
        const string talisker = "Talisker";
        const string highland_park = "Highland Park";

        MockWarehouse warehouse_;

    public:
        OrderInteractionsTests() = default;
    };

    TEST_F(OrderInteractionsTests, FillingOrderRemovesInventoryIfEnoughInStock)
    {
        Order order{talisker, 50};

        ON_CALL(warehouse_, has_inventory(talisker, 50)).WillByDefault(Return(true));
        EXPECT_CALL(warehouse_, remove(talisker, 50)).Times(1);

        order.fill(warehouse_);
    }

    TEST_F(OrderInteractionsTests, OrderIsFilledIfEnoughInStock)
    {
        Order order{talisker, 50};

        ON_CALL(warehouse_, has_inventory(talisker, 50)).WillByDefault(Return(true));
        EXPECT_CALL(warehouse_, remove(_, _));

        order.fill(warehouse_);

        ASSERT_TRUE(order.is_filled());
    }

    TEST_F(OrderInteractionsTests, FillingOrderDoesNotRemoveInventoryIfNotEnoughInStock)
    {
        Order order{talisker, 50};

        ON_CALL(warehouse_, has_inventory(talisker, 50)).WillByDefault(Return(false));
        EXPECT_CALL(warehouse_, remove(_, _)).Times(0);

        order.fill(warehouse_);
    }

    TEST_F(OrderInteractionsTests, OrderIsNotFilledIfNotEnoughInStock)
    {
        Order order{talisker, 50};

        ON_CALL(warehouse_, has_inventory(talisker, 50)).WillByDefault(Return(false));

        order.fill(warehouse_);

        ASSERT_FALSE(order.is_filled());
    }


W powyższych testach faza setupu jest zupełnie inna. Możemy wyodrębnić w niej dwa etapy:

1. Konfiguracja danych. 
     Konfigurujemy, jakie dane są zwracane z obietu współpracującego.
2. Konfiguracja wymagań. 
     Konfigurujemy, jakie są nasze oczekiwania dotyczące interakcji z obiektem pozorującym - jakie metody oraz z jakimi argumentami mają być wywołane.


Różne są też tworzone obiekty. Obiekt SUT jest taki sam, ale jego współpracownik jest obiektem pozorującym (mockiem) utworzonym za pomocą biblioteki *googlemock*.


Kiedy konfiguracja jest zakończona, testujemy obiekt SUT (``Order``). 
Po teście sprawdzamy asercją stan SUT oraz weryfikujemy zachowanie obiektu mock'a (czy zostały wywołane na nim oczekiwane metody).
