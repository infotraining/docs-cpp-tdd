# Testy jednostkowe - *Unit Tests*

Posiadanie zestawu testów jednostkowych obejmujących kod produkcyjny jest kluczowe w utrzymaniu projektu.
Testy znacznie zwiększają możliwości rozwijania aplikacji, ponieważ pozwalają na zmiany.

## Cele testów jednostkowych

* Testy powinny pomagać w poprawianiu jakości

  * Pomagają specyfikować zachowanie systemu w różnych scenariuszach definiowanych w uruchamialnej formie ("executable specification")
  * Wyłapują i lokalizują błędy

* Testy powinny pomóc w zrozumieniu systemu

  * Działają jak dokumentacja

* Testy powinny redukować ryzyko

  * Umożliwiają bezpieczne przeprowadzenie refaktoringu

* Testy powinny być łatwe w uruchamianiu

  * Powinny być szybkie, w pełni zautomatyzowane i powtarzalne (patrz FIRST)

* Testy powinny być łatwe w pisaniu i utrzymaniu

  * Powinny być proste i czytelne - testy stają się zbyt skomplikowane, gdy chcemy w pojedynczym teście zweryfikować więcej
    niż jedną funkcjonalność

* Testy powinny wymagań niewielkiego nakładu pracy przy ich utrzymaniu w trakcie rozwoju systemu

## Atrybuty testów jednostkowych - F.I.R.S.T.

Dobre testy jednostkowe powinny spełniać pięć zasad:

* **Szybkie (Fast)** - Testy powinny być szybkie.
* **Niezależne (Independent)** - Testy nie powinny zależeć od siebie.
* **Powtarzalne (Repeatable)** - Testy powinny być powtarzalne w każdym środowisku.
* **Samokontrolujące (Self-Validating)** - Testy powinny mieć jeden parametr wyjściowy typu logicznego. Mogą się powieść albo nie.
* **O czasie (Timely)** - Testy powinny być pisane w odpowiednim momencie. Testy jednostkowe powinny być pisane bezpośrednio przed tworzeniem kodu produkcyjnego.

## Zasady testów jednostkowych

1. Najpierw napisz test
2. Projektuj pod kątem testów
3. Komunikuj intencje - nazwa testu powinna wyjaśniać jego intencje
4. Izoluj testowany system ("System Under Test" - SUT) - jeśli testowany system odwołuje się do trudnych w kontrolowaniu
   zależności, takich jak baza danych, sieć, system plików, należy je zastąpić obiektami pozorującymi (*Test Double*)
5. Unikaj w kodzie produkcyjnym zależności od testów - kod produkcyjny nie powinien zawierać instrukcji warunkowych typu
   `if (testing) then`
6. Weryfikuj jedną koncepcję na test

   * jedna asercja na test - ułatwia nazywanie testów, ale prowadzi do konieczności tworzenia wielu testów, jeśli mamy
     zweryfikować stan wielu pól dla SUT
   * wiele asercji weryfikujących jeden aspekt funkcjonalności

## Organizacja testów jednostkowych

Nie ma jednego, uniwersalnego sposobu organizacji testów. W zależności od struktury kodu i celów testowania organizacja testów w projekcie może przybierać różne formy.
Możemy wyróżnić trzy podstawowe scenariusze:

### Klasa testów per testowana klasa - *Tests Per Class*

  - najprostszy przypadek - wszystkie testy danej klasy są umieszczone w jednej klasie testowej (grupie testów)
  - każda klasa produkcyjna ma odpowiadającą jej klasę testową
  - stosowana w przypadku, gdy klasa testowana jest mała i niezależna od innych klas

### Klasa testów per fikstura (konfiguracja SUT pod test) - *Tests Per Fixture*

  - testy są zorganizowane według tzw. fixture, czyli kontekstu testowego. 
  - fikstury przygotowują środowisko testowe i dane przed każdym testem. Jest to przydatne, gdy różne testy wymagają podobnych ustawień.

### Klasa testów per cecha funkcjonalności -  *Tests Per Feature*

  - klasa testowa obejmuje zbiór metod testujących kolektywnie pewną wybraną cechę funkcjonalności SUT
  - ułatwia dokumentowanie zachowania SUT w danym kontekście

## Nazewnictwo testów

Nazwy testów powinny być opisowe i jednoznaczne. Nazwa testu powinna wyjaśniać, co jest testowane i w jakich warunkach.

### Konwencje nazewnictwa testów jednostkowych

Konwencje nazewnictwa testów jednostkowych są kluczowe dla utrzymania czytelności i spójności. Jasne i opisowe nazwy testów pomagają programistom szybko zrozumieć, co test weryfikuje. Oto kilka powszechnych konwencji nazewnictwa testów jednostkowych:

#### Nazewnictwo oparte na funkcjonalności

Nazwa testu opisuje testowaną funkcjonalność.

Przykład: `TestFunctionName_StateUnderTest_ExpectedBehavior`

```c++
// Functionality-Based Naming

TEST(RecentlyUsedListTest, Add_MostRecentlyUsedUpdated) 
{
    RecentlyUsedList recently_used_list;
    recently_used_list.add("Item1");
    ASSERT_EQ("Item1", recently_used_list.get_recent_item());
}

TEST(RecentlyUsedListTest, Add_DuplicateItem_ListSizeIsNotChanged) 
{
    RecentlyUsedList recently_used_list;
    recently_used_list.add("Item1");
    EXPECT_EQ(1, recently_used_list.size());
    recently_used_list.add("Item1");
    ASSERT_EQ(1, recently_used_list.size());
}
```

#### Nazewnictwo oparte na zachowaniu

Nazwa testu opisuje oczekiwane zachowanie.

Przykład: `FunctionUnderTest_ExpectedResult`

```c++
// Behavior-Based Naming
TEST(RecentlyUsedListTest, GetRecentItem_ReturnsLastAddedItem) 
{
    RecentlyUsedList recently_used_list;
    recently_used_list.add("Item1");
    recently_used_list.add("Item2");
    EXPECT_EQ("Item2", recently_used_list.getMostRecentlyUsed());
}

TEST(RecentlyUsedListTest, GetRecentItem_ThrowsExceptionWhenListIsEmpty) 
{
    RecentlyUsedList recently_used_list;
    EXPECT_THROW(recently_used_list.getMostRecentlyUsed(), std::out_of_range);
}
```

#### Nazewnictwo Given-When-Then

Nazwa testu podąża za wzorcem ustawienia scenariusza (**Given/Arrange**), wykonania akcji (**When/Act**) i weryfikacji wyniku (**Then/Assert**).

Przykład: `GivenInitialState_WhenSomethingHappens_ThenResultIsObserved`


`````{tab-set}
````{tab-item} GTest
```c++
// Given-When-Then Naming
TEST(RecentlyUsedListTest, GivenEmptyList_WhenItemIsAdded_ThenListIsNotEmpty) 
{
    RecentlyUsedList recently_used_list;
    recently_used_list.add("Item1");
    EXPECT_FALSE(recently_used_list.empty());
}

TEST(RecentlyUsedListTest, GivenListWithItems_WhenDuplicateIsAdded_ThenListSizeIsNotChanged) 
{
    // Arrange
    RecentlyUsedList recently_used_list;
    recently_used_list.add("Item1");

    // Act
    EXPECT_EQ(1, recently_used_list.size());
    recently_used_list.add("Item1");

    // Assert
    ASSERT_EQ(1, recently_used_list.size());
}
```
````
````{tab-item} Catch2
```cpp
// Given-When-Then Naming
GIVEN("An empty list") 
{
    RecentlyUsedList recently_used_list;

    WHEN("An item is added") 
    {
        recently_used_list.add("Item1");

        THEN("The list is not empty") 
        {
            REQUIRE_FALSE(recently_used_list.empty());
        }
    }
}

TEST_CASE("List with items)
{
    RecentlyUsedList recently_used_list;
    recently_used_list.add("Item1");

    SECTION("Adding duplicate")
    {
        recently_used_list.add("Item1");
        CHECK(recently_used_list.size() == 1);
        
        SECTION("List size is not changed") 
        {
          REQUIRE(recently_used_list.size() == 1);
        }
    }
}
```
````
`````