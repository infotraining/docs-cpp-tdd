*******************
Refaktoryzacja kodu
*******************

Refaktoryzujemy zawsze, kiedy jest taka potrzeba.

Trzy sytuacje, w których musimy bezwzględnie przeprowadzić refaktoryzację:

1.  Jeśli występuje duplikacja kodu
2.  Jeśli kod i/lub jego intencja nie jest jasna
3.  Jeśli wykryjemy zły kod (*code smell*) – symptomy, że występuje problem z jakością projektu


Duplikacja kodu
---------------
Występowanie duplikacji kodu jest znacznikiem niskiej jakości kodu. Jeżeli istnieje kod, który się powtarza, należy pozbyć się duplikacji za pomocą ekstrakcji metody.

Duplikacji należy unikać również w kodzie testów jednostkowych. Często w kodzie testach powtarzane jest ustawienie obiektów do testu. Możemy wyodrębnić ten kod do metod ustawiających fiksturę testu (*test fixture*).


Niejasne intencje
-----------------
Kod jest najważniejszą wartością, którą dostarcza programista. W związku z tym powinien być tak jasny i zrozumiały, jak to tylko możliwe. Nie zawsze możemy napisać taki kod na początku, ale możemy go refaktoryzować, aby był bardziej zrozumiały.

Prostym sposobem wyjaśnienia intencji jest wybór lepszych nazw.

Stosowanie TDD pomaga w jasnym przekazaniu intencji w czasie pisania kodu, ponieważ musimy myśleć o interfejsie klasy, a nie o jej implementacji. Decydujemy, co jest najbardziej sensowne z punktu widzenia użytkownika klasy bez wchodzenia w szczegóły implementacji.


Zły kod
-------
Pojęcie złego kodu (*code smell*) jest szeroko używane przez społeczność eXtreme Programming w odniesieniu do kodu o nieakceptowalnej jakości. Termin ten został po raz pierwszy użyty przez Fowler’a i Beck’a.
    
Jeśli natrafimy na zły kod, powinniśmy przeprowadzić refaktoryzację i pozbyć się go. Zły kod nie zawsze oznacza problem, ale powinniśmy przyjrzeć się temu fragmentowi kodu, aby wyeliminować takie niebezpieczeństwo.


Przeprowadzenie refaktoryzacji
------------------------------
Niezbędne są automatyczne testy, które dadzą nam informację zwrotną, czy w czasie refaktoryzacji czegoś nie zepsuliśmy. Nie chcemy zmieniać zachowania, a testy sprawdzające zachowanie poinformują nas, jeśli taka zmiana zajdzie.

Refaktoryzacja jest przeprowadzana małymi krokami, a po każdym z nich uruchamiane są testy. Dzięki temu od razu wiemy, czy czegoś nie zepsuliśmy. Łatwo określić, co spowodowało problem – ostatni krok. Należy wycofać ten krok i spróbować ponownie.


Ekstrakcja klasy
----------------
Jeśli klasa jest za duża lub jej działanie jest niejasne, musimy podzielić ją na części mające spójne zachowanie. Ekstrakcja klasy pozwala wyodrębnić jeden z tych zestawów zachowań do nowej klasy. Przy ekstrakcji klasy należy zachować Zasadę Pojedynczej Odpowiedzialności (SRP).

Jeśli potrzebujemy powielić implementację jakiegoś zachowania, to możemy wyodrębnić kod do oddzielnej klasy. Następnie możemy wyekstrahować interfejs i napisać wymagane implementacje.

.. code-block:: cpp

    class Movie
    {
    public:
        void write_to(std::ostream& destination)
        {
            destination << get_name() << " " << get_category() << endl;
        }
        // ...
    };

    class MovieList
    {
        std::vector<Movie> movies;
    public:
        void write_to(std::ostream& destination)
        {
            for (const auto& m : movies) 
                m.write_to(destination);                    
        }
    };


Wyodrębniamy nową klasę ``MovieWriter``, która odpowiada za jeden zakres funkcjonalności.

.. code-block:: cpp

    class MovieWriter
    {
    public:
        void write_to(std::ostream& destination, const Movie& movie)
        {
            destination << movie.get_name() << " " << movie.get_category() << endl;
        }
        
        void write_to(std::ostream& destination, const std::vector<Movie>& movies)
        {
            for (const auto& m : movies)
                    write_to(destination, m);
        }
    };


Zastąpienie instancji klasy obiektem std::function
--------------------------------------------------

Jeżeli klasa (abstrakcyjna) posiada tylko jedną metodę, wówczas jest spore prawdopodobieństwo, że taką klasę można zastąpić funktorem lub ``std::function``.

.. code-block:: cpp

    class ShapeCreator
    {
    public:
        virtual ~ShapeCreator() = defualt;
        virtual std::unique_ptr<Shape> create() = 0;
    };

    class RectangleCreator : public ShapeCreator
    {
    public: 
        std::unique_ptr<Rectangle> create() override
        {
            return std::make_unique<Rectangle>();
        }
    };

    // using creators for mapping string on shape instance
    std::map<std::string, std::unique_ptr<ShapeCreator>> shape_factory;
    shape_factory.insert(make_pair("Rectangle"), std::make_unique<RectangleCreator>());

    auto r = shape_factory["Rectangle"]->create();


Powyższa klasa abstrakcyjna ``ShapeCreator`` zawiera tylko jedną metodę. Możemy
uprościć i uelastycznić kod zastępując ją typem ``std::function``:

.. code-block:: cpp

    using ShapeCreator = std::function<std::unique_ptr<Shape>()>;

    std::map<std::string, ShapeCreator> shape_factory;

    shape_factory.insert("Rectangle", [] { return std::make_unique<Rectangle>(); });

    auto& rectangle_factory = shape_factory["Rectangle"];
    auto r = rectangle_factory();

Ekstrakcja interfejsu
---------------------

Aby poprawić testowalność lub uniknąć zależności od konkretnych implementacji
możemy z klasy wyekstrahować jej interfejs:

.. code-block:: cpp

    class Logger 
    {
        std::ofstream* out;
    public:
        //...

        void write_log(const string& message) 
        {
            *out << "Log: " << message << "\n";
        }
    };

Po refaktoringu:

.. code-block:: cpp

    class ILogger 
    {
    public:
        virtual ~ILogger() = default;
        virtual void write_log(const string& message) = 0;
        
    };

    class Logger : public ILogger 
    {
        std::ofstream out;
    public:
        //...

        void write_log(const string& message) 
        {
            out << "Log: " << message << "\n";
        }
    };


Ekstrakcja metody lub funkcji
-----------------------------

Ekstrakcję metody lub funkcji stosujemy, jeśli kod metody jest zbyt długi lub występują w niej komentarze, które wskazują cel implementacji.

Metoda przed refaktoringiem:

.. code-block:: cpp

    void process_file(const char* file_name)
    {
        std::ifstream file(file_name);

        // oblicz rozmiar pliku i utwórz bufor znaków
        file.seek(0, ios::end);
        const int file_size = file.tellg();
        std::string buffer(file_size, '\0')
        
        // wczytaj całą zawartość pliku do bufora znaków
        file.seekg(0, ios::beg);
        file.read(buffer.begin(), file_size);
        file.close();
        
        // przetwórz bufor znaków    
        //...
    }

Efekt refaktoringu przy pomocy ekstrakcji metody:

.. code-block:: cpp

    void process_file(const char* file_name)
    {
        const auto content = read_file_to_string(file_name);	
        
        // przetwórz bufor znaków    
        //…
    }

    std::string read_file_to_string(const char* file_name) 
    {
        std::ifstream file(file_name);
        
        // oblicz rozmiar pliku i utwórz bufor znaków
        file.seek(0, ios::end);
        const int file_size = file.tellg();
        std::string buffer(file_size, '\0')
        
        // wczytaj całą zawartość pliku do bufora znaków
        file.seekg(0, ios::beg);
        file.read(buffer.begin(), file_size);  
        
        return buffer;
    }

Zastępowanie *type-code* podklasami
-----------------------------------

Tę refaktoryzację stosujemy w sytuacji, gdy istnieje klasa, która wykorzystuje typy proste jako identyfikatory wykorzystywane przy implementacji zachowania.

* Tworzymy podklasy dla każdego identyfikatora
* Zastępujemy sekwencję instrukcji ``if`` polimorfizmem

.. code-block:: cpp

    class Employee
    {
        EmployeeType type_;
    public:
        enum class EmployeeType { engineer, salesman };

        double calculate_pay()
        {
            double salary{};

            if (type_ == EmployeeType::engineer)
                salary += engineer_bonus();
            else if (type_ == Salesman)
                salary += salesman_bonus();

            return salary;
        }
    };


Po refaktoryzacji:

.. code-block:: cpp

    class Employee
    {
    public:
        virtual double calculate_pay() = 0;
        virtual ~Employee() = default;
    };

    class Engineer : public Employee {};
    class Salesman : public Employee {};


Zastępowanie wyrażenia warunkowego polimorfizmem
------------------------------------------------

Wyrażenia warunkowe można zastąpić podklasami obsługującymi różne przypadki. Jeśli istnieją już podklasy, to można rozważyć umieszczenie w nich zachowania warunkowego.


Wprowadzenie zmiennej opisującej
--------------------------------

W przypadku złożonego wyrażenia, które jest trudne do zrozumienia, możemy wyodrębnić jego części i przechować pośrednie wyniki w dobrze nazwanych zmiennych tymczasowych. Dzięki temu uzyskujemy łatwe do zrozumienia części, a całe wyrażenie jest jaśniejsze.

.. code-block:: cpp

    double calculate_total()
    {
        return (get_subtotal() + (get_taxable_subtotal() * 0.15)
                - get_subtotal()) > 100.0 ? (get_subtotal() * 0.10) : 0;
    }


Po refaktoringu:

.. code-block:: cpp

    double calculate_total()
    {
        double subtotal = get_subtotal(); 
        double tax = get_taxable_subtotal() * 0.15;
        double total = subtotal + tax;
        bool qualifies_for_discount = get_subtotal() > 100.0;
        double discount = qualifies_for_discount ? subtotal * 0.10 : 0.0;
        return total - discount;
    }


Zastępowanie dziedziczenia delegowaniem
---------------------------------------

Dziedziczenie powinno być używane tylko, jeśli podklasy są specjalnymi rodzajami klasy nadrzędnej lub rozszerzają ją, a nie jedynie nadpisują jej części.
Jeśli dziedziczenie jest stosowane tylko w celu ponownego użycia pewnych funkcjonalności klasy nadrzędnej, to powinno zostać zastąpione delegowaniem.

.. code-block:: cpp
 
    class MovieList : public std::vector<Movie>
    {
        //…
    };

.. code-block:: cpp

    class MovieList
    {
        std::vector<Movie> movies_;
    public:
        void add(const Movie& m);
    };


Zastępowanie magicznej wartości stałą symboliczną
-------------------------------------------------

Nie zaleca się używania w kodzie wartości literalnych zakodowanych na sztywno. Trudniej je zauważyć i są rażącą duplikacją, a ich zmiana to tzw. shotgun surgery. Zamiast nich należy używać dobrze nazwanych stałych symbolicznych. Zmiana wartości to edycja tylko w jednym miejscu.
To zalecenie jest bardziej ogólne i odnosi się do dowolnej wartości literalnej, np. string.

.. code-block:: python

    double distance_in_km = 1.609344 * distance_in_miles;

.. code-block:: python

    constexpr double km_per_mile = 1.609344;

    float distance_in_km = km_per_mile * distance_in_miles;


Refaktoryzacja do wzorców
-------------------------

Wzorce projektowe są esencją sprawdzonych idei projektowych. Powinniśmy poznać jak najwięcej wzorców oraz wiedzieć, kiedy ich używać, a kiedy nie. Jeśli nie znamy wzorców projektowych lub ich nie używamy, tworzony kod może nie wykazywać cech dobrego kodu obiektowego. Nie dostrzeżemy tak łatwo podobieństw i może się okazać, że rozwiązujemy ponownie te same problemy. Znajomość wzorców pomaga rozpoznać powtarzające się problemy i ułatwia ich rozwiązanie.


Niebezpieczeństwem wzorców projektowych jest ich nadużywanie

* Dostrzeganie wzorca wokół każdego wymagania
* Projektowanie z użyciem wzorców jako wyniku


Jak należy używać wzorców?

* Jako celów dla refaktoryzacji
* Nie należy używać ich na samym początku projektu, lecz w razie potrzeby wprowadzać stopniowo w czasie refaktoryzacji


Wzorce projektowe najczęściej występujące w fazie refaktoringu:

* Factory Method
* Strategy
* State
* Facade
* Command
* Template Method
* Decorator