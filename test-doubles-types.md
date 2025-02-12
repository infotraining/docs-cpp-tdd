# Rodzaje obiektów pozorujących

W testach jednostkowych rozróżniamy kilka rodzajów obiektów pozorujących:

| Typ obiektu | Opis                                                                                                                                                  |
|-------------|-------------------------------------------------------------------------------------------------------------------------------------------------------|
| Dummy       | Obiekt, który jest przekazywany do metody, ale nie jest używany. Może być używany jako zastępcza wartość, aby spełnić wymagania interfejsu.           |
| Stub        | Najprostsza implementacja interfejsu. Metody stub’a zwykle zwracają zakodowane na sztywno wartości.                                                   |
| Fake        | Bardziej zaawansowana konstrukcja niż stub. Alternatywna implementacja interfejsu. Fake wygląda i działa jak prawdziwy obiekt, a stub tylko wygląda.  |
| Spy         | Obiekt, który zapisuje informacje o wywołaniach metod, takie jak parametry, liczba wywołań, itp. Może być używany do weryfikacji zachowania.          |
| Mock        | Najbardziej zaawansowany obiekt pozorujący. Mock używa asercji do sprawdzenia oczekiwanej współpracy z innymi obiektami w czasie testu. W zależności od implementacji, może zwracać zakodowane na sztywno wartości lub dostarczać naśladujące implementacje logiki. Zwykle jest generowany za pomocą odpowiednich frameworków i bibliotek takich jak gmock, ale może być również implementowany ręcznie. |


## Stub

Obiekty typu **Stub** dostarczają predefiniowanych wartości na potrzeby testu:

```c++
class StubPricingService : public PricingService
{
public:
    double get_discount_percentage(const Customer& c, const Product& p) override
    {
        return 10.0;
    }
};
```

## Fake

```c++
class FakePricingService : public PricingService
{
public:
    double get_discount_percentage(const Customer& c, const Product& p) override
    {
        if (p.is_discountable())
        {
            if (c.is_vip())
                return 20.0;

            return 10.0;
        }
        
        return 5.0;
    }
};
```

## Spy

```c++
class SpyPricingService : public PricingService
{
public:
    double get_discount_percentage(const Customer& c, const Product& p) override
    {
        calls_count++;
        return 10.0;
    }

    int calls_count = 0;
};
```