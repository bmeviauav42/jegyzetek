# Kommunikáció

## Előadás

[Kommunikációs lehetőségek](https://edu.vik.bme.hu/mod/resource/view.php?id=42042)

## Cél

A laborfeladatok célja a mikroszolgáltatások fejlesztése során leggyakrabban felmerülő megoldások alapszintű gyakorlása.

## Előkövetelmények

* Docker Desktop
* Visual Studio 2019
    * min v16.3 (gRPC miatt)
    * ASP.NET Core 3.1 SDK (gRPC miatt)
* Kiinduló projekt: <https://github.com/bmeviauav42/komm-kiindulo>
* Postman vagy Fiddler

## REST webszolgáltatások készítése

A labor külön nem tér ki a REST szerű webszolgáltatások készítésének módszereire, arra a szakirányos képzés és a Szoftverfejlesztés .NET platformra című választható tárgy ASP.NET Core anyaga az ajánlott irodalom.

## Hibatűrő kommunikációs módszerek

Az előadáson tárgyalt tervezési minták nem csak a kommunikáció implementációja során hasznos, hanem bármilyen olyan komponens hívása során, ami nem várt tranziens hibajelenséget produkálhat. Tény, hogy leggyakrabban egy távoli hívás kommunikációja során történhet ilyen, így ott mindenképpen érdemes a hibatűrést valamilyen módon megvalósítani.

A laborfeladat során két ASP.NET Core mikroszolgáltatás közötti REST-es kommunikációt szeretnénk hibatűrőbbé tenni. Ehhez a [Polly](https://github.com/App-vNext/Polly) osztálykönyvtárat hívjuk segítségül, ami a leggyakoribb mintákat valósítja meg, nekünk csak felkonfigurálnunk kell. Az egyszerűség kedvéért most a Retry mintát valósítsuk meg.

### Kiinduló projekt áttekintése

Klónozzuk le a kiinduló projektet, és nyissuk meg a solutiont Visual Studio-val.

!!! warning "Ékezetes elérési út"
    Fontos, hogy ne legyen az elérési útban speciális (és ékezetes) karakter, különben nem tud a VS a docker-compose-hoz Debuggerrel csatlakozni.

```cmd
mkdir c:\munka\[neptun]\MSA\komm
cd c:\munka\[neptun]\MSA\komm
git clone https://github.com/bmeviauav42/komm-kiindulo
```

A `master` branchen található a kiinduló, míg a megoldások külön branchre kerültek fel, ha valamelyik részfeladatnál lemaradtál volna.

Mind a két projekt már Dockerizált (Projekten jobb gomb / Add / Docker support), a teljes solutionhöz pedig tartozik egy Docker Compose leíró (Projekteken jobb gomb / Add / Docker Orchestrator support / Docker Compose), ami egyben a futtatandó projekt is.

Két ASP.NET Core projektünk van, nézzük meg jobban őket:

* `Catalog`: REST API, törekedve az egyszerűségre, hogy a labort ne ezzel bonyolítsuk el.
    * `ProductController`
        * `Get()`: `Product` listával tér vissza, az egyszerűség kedvéért az adatok egy statikus listában vannak
        * `Get(int id)`: Egy adott azonosítóval rendelkező terméket ad vissza a listából
        * A listás `Get()` kérés véletlenszerűen hibával (503-as hibakóddal) tér vissza. Ezzel szimuláljuk a szolgáltatás esetleges kimaradását.
* `Order`: szintén egy egyszerű REST-es webszolgáltatás, most csak tesztelés céljából
    * `ApiClients/ICatalogApiClient`
        * [Refit](https://github.com/reactiveui/refit) alapú erősen típusos HTTP kliens szolgáltatás
            * Refit könyvtár azért is hasznos, mert elég csak az interfészt megírni a hívások metaadataival felattributálva, és az implementácót a Refit legenerálja a háttérben.
            * (Persze egy contract-first swagger alapú megközelítést is választhattunk volna, de most ne ezzel bonyolítsuk a labort)
    * `Startup`
        * A Refit klienst beregisztráljuk a DI konténerbe az `AddRefitClient` hívással, ahol megadjuk a távoli szolgáltatás base URL-jét is.
        * Az  `AddRefitClient` hívás az `IHttpClientFactory` mintára épül: modern .NET alkalmazásokban már ez az ajánlott módszer, és nem a `HttpClient` kézi megpéldányosítása. [HttpClientFactory docs](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/http-requests?view=aspnetcore-3.0)
        * Ez azért is jó, mert a Polly könyvtár egyszerűen tud ide integrálódni.
    * `TestController`
        * Csak a tesztelés céljából létrehozott végpontokat tartalmaz, amin keresztül az áthívást tudjuk majd tesztelni
        * DI-ból elkérjük az `ICatalogApiClient` objektumot és azon keresztül áthívunk a távoli szolgáltatásba, majd nagyon egyszerűen annak az eredményével térünk vissza.
* `docker-compose`
    * Figyeljük meg, hogy az `order` konténer függ a `catalog` konténertől (`depends_on`). Ez azért fontos, hogy közöttük felépüljön a hálózati kapcsolat
    * Ha visszatekintünk az `Order` szolgáltatás `Startup` osztályára, akkor láthatjuk, hogy a szolgáltatás nevével hivatkozunk a másik konténerre.
        * Ha ez nem tetszik, akkor a docker-compose fájlban a hostnevet felül is lehet definiálni.
        * Azt is megfigyelhetjük, hogy nem a localhostra kiajánlott portot kell használjuk, hanem egymás felé a tényleges portok vannak nyitva a konténereken. (80, 443).
        * Most HTTP-t használjunk, hogy egyrészt a tanusítványokkal ne kelljen foglalkozzunk, illetve a docker-compose-on belül lévő kommunikáció esetében nem ördögtől való a sima http sem.
        * Ha valamilyen összetettebb orchesztrátort használunk, akkor érdemes nem beégetni ezeket a hostneveket, hanem konfigurációból várni azokat, és egy service discovery szolgáltatást igénybe venni. (most ezt nem nézzük meg)

Próbáljuk futtatni a projekteket! Vizsgáljuk meg az elérhető hívás viselkedését! Azt tapasztaljuk, hogy a Catalog `Get()` kérése (`api/Product`) és így az Order `Get()` kérése is (`api/test`) véletlenszerűen elszáll.

### Polly használata

Vegyük fel az Order projektbe az alábbi NuGet csomagot. Ez függőségként behúzza a [Polly](https://github.com/App-vNext/Polly)-t is, és támogatást ad az `IHttpClientFactory`val történő integrációra.

```xml
<PackageReference Include="Microsoft.Extensions.Http.Polly" Version="3.1.9" />
```

#### Egyszerű Retry

A Startup osztályba adjuk hozzá a Retry policy-t az `IHttpClientFactory`-hoz. Állítsuk be, hogy a kapcsolódási hibák és az általunk tranziens hibáknak vélt státuszkódok esetén próbálkozzon újra 5x.

```C#
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

Ez a Policy gyakorlatilag beépül a `HttpClient` Handler Pipeline-jába, így a hívó számára transzparens lesz az újrapróbálkozási logika. Viszont érdemes odafigyelni a `HttpClient` `Timeout`jára is, mert az újrapróbálkozások során így az nem indul újra.

!!! note "Polly általánosan"
    A Polly-t nem csak `HttpClient`-tel lehet használni, hanem tetszőleges kódban: össze lehet rakni egy Policy láncot, és abba beburkolni a kívánt hívást. A Policy-ket akár karbantarthatjuk a DI konténerben is.

#### Exponenciálisan növekvő Retry időköz

Azonnali Retry helyett várjunk egy kicsit az újrapróbálkozások között, mégpedig exponenciálisan egyre többet.

```C#
    //.RetryAsync(5)
    .WaitAndRetryAsync(5, retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)))
```

Próbáljuk ki!

#### Több Policy használata

Most laboron nem nézünk példát több Policy használatára, de szóba jöhetne még a Timeout, a Circuit breaker, Cache vagy akár a Fallback policy [is](https://github.com/App-vNext/Polly/blob/master/README.md). Az előadás anyagban találtok egy összetettebb szekvencia diagrammot, az ajánlott összetételről.

## Aszinkron kommunikáció RabbitMQ-val

A laboron egy esemény alapú aszinkron kommunikációt valósítunk meg. Az Order szolgáltatás fog publikálni egy `OrderCreated` integrációs eseményt, amit egy üzenetsorba rak. Erre tetszőleges szolgáltatás feliratkozhat és reagálhat rá. Esetünkben a Catalog szolgáltatás fogja a megrendelt termék raktárkészletét csökkenteni. Most az egyszerűség kedvéért ne foglalkozzunk az idempotens megvalósítással.

Az aszinkron kommunikációt most RabbitMQ-n keresztül valósítjuk meg, és a MassTransit osztálykönyvtár segítségével fedjük el, hogy ne kelljen az alacsonyszintű implementációval foglalkoznunk.

### RabbitMQ beüzemelése

A docker-compose konfigurációnkba vegyünk fel egy RabbitMQ image alapú konténert.

```yaml
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

```yaml
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

Ismételjük meg ezt a Catalog szolgáltatásban is.

### MassTransit használata

#### Küldő oldal

Kezdjük a a küldő oldallal. Vegyük fel az Order szolgáltatásba a következő NuGet csomagokat.

```xml
<PackageReference Include="MassTransit.AspNetCore" Version="7.0.4" />
<PackageReference Include="MassTransit.RabbitMQ" Version="7.0.4" />
```

Konfiguráljuk be a `Startup` osztályban a MassTransit-ot, hogy RabbitMQ-t használjon, és hogy melyik üzenetsorba rakja az `IOrderCreatedEvent` eseményünket.

```C#
services.AddMassTransit(x =>
{
    x.UsingRabbitMq((ctx, config) =>
    {
        config.Host(new Uri($"rabbitmq://rabbitmq:/"),
            hostConfig =>
            {
                hostConfig.Username("guest");
                hostConfig.Password("guest");
            });
    });

    EndpointConvention.Map<IOrderCreatedEvent>(
        new Uri("rabbitmq://rabbitmq:/integration"));
});

services.AddMassTransitHostedService();
```

!!! note "Konfiguráció korrektebben"
    Itt is érdemesebb lenne a rabbitmq hosztnevet és a bejelentkezési adatokat konfigurációból nyerni.

Süssük el az eseményt a `TestController`ben. Kérjük el az `IPublishEndpoint` objektumot a konstruktorban, és azon hívjuk meg a `Publish` metódust a `CreateOrder` actionben. MassTransit esetében a `Publish` süti el a broadcast szerű eseményeket, míg a `Send` inkább a command típusú üzenetekre van kihegyezve.

```C#
private readonly ICatalogApiClient _catalogApiClient;
private readonly IPublishEndpoint _publishEndpoint;

public TestController(ICatalogApiClient catalogApiClient, IPublishEndpoint publishEndpoint)
{
    _catalogApiClient = catalogApiClient;
    _publishEndpoint = publishEndpoint;
}

// ...

[HttpPost("[action]")]
public async Task<ActionResult> CreateOrder()
{
    await _publishEndpoint.Publish(new OrderCreatedEvent
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
<PackageReference Include="MassTransit.AspNetCore" Version="7.0.4" />
<PackageReference Include="MassTransit.RabbitMQ" Version="7.0.4" />
```

Szükségünk lesz egy az eseményt lekezelő osztályra is, aminek MassTransit esetben az `IConsumer<T>` interfészt kell megvalósítania.

Vegyünk fel a Catalog projektbe egy `IntegrationEventHandlers` mappát, majd abba hozzunk létre egy új osztályt `OrderCreatedEventHandler` néven az alábbi tartalommal. Itt csak a kapott adatok alapján frissítsük az adatainkat: a mi Móricka példánkban a `ProductController`ben lévő statikus listán dolgozunk.

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

Konfiguráljuk be a `Startup`-ban a MassTransit-ot, hogy RabbitMQ-t használjon, illetve hogy melyik üzenetsorból várja az `IOrderCreatedEvent` eseményünket, és azt melyik `IConsumer` megvalósítás kezelje le.

```C#
services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedEventHandler>();
    x.UsingRabbitMq((ctx, cfg) =>
    {
        cfg.Host(new Uri($"rabbitmq://rabbitmq:/"), hostConfig =>
        {
            hostConfig.Username("guest");
            hostConfig.Password("guest");
        });
        cfg.ReceiveEndpoint("integration", e =>
        {
            e.ConfigureConsumer<OrderCreatedEventHandler>(ctx);
        });
    });
});

services.AddMassTransitHostedService();
```

Az `AddMassTransitHostedService` metódus egy háttérfolyamatot regisztrál be, ami figyeli az üzenetsorokat.

Próbáljuk ki!

* Kérjük le a termékeket
* Süssünk el fiddlerből vagy Postmanből egy POST /api/Test/CreateOrder kérést
* Nézzük meg, hogy frissült-e a termék raktárkészlete

!!! tip "Hibakeresés"
    Ha nem frissült, akkor a logokból vagy a rabbitmq menedzsment felületéről lehet nyomozni.

    * Ha azt tapasztaljuk hogy nem tud csatlakozni valamelyik szolgáltatás, akkor ellenőrizzük a docker-compose file-t és a connection string-eket.
    * Ha azt tapasztaljuk, hogy `skipped` üzenetsorba kerülnek az üzenetek, akkor a küldő oldal rendben működött, de valamiért a fogadó oldal nem tudott a megadott üzenettípusra egyszer sem feliratkozni helyesen.

!!! note "Kitekintés"
    A fenti példában nem törődtünk az idempotens megvalósítással, ez mindig külön tervezést igényel, az üzleti logikánk függvényében, de mindenképpen érdemes a tervezés során figyelni erre.

    Mi most broadcast jellegű integrációs eseményt sütöttünk el. Ne feledjünk van ennek egy másik variánsa is, amikor command szerű üzenetet küldünk egy másik szolgáltatásnak, és ott elvárjuk az esemény lefutását. Integrációs esemény során a fogadó félre van bízva, hogy mit kezd a kapott információval.

## Contract-First API készítés - gRPC

A feladat célja kipróbálni a contract-first megkozelítést: tehát előbb az interfész leírót készítjük el egy Domain Specific Language (DSL) segítésével, majd abból generálunk kliens és szerver oldali kódot.

Ezt REST-es API-val is meg tudnánk tenni a Swagger/OpenAPI leíróval, de mi most gRPC-n keresztül próbáljuk ki. Feladatunkban a Catalog szolgáltatás `ProductController` két műveletét írjuk meg gRPC protokollal. Majd hívjuk meg ezeket az Order szolgáltatásból.


#### Szerver oldal

Lehetőség lenne egy Projektsablonból is dolgoznunk (File / New Project / gRPC Serice), ami szintén egy ASP.NET Core 3.1-ás projekt, de most a meglévő Catalog projektünkbe rakjuk bele ezt a funkcionalitást.

Vegyük fel a Catalog projektbe az alábbi NuGet csomagot.

```xml
<PackageReference Include="Grpc.AspNetCore" Version="2.32.0" />
```

A Catalog projektbe vegyünk fel egy `Protos` mappát, és abba egy Text fájlt `catalog.proto` néven, ami a szolgáltatásleírónk lesz. Készítsünk el egy saját szolgáltatásleírót a Catalog REST API-nk mintájára. Tartalom a következő:

* Egy szolgáltatásunk lesz `CatalogService` néven
* A szolgáltatásban `rpc` kulcsszóval tudunk műveleteket definiálni, megadva a bemenő paramétert és a visszatérési értéket.
    * Ezek külön definiált `message` típusok lehetnek.
* Az első legyen a `GetProducts`
    * Sajnos a proto szabványban nincs `void` típusú üzenet, ezért szükséges felvennünk egy üres üzenetet `Empty` néven, ez fogja majd a `void` típust reprezentálni
    * A visszatérési érték típusát a `ProductsResponse` üzenet írja le, amiben több `Product` típusú üzenet lesz (nevezhetnénk tömbnek is, itt ez a `repeated` kulcsszó)
    * Az üzenetekben meg kell adnunk a mezők sorrendjét az egyenlőség mögött a bináris sorosítás miatt
    * `Product` üzenet az eddig megszokott propertykkel rendelkezzen a proto-s típusokkal (sajnos itt nincs `decimal`)
* A `GetProduct` művelet egy azonosítót vár, és egy `Product`-tal tér vissza.
    * Az `id`-t is szükséges becsomagolni egy üzenettípusba: `GetProductRequest`

```proto
syntax = "proto3";

option csharp_namespace = "Msa.Comm.Lab.Services.Catalog.Grpc";

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

A `.proto` fájlból a modellek és egy ősosztály is generálódik a fordítás során. Ehhez viszont a Catalog projekt fájlban fel kell venni a következő elemet:

```xml
  <ItemGroup>
    <Protobuf Include="Protos\catalog.proto" GrpcServices="Server" />
  </ItemGroup>
```

A Catalog projektbe hozzunk létre egy `Services` mappát, majd abba egy `CatalogService` osztályt, ami a  `Grpc.CatalogService.CatalogServiceBase`-ből származik le. Nekünk csak felül kell definiálni a kívánt metódusokat. Ide is vegyük fel az alábbi statikus listát, és ennek a segítségével valósítsuk meg az üzleti műveleteket. Figyeljük meg, hogy a .proto-ból generált típusokkal tudunk itt dolgozni.

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

A Catalog `Startup` osztályban regisztráljuk be a gRPC szolgáltatásainkat.

```C#
services.AddGrpc();
```

```C#
app.UseEndpoints(endpoints =>
{
    endpoints.MapGrpcService<Services.CatalogService>();
    endpoints.MapControllers();
});
```

#### Kliens oldal

Adjunk az Order projekthez egy szolgáltatás referenciát. Az Order projekten jobb gomb / Add / Service Reference / gRPC / Add / File ahol tallózzuk ki a másik projektben található .proto fájlt és **Client** módban generáljuk le a szükséges osztályokat. Ez a művelet a szükséges NuGet csomagokat is hozzáadja a projekthez.

![image](https://user-images.githubusercontent.com/8333960/66718227-b87af080-ede1-11e9-8b45-48e8e06449fa.png)

A `Startup` osztályban regisztráljuk be a DI konténerbe a gRPC kliensünket. Mivel a gRPC-nek szüksége van HTTPS-re, viszont most a konténereink nem bíznak egymás tanúsítványaiban, ezért most ideiglenesen kapcsoljuk ki a HTTPS hibákat a gRPC-t alatt lévő `HttpClient`-ben.

```C#
services.AddGrpcClient<CatalogService.CatalogServiceClient>(o =>
{
    o.Address = new Uri("https://msa.comm.lab.services.catalog");
}).ConfigurePrimaryHttpMessageHandler(p => new HttpClientHandler
{
    ServerCertificateCustomValidationCallback =
        HttpClientHandler.DangerousAcceptAnyServerCertificateValidator
});
```

!!! note "IHttpClientFactory használata"
    Megfigyelhetjük, hogy a gRPC kliens is az `IHttpClientFactory` megoldásra épít.

A `TestController`ben cseréljük le A REST kliensünket a gRPC kliensre.

```C#
//private readonly ICatalogApiClient _catalogApiClient;
private readonly IPublishEndpoint _publishEndpoint;
private readonly CatalogService.CatalogServiceClient _catalogServiceClient;

public TestController(
    ICatalogApiClient catalogApiClient,
    IPublishEndpoint publishEndpoint,
    CatalogService.CatalogServiceClient catalogServiceClient)
{
    //_catalogApiClient = catalogApiClient;
    _publishEndpoint = publishEndpoint;
    _catalogServiceClient = catalogServiceClient;
}

[HttpGet]
public async Task<ActionResult<IEnumerable<Catalog.Grpc.Product>>> Get()
{
    return (await _catalogServiceClient.GetProductsAsync(new Empty())).Products;
}
```

Próbáljuk ki!

## Összefoglalás

A gyakorlat során néztünk példát REST-es kommunikáció esetében a hibatűrés növelésére egy Retry policy-vel. Majd aszinkron kommunikációt implementáltunk a két szolgáltatásunk között a RabbitMQ üzenetsor és a MassTransit könyvtár segítségével. Végül Contract-first megközelítéssel készítettünk szolgáltatást, most gRPC-n kipróbálva.
