Wprowadzenie do testów jednostkowych
====================================

Posiadanie zestawu testów jednostkowych obejmujących kod produkcyjny jest kluczowe w utrzymaniu projektu.
Testy znacznie zwiększają możliwości rozwijania aplikacji, ponieważ pozwalają na zmiany.

Cele testów jednostkowych
-------------------------

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

Atrybuty testów jednostkowych - F.I.R.S.T.
------------------------------------------

Dobre testy jednostkowe powinny spełniać pięć zasad:

* **Szybkie (Fast)** - Testy powinny być szybkie.
* **Niezależne (Independent)** - Testy nie powinny zależeć od siebie.
* **Powtarzalne (Repeatable)** - Testy powinny być powtarzalne w każdym środowisku.
* **Samokontrolujące (Self-Validating)** - Testy powinny mieć jeden parametr wyjściowy typu logicznego. Mogą się powieść albo nie.
* **O czasie (Timely)** - Testy powinny być pisane w odpowiednim momencie. Testy jednostkowe powinny być pisane bezpośrednio przed tworzeniem kodu produkcyjnego.


Zasady testów jednostkowych
---------------------------

#. Najpierw napisz test
#. Projektuj pod kątem testów
#. Komunikuj intencje - nazwa testu powinna wyjaśniać jego intencje
#. Izoluj testowany system ("System Under Test" - SUT) - jeśli testujemy klasę X, która zależy od klas Y i Z, klasy Y oraz Z powinny zostać
   zastąpione obiektami pozorującymi (*Test Double*)
#. Unikaj w kodzie produkcyjnym zależności od testów - kod produkcyjny nie powinien zawierać instrukcji warunkowych typu
   ``if(testing) then``
#. Weryfikuj jedną koncepcję na test

   * jedna asercja na test - ułatwia nazywanie testów, ale prowadzi do konieczności tworzenia wielu testów, jeśli mamy
     zweryfikować stan wielu pól dla SUT
   * wiele asercji weryfikujących jeden aspekt funkcjonalności


Organizacja testów jednostkowych
--------------------------------

Nie ma jednego, uniwersalnego sposobu organizacji testów.
Możemy wyróżnić trzy podstawowe scenariusze:

* **Klasa testów per testowana klasa**

  - najprostszy przypadek - wszystkie testy danej klasy są umieszczone w jednej klasie testowej

* **Klasa testów per cecha (feature)**

  - klasa testowa obejmuje zbiór metod testujących kolektywnie pewną funkcjonalność SUT
  - ułatwia dokumentowanie zachowania SUT w danym kontekście

* **Klasa testów per fikstura**

  - stosowana gdy zbiór metod testowych wymaga identycznej konfiguracji SUT