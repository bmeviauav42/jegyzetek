# Mikroszolgáltatások és konténer alapú szoftverfejlesztés

# Kommunikációs lehetőségek - gyakorlat

A laborfeladatok célja a mikroszolgáltatások fejlesztése során leggyakrabban felmerülő megoldások alapszintű gyakorlása.

## Előkövetelmények
* Előadás anyaga: https://www.aut.bme.hu/Upload/Course/VIAUAV42/hallgatoi_jegyzetek/Kommunik%c3%a1ci%c3%b3s%20lehet%c5%91s%c3%a9gek.pdf
* Docker Desktop
* Visual Studio 2019 
    * min v16.3 (gRPC miatt)
    * ASP.NET Core 3.0 SDK (gRPC miatt)
* Kiinduló projekt: https://github.com/bmeviauav42/komm-kiindulo
* Postman vagy Fiddler

## REST webszolgáltatások készítése

A labor külön nem tér ki a REST szerű webszolgáltatások készítésének módszereire, arra a szakirányos képzés és a Szoftverfejlesztés .NET platformra című választható tárgy ASP.NET Core anyaga az ajánlott irodalom.

## Hibatűrő kommunikációs módszerek

Az előadáson tárgyalt tervezési minták nem csak a kommunikáció implementációja során hasznos, hanem bármilyen olyan komponens hívása során, ami nem várt tranziens hibajelenséget produkálhat. Tény, hogy leggyakrabban egy távoli hívás kommunikációja során történhet ilyen, így ott mindenképpen érdemes a hibatűrést valamilyen módon megvalósítani.

A laborfeladat során két ASP.NET Core mikroszolgáltatás közötti REST-es kommunikációt szeretnénk hibatűrőbbé tenni. Ehhez a Polly osztálykönyvtárat hívjuk segítségül, ami a leggyakoribb mintákat valósítja meg, nekünk csak felkonfigurálnunk kell. Az egyszerűség kedvéért most a Retry mintát valósítsuk meg.

### Kiinduló projekt áttekintése

Klónozzuk le a kiinduló projektet, és nyissuk meg a solutiont Visual Studio-val.

``` cmd
mkdir c:\munka\[neptun]\MSA\komm
cd c:\munka\[neptun]\MSA\komm
git clone https://github.com/bmeviauav42/komm-kiindulo
```

Mind a két projekt már Dockerizált (Projekten jobb gomb / Add / Docker support), a teljes solutionhöz pedig tartozik egy Docker Compose leíró (Projekteken jobb gomb / Add / Docker Orchestrator support / Docker Compose), amit egyben a futtatandó projekt is.

Két ASP.NET Core projektünk van, nézzük meg jobban őket:
* `Catalog`: REST API, törekedve az egyszerűségre, hogy a labort ne ezzel bonyolítsuk el.
    * `ProductController`
        * `Get()`: `Product` listával tér vissza, az egyszerűség kedvéért az adatok egy statikus listában vannak
        * `Get(int id)`: Egy adott azonosítóval rendelkező terméket ad vissza a listából
        * A listás `Get()` kérés véletlenszerűen hibával (503-as hibakóddal) tér vissza. Ezzel szimuláljuk a szolgáltatás esetleges kimaradását.
* `Order`: szintén egy egyszerű REST-es webszolgáltatás, most csak tesztelés céljából
    * `ApiClients/ICatalogApiClient`
        * [Refit](https://github.com/reactiveui/refit) alapú erősen típusos HTTP kliens szolgáltatás
            * Refit könyvtár azért is jó, mert elég csak az interfészt megírni a hívások metaadataival felattributálva, és az implementácót a Refit legenerálja a háttérben.
            * (Persze egy contract-first megközelítést is választhattunk volna, de most ne ezzel bonyolítsuk a labort)
    * `Startup`
        * A Refit klienst beregisztráljuk a DI konténerbe az `AddRefitClient` hívással, ahol megadjuk a távoli szolgáltatás base URL-jét is. 
        * Az  `AddRefitClient` hívás az `IHttpClientFactory` mintára épül: modern .NET alkalmazásokban már ez az ajánlott módszer, és nem a `HttpClient` kézi megpéldányosítása. https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-3.0
        * Ez azért is jó, mert a [Polly](https://github.com/App-vNext/Polly) könyvtár egyszerűen tud ide integrálódni.
    * `TestController`
        * Csak a tesztelés céljából létrehozott végpontokat tartalmaz, amin keresztül az áthívást tudjuk majd tesztelni
        * DI-ból elkérjük az `ICatalogApiClient` objektumot és azon keresztül áthívunk a távoli szolgáltatásba, majd nagyon egyszerűen annak az eredményével térünk vissza.
* `docker-compose`
    * Figyeljük meg, hogy az `order` konténer függ a `catalog` konténertől (`depends_on`). Ez azért fontos, hogy közöttük felépüljön a hálózati kapcsolat
    * Ha visszatekintünk az `Order` szolgáltatás `Startup` osztályára, akkor láthatjuk, hogy a szolgáltatás nevével hivatkozunk a másik konténerre. 
        * Ha ez nem tetszik, akkor a docker-compose fájlban a hostnevet felül is lehet definiálni.
        * Azt is megfigyelhetjük, hogy nem a localhostra kiajánlott portot kell használjuk, hanem egymás felé a tényleges portok vannak nyitva a konténereken. (80, 443).
        * Most HTTP-t használjunk hogy a tanusítványokkal ne kelljen foglalkozzunk.
        * Ha valamilyen összetettebb orchesztrátort használunk, akkor érdemes nem beégetni ezeket a hostneveket, hanem konfigurációból várni azokat. (most ezt nem nézzük meg)
    
Próbáljuk futtatni a projekteket! Vizsgáljuk meg az elérhető hívás viselkedését! Azt tapasztaljuk, hogy a Catalog `Get()` kérése (`api/Product`) és így az Order `Get()` kérése is (`api/test`) véletlenszerűen elszáll. 

### Polly használata

Vegyük fel az Order projektbe az alábbi NuGet csomagot. Ez függőségként behúzza a [Polly](https://github.com/App-vNext/Polly)-t is, és támogatást ad az `IHttpClientFactory`val történő integrációra.

``` xml
<PackageReference Include="Microsoft.Extensions.Http.Polly" Version="2.2.0" />
```

#### Egyszerű Retry

A Startup osztályba adjuk hozzá a Retry policy-t az `IHttpClientFactory`-hoz. Állítsuk be, hogy a kapcsolódási hibák és az általunk tranziens hibáknak vélt státuszkódok esetén próbálkozzon újra 5x.

``` C#
bool RetryableStatusCodesPredicate(HttpStatusCode statusCode) =>
    statusCode == HttpStatusCode.BadGateway
        || statusCode == HttpStatusCode.ServiceUnavailable
        || statusCode == HttpStatusCode.GatewayTimeout;

services.AddRefitClient<ICatalogApiClient>()
    .ConfigureHttpClient(c => c.BaseAddress = new Uri("http://msa.comm.lab.services.catalog"))
    .AddPolicyHandler(Policy
        .Handle<HttpRequestException>()
        .OrResult<HttpResponseMessage>(msg => RetryableStatusCodesPredicate(msg.StatusCode))
        .RetryAsync(5)
    );
```

Próbáljuk ki! Tapasztalatunk szerint szinte megszűntek a hibák.

Ez a Policy gyakorlatilag beépül a `HttpClient` Handler Pipeline-jába, így a hívó számára transzparens lesz az újrapróbálkozási logika. Viszont éredemes odafigyelni a `HttpClient` `Timeout`jára is, mert az újrapróbálkozások során így az nem indul újra.

> **Megj.:** A Polly-t nem csak `HttpClient`-tel lehet használni, hanem tetszőleges kódban: össze lehet rakni egy Policy láncot, és abba beburkolni a kívánt hívást. A Policy-ket akár karbantarthatjuk a DI konténerben is.

#### Exponenciálisan növekvő Retry időköz

Azonnali Retry helyett várjunk egy kicsit az újrapróbálkozások között, mégpedig exponenciálisan egyre többet.

``` C#
    //.RetryAsync(5)
    .WaitAndRetryAsync(5, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)))
```

Próbáljuk ki!

#### Több Policy használata

Most laboron nem nézünk példát több Policy használatára, de szóbajöhetne még a Timeout, a Circuit breaker, Cache vagy akár a Fallback policy [is](https://github.com/App-vNext/Polly/blob/master/README.md). Az előadás anyagban találtok egy összetettebb szekvencia diagrammot, az ajánlott összetételről.

## Aszinkron kommunikáció RabbitMQ-val

A laboron egy esemény alapú aszinkron kommunikációt valósítunk meg. Az Order szolgáltatás fog publikálni egy `OrderCreated` integrációs eseményt, amit egy üzenetsorba rak. Erre tetszőleges szolgáltatás feliratkozhat és reagálhat rá. Esetünkben a Catalog szolgáltatás fogja a megrendelt termék raktárkészletét csökkenteni. Most az egyszerűség kedvéért ne foglalkozzunk az idempotens megvalósítással.

Az aszinkron kommunikációt most RabbitMQ-n keresztül valósítjuk meg, és a MassTransit osztálykönyvtár segítségével fedjük el, hogy ne kelljen az akacsonyszintű implementációval foglalkoznunk.

### RabbitMQ beüzemelése

A docker-compose konfigurációnkba vegyünk fel egy RabbitMQ image alapú konténert.

``` yaml
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    container_name: rabbitmq
    hostname: rabbitmq
```

A RabbitMQ funkcióit a 5672 porton érjük el, míg a 15672-n az admin felületet nézhetjük meg.  
Alapértelmezett felhasználónév: **guest**  jelszó: **guest**

Adjuk meg, a másik két service esetében a rabbitmq-tól való függést.

``` yaml
  msa.comm.lab.services.catalog:
    # ...
    depends_on:
      - rabbitmq

  msa.comm.lab.services.order:
    # ...
    depends_on:
      - msa.comm.lab.services.catalog
      - rabbitmq
```

### Integrációs esemény

Hozzunk létre egy új .NET Standard projektet `Msa.Comm.Lab.Events` néven, ami lényegében a kommunikáció contractja lesz. Ebbe vegyünk fel egy interfészt `IOrderCreatedEvent` néven az alábbi tartalommal. Ez fog majd utazni az üzenetsorban.

```C#
public interface IOrderCreatedEvent
{
    int ProductId { get; }
    DateTimeOffset OrderPlaced { get; }
    int Quantity { get; }
}
```

Az Order projektben adjunk referenciát az előzőleg létrehozott projektre.

Hozzunk létre egy `IntegrationEvents` mappát az Order projektben, majd abba implementáljuk egy osztályba az `IOrderCreatedEvent` interfészt.

```C#
public class OrderCreatedEvent : IOrderCreatedEvent
{
    public int ProductId { get; set; }
    public DateTimeOffset OrderPlaced { get; set; }
    public int Quantity { get; set; }
}
```

Ismételük meg ezt a Catalog szolgáltatásban is.

### MassTransit használata

#### Küldő oldal

Kezdjük a a küldő oldallal. Vegyük fel az Order szolgáltatásba a következő NuGet csomagokat.

```xml
<PackageReference Include="MassTransit.Extensions.DependencyInjection" Version="5.5.5" />
<PackageReference Include="MassTransit.Extensions.Logging" Version="5.5.5" />
<PackageReference Include="MassTransit.RabbitMQ" Version="5.5.5" />
```

Konfiguráljuk be a `Startup` osztályban a MassTransit-ot, hogy RabbitMQ-t használjon, adjuk meg a logolás módját, és hogy melyik üzenetsorba rakja az `IOrderCreatedEvent` eseményünket.

```C#
services.AddMassTransit(x =>
{
    x.AddBus(provider =>
        Bus.Factory.CreateUsingRabbitMq(cfg =>
        {
            cfg.Host(new Uri($"rabbitmq://rabbitmq:/"),
                hostConfig =>
                {
                    hostConfig.Username("guest");
                    hostConfig.Password("guest");
                });
            cfg.UseExtensionsLogging(provider.GetRequiredService<ILoggerFactory>());
        }));

    EndpointConvention.Map<IOrderCreatedEvent>(new Uri("rabbitmq://rabbitmq:/integration"));
});
```

> **Megj.:** Itt is érdemesebb lenne a rabbitmq hosztnevet és a bejelentkezési adatokat konfigurációból nyerni.

Süssük el az eseményt a `TestController`ben. Kérjük el az `IBusControl` objektumot, és azon hívjuk meg a `Publish` metódust. MassTransit esetében a `Publish` süti el a broadcast szerű eseményeket, míg a `Send` inkább a command típusú üzenetekre van kihegyezve.

```C#
private readonly ICatalogApiClient _catalogApiClient;
private readonly IBusControl _bus;

public TestController(ICatalogApiClient catalogApiClient, IBusControl bus)
{
    _catalogApiClient = catalogApiClient;
    _bus = bus;
}

// ...

[HttpPost("[action]")]
public async Task<ActionResult> CreateOrder()
{
    await _bus.Publish(new OrderCreatedEvent 
    { 
        ProductId = 1,
        Quantity = 1,
        OrderPlaced = DateTimeOffset.UtcNow
    });

    return Ok(new { Message = "Megrendelés sikeres!" });
}
```

#### Fogadó oldal

Térjünk át a fogadó oldalra. A Catalog szolgáltatás projektbe vegyük fel szintén az alábbi NuGet csomagokat.

```xml
<PackageReference Include="MassTransit.Extensions.DependencyInjection" Version="5.5.5" />
<PackageReference Include="MassTransit.Extensions.Logging" Version="5.5.5" />
<PackageReference Include="MassTransit.RabbitMQ" Version="5.5.5" />
```

Szükségünk lesz egy az eseményt lekezelő osztályra is, aminek MassTransit esetben az `IConsumer<T>` interfészt kell megvalósítania.

 Vegyünk fel a Catalog projektbe egy `IntegrationEventHandlers` mappát, majd abba hozzunk létre egy új osztályt `OrderCreatedEventHandler` néven az alábbi tartommal. Itt csak a kapott adatok alapján frissítsük az adatainkat: a mi Móricka példánkban a `ProductController`ben lévő statikus listán dolgozunk.

```C#
public class OrderCreatedEventHandler : IConsumer<IOrderCreatedEvent>
{
    public Task Consume(ConsumeContext<IOrderCreatedEvent> context)
    {
        var product = ProductController.Products
            .SingleOrDefault(p => p.ProductId == context.Message.ProductId);
        if (product != null)
        {
            product.Stock -= context.Message.Quantity;
        }

        return Task.CompletedTask;
    }
}
```

Konfiguráljuk be a `Startup`-ban a MassTransit-ot, hogy RabbitMQ-t használjon, adjuk meg a logolás módját, illetve hogy melyik üzenetsorból várja az `IOrderCreatedEvent` eseményünket, és azt melyik `IConsumer` megvalósítás kezelje le. 

```C#
services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedEventHandler>();
    x.AddBus(provider => Bus.Factory.CreateUsingRabbitMq(cfg =>
    {
        var host = cfg.Host(new Uri($"rabbitmq://rabbitmq:"), hostConfig =>
        {
            hostConfig.Username("guest");
            hostConfig.Password("guest");
        });
        cfg.UseExtensionsLogging(provider.GetRequiredService<ILoggerFactory>());
        cfg.ReceiveEndpoint(host, "integration", ep =>
        {
            ep.ConfigureConsumer<OrderCreatedEventHandler>(provider);
        });
    }));
});
```

Fogadó oldalon még szükséges elindítani egy háttérfolyamatot is, ami figyeli az üzenetsorokat. Ezt most tegyük meg a `Startup.Configure()` metódus végén.

```C#
public void Configure(
    IApplicationBuilder app,
    IHostingEnvironment env,
    IApplicationLifetime lifetime)
{
    // ...
    app.UseMvc();

    var bus = app.ApplicationServices.GetService<IBusControl>();
    var busHandle = TaskUtil.Await(() => bus.StartAsync());
    lifetime.ApplicationStopping.Register(() => busHandle.Stop());
}
```

Próbáljuk ki!
* Kérjük le a termékeket
* Süssük el fiddlerből vagy Postmanből a `CreateOrder` Actiont
* Nézzük meg, hogy frissült-e a termék raktárkészlete

> **Tipp:** Ha nem frissült, akkor a logokból vagy a rabbitmq menedzsment felületéről lehet nyomozni. Ha azt tapasztaljuk hogy nem tud csatlakozni valamilyik szolgáltatás, akkor ellenőrízzük a docker-compose file-t és a connection string-eket. Ha azt tapasztaljuk, hogy `skipped` üzenetsorba kerülnek az üzenetek, akkor a küldő oldal rendben működött, de valamiért a fogadó oldal nem tudott a megadott üzenettípusra egyszer sem feliratkozni helyesen.

> **Megj.:** A fenti példában nem törődtünk az idempotens megvalósítással, ez mindig külön tervezést igényel, az üzleti logikánk függvényében, de mindenképpen érdemes a tervezés során figyelni erre.

> **Megj.:** Mi most broadcast jellegű integrációs eseményt sütöttünk el. Ne feledjünk van ennek egy másik variánsa is, amikor command szerű üzenetet küldünk egy másik szolgáltatásnak, és ott elvárjuk az esemény lefutását. Integrációs esemény során a fogadó félre van bízva, hogy mit kezd a kapott információval.

## Contract-First API készítés - gRPC

A feladat célja kipróbálni a contract-first megkozelítést: tehát előbb az interfész leírót készítjük el egy Domain Specific Language (DSL) segítésével, majd abból generálunk kliens és szerver oldali kódot. 

Ezt REST-es API-val is meg tudnánk tenni a Swagger/OpenAPI leíróval, de mi most gRPC-n keresztül próváljuk ki. Feladatunkban a Catalog szolgáltatás `ProductController` két műveletét írjuk meg gRPC protokollal. Majd hívjuk meg ezeket az Order szolgáltatásból.

> **Megj.:** A gRPC csak ,NET Core 3.0-tól támogatott, így érdemes a Visual Studio-t frissíteni legalább 16.3.x verzióra

#### Szerver oldal

Készítsünk a solutionbe egy új **gRPC Service** projektet `Msa.Comm.Lab.Services.Catalog2` néven. Docker támogatás most nem kell, mert a labor idejébe már nem férne bele a docker/kestrel HTTPS támogatásának a konfigurációja, viszont a gRPC számára ez egy kötelező elem.

Tekintsük át a generált projektet:
* Ez is egy ASP.NET Core 3.0 projekt, amibe most csak a gRPC szolgáltatások vannak felkonfigurálva
* gRPC-hez a `Grpc.AspNetCore` NuGet csomag szükséges szerver oldalon.
* Kestrel webszerverben alapértelmezettként van beállítva a HTTP/2
* A `Protos` mappában van a szolgáltatás leíró `.proto` állomány
* A `.proto` fájlból a modellek és egy ősosztály is generálódik a fordítás során, amiből a `Services` mappában lévő osztály származik le. Nekünk csak felül kell definiálni a kívánt metódusokat.

Készütsünk el egy saját szolgáltatásleírót a Catalog REST API-nk mintájára. A `Protos` mappában lévő fájlt nevezzük át `catalog.proto`-ra.

Tartalom a következő:
* Egy szolgáltatásunk lesz `CatalogService` néven
* A szolgáltatásban `rpc` kulcsszóval tudunk műveleteket definiálni, megadva a bemenő paramétert és a visszatérési értéket.
    * Ezek külön definiált `message` típusok lehetnek.
* Az első legyen a `GetProducts`
    * Sajnos a proto szabványban nincs `void` típusú üzenet, ezért szükséges felvennünk egy üres üzenetet `Empty` néven, ez fogja majd a `void` típust reprezentálni
    * A visszatérési érték típusát a `ProductsResponse` üzenet írja le, amiben több `Product` típusú üzenet lesz (nevezhetnénk tömbnek is, itt `repeated` kulcsszó)
    * Az üzenetekben meg kell adnunk a mezők sorrendjét az egyenlőség mögött a bináris sorosítás miatt
    * `Product` üzenet az eddig megszokott propertykkel rendelkezzen a proto-s típusokkal (sajnos itt nincs `decimal`)
* A `GetProduct` művelet egy azonosítót vár, és egy `Product`-tal tér vissza.
    * Az id-t is szükséges becsomagolni egy üzenettípusba: `GetProductRequest`

```proto
syntax = "proto3";

option csharp_namespace = "Msa.Comm.Lab.Services.Catalog2.Grpc";

package Catalog;

service CatalogService {
  rpc GetProducts (Empty) returns (ProductsResponse);
  rpc GetProduct (GetProductRequest) returns (Product);
}

message Empty {
}

message ProductsResponse {
	repeated Product Products = 1;
}

message Product {
	int32 ProductId = 1;
	string Name = 2;
	double UnitPrice = 3;
	int32 Stock = 4;
}

message GetProductRequest {
	int32 ProductId = 1;
} 
```

Fordítsuk le a projektet. Utána a `GreeterService`-t nevezzük át `CatalogService`-re (CTRL + R, CTRL + R a típuson állva), és írjuk át az ősosztályát a generált `Grpc.CatalogService.CatalogServiceBase`-re. Ide is vegyük fel az alábbi statikus listát, és ennek a segítségével valósítsuk meg az üzleti műveleteket. Figyeljük meg, hogy a .proto-ból generált típusokkal tudunk itt dolgozni.

```C#
public class CatalogService : Grpc.CatalogService.CatalogServiceBase
{
    private readonly ILogger<CatalogService> _logger;

    internal static List<Product> _products = new List<Product>
    {
        new Product { ProductId = 1, Name = "Sör", Stock = 10, UnitPrice = 250 },
        new Product { ProductId = 2, Name = "Bor", Stock = 5, UnitPrice = 890 },
        new Product { ProductId = 3, Name = "Csoki", Stock = 15, UnitPrice = 200 },
    };

    public CatalogService(ILogger<CatalogService> logger)
    {
        _logger = logger;
    }

    public override Task<ProductsResponse> GetProducts(Empty request, ServerCallContext context)
    {
        var response = new ProductsResponse();
        response.Products.Add(_products);
        return Task.FromResult(response);
    }

    public override Task<Product> GetProduct(GetProductRequest request, ServerCallContext context)
    {
        var product = _products.SingleOrDefault(p => p.ProductId == request.ProductId);
        return Task.FromResult(product);
    }
}
```

#### Kliens oldal

Mivel a gRPC csak .NET Core 3.0-val működik, így szükségünk lesz az Order projekt felfrissítésére. 
* A projekt fájlban írjuk át a target framework-öt `netcoreapp3.0`-ra
* Töröljük a `Microsoft.AspNetCore.App` csomagot
* A `Startup`-ban pedig most ideiglenesen egészítsük ki a következővel az MVC konfigurációját `services.AddMvc(o => o.EnableEndpointRouting = false)`

Ezütán Order projekten jobb gomb / Add / Service Reference / gRPC / Add / File ahol tallózuk ki a másik projektben található .proto fájlt és **Client** módban generáljuk le a szükséges osztályokat. Ez a művelet a szükséges NuGet csomagokat is hozzáadja a projekthez.

![image](https://user-images.githubusercontent.com/8333960/66718227-b87af080-ede1-11e9-8b45-48e8e06449fa.png)

A `Startup` osztályban regisztráljuk be a DI konténerbe a gRPC kliensünket.

```C#
services.AddGrpcClient<CatalogService.CatalogServiceClient>(o =>
{
    o.Address = new Uri("https://localhost:5001");
});
```

A `TestController`ben pedig cseéljük le A REST kliensünket a gRPC kliensre.

```C#
private readonly CatalogService.CatalogServiceClient _catalogServiceClient;

public TestController(CatalogService.CatalogServiceClient catalogServiceClient)
{
    _catalogServiceClient = catalogServiceClient;
}

[HttpGet]
public async Task<ActionResult<IEnumerable<Product>>> Get()
{
    return (await _catalogServiceClient.GetProductsAsync(new Empty())).Products;
}
```

Próbáljuk ki! 

Sajnos most nincs időnk a dockert felkonfigurálni a HTTPS-re, így, állítsuk át a következő módon a kiinduló projekteket:
* Order legyen a Startup projekt, majd állítsuk át IIS Express futtatási módra a Play gomb legördülőjében
* Majd a Solution-ben adjunk meg több startup projektet, mégpedig a **Catalog2**-t és az **Order**t.

## Összefoglalás

A gyakorlat során néztünk példát REST-es kommunikáció esetében a hibatűrés növelésére egy Retry policy-vel. Majd Aszinkron kommunikációt implementáltunk egy RabbitMQ üzenetsor és MassTransit konyvtár segítségével a két szolgáltatásunk között. Végül Contract first megközelítéssel készítettünk szolgáltatást, most gRPC-n kipróbálva.
