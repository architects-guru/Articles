# CQRS w praktyce

### Kontekst biznesowy

Na potrzeby przykładu, założmy, że stoimy przed zadaniem zamodelowania części pewnej domeny z udziałem produktu.

Wymaganiami biznesu jest:
* tworzyć nowe produkty (model: id, name)
* wyświetlać istniejące (klucz: id)
  * informacja o tym kto utworzył produkt
  * rozwiązanie powinno być efektywne, więc do odczytu zastosujemy index zamiast bazy danych
* sprawdzać dostępność produktu (klucz: id)
  * dostępność będzie dostarczana przez zewnętrzny provider
  * operacja jest dosyć kosztowna, więc jest dostępna na żądanie (zamiast każdorazowo podczas wyświetlania produktu)
  * powinna być zawsze aktualna więc odrzucamy od razu ewentualną spójność *(ang. `Eventual consistency`)*

Na tym etapie nie znamy więcej wymagań, nie wiemy czy interakcja będzie poprzez Web API czy jakoś inaczej.


### Implementacja

Przystępujemy więc klasycznie do modelowania wejścia do domeny produktu. Wstępnie decydujemy, że jeden duży serwis zagreguje wszystkie  trzy odpowiedzialności: tworzenie, wyświetlanie oraz dostępność produktu. Powstanie więc serwis `ProductService` oraz dwa modele `CreateProduct` i `Product`.

```
public class ProductService
{
    IProductAvailabilityProvider ProductAvailabilityProvider;
    IWriteStorageProvider WriteStorageProvider;
    IReadIndexProvider ReadIndexProvider;

    Create(CreateProduct product)
    {
        // some validation
        // some domain logic
        // save to storage (WriteStorageProvider)
    }

    Product Get(string id)
    {
      // read from storage (ReadIndexProvider)
      // some projection
    }

    bool IsAvailable(string id)}
    {
        // read from external provider (ProductAvailabilityProvider)
    }
}

public class CreateProduct
{
    public string Id { get; }
    public string Name { get; }
}

public class Product : CreateProduct
{
    public string Creator { get; }
}
```

### Problemy

Spoktykamy się tutaj z kilkoma problemami:
* wiele odpowiedzialności serwisu (tworzenie, wyświetlanie, sprawdzanie dostępności), złamane *(ang. `Single Responsibility Principle`)*
* wiele zależności serwisu  (*ang. `coupling`*) - do providera zapisu, do providera odczytu, do providera który sprawdza dostępność
* każdorazowe wstrzykiwanie wszystkich zależności *np. do kontrolera API który tylko wyświetla produkt*
* zmiany w części serwisu mogą powodować koniecznośc poprawy wszystkich testów jednostkowych związanych z tym serwisem (można to obejść stosując auto-wstrzykiwanie zależności w testach ([wiecej w artykule o testach jednostkowych](link))


### Refaktoryzacja

Spróbujmy poprawić nasz serwis, żeby pozbyć się powyższych mankamentów:
* rozbicie na mniejsze serwisy, tak aby każdy z nich miał jedną odowiedzialność oraz mniej zależności
* grupując je na dwa typy gdzie kryterium będzie rodzaj operacji na danych:
  * serwisy które modyfikują dane
  * serwisy które odczytują dane

**Modyfikacja danych**
* `ProductCreationService` - będzie odpowiedzialny za tworzenie produktów

```
public class ProductCreationService
{
    IWriteStorageProvider WriteStorageProvider;

    Create(CreateProduct product)
    {
        // some validation
        // some domain logic
        // save to storage (WriteStorageProvider)
    }
}
```

**Odczyt danych**
* `ProductAvailabilityService` - będzie odpowiedzialny za sprawdzanie dostępności produktów

```
public class ProductAvailabilityService
{
    IProductAvailabilityProvider ProductAvailabilityProvider;

    bool IsAvailable(string id)}
    {
        // read from external provider (ProductAvailabilityProvider)
    }
}
```

* `ProductProjections` - będzie odpowiedzialny za projekcje produktów na potrzeby różnych widoków.

```
public class ProductProjections
{
  IReadIndexProvider ReadIndexProvider;

  Product Get(string id)
  {
    // read from storage (ReadIndexProvider)
    // some projection
  }
}
```

### Command/Query Responsibility Segregation
*`Command/Query Responsibility Segregation`* aka (`CQRS`) to wzorzec który jako kryterium nadrzędne separuje ścieżki odczytu i zapisu danych, opierając się na poglądzie, że do tych operacji używane są różne flow oraz modele. Taka separacja dosyć intuicyjnie narzucała się w przypadku naszej domeny. Oczywiście nie zawsze tak będzie, ale o tym później.

Przykład z naszej domeny potwierdza, że:
* Model zapisu jest inny niż model odczytu (projekcji)
* Walidacja po stronie zapisu danch nie występuje przy operacjach odczytu
* Operacje zapisu są całkiem agnostyczne od odczytu - zapis bezpośrednio do bazy danych, a odczyt z indeksu
* Projekcje - różne ujęcia odczytu danych, grupowanie, itp będę powodowały konieczność tworzenia nowych modeli


CQRS wprowadza nazewnictwo:
* rozkaz *(ang. `command`)* - dla operacji które modyfikują dane
* zapytanie *(ang. `query`)* - dla operacji które odczytują dane


### Refaktoryzacja do CQRS
Skoro mamy już pogrupowane serwisy wg kryterium odczyt i zapis danych, zrefaktorujmy je na `Commands` oraz `Queries`.

```
public class CreateProduct : Command
{
    public CreateProduct(string id, string name)
    {
      Id = id;  
      Name = name;
    }

    public string Id { get; }
    public string Name { get; }
}

public class CreateProductHandler : CommandHandler<CreateProduct>
{
    IWriteStorageProvider WriteStorageProvider;

    public override void Handle(CreateProduct command)
    {
      // some validation
      // some domain logic
      // save to storage (WriteStorageProvider)
    }
}
```

Model `CreateProduct` stał się komendą, a serwis `ProductCreationService` stał się handlerem dla tej komendy.


[...]

### Kiedy CQRS nie jest najlepszym pomysłem?
CQRS jak wszystko ma swoje wady:
* Wprowadza dodatkową złożoność

Zatem wszędzie tam gdzie mamy systemy tpu `CRUD` pchanie na siłę CQRS nie ma wg mnie sensu.


Autor: `Mateusz Westrych`
Data: `2020.01.22`
