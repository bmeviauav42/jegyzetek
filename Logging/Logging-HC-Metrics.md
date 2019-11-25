# Naplózás, Health checks, Metrikák implementálása

## Naplózás

Az ASP.NET Core TODO webalkalmazásunkban implementáljunk szemantikus naplózást. A naplóbejegyzéseket az úgynevezett ELK technológiai stackkel fogjuk feldolgozni. 
* **E**: Elasticsearch
  * A naplóbejegyzések tárolásáért és indexeléséért/kereshetőség biztosításáért felelős
* **L**: Logstash
  * Transzformációs réteg az alkalmazás és a perzisztencia réteg között
* **K**: Kibana
  * Adatvizualizációért felelős komponens

A labor keretében most a Logstash transzformációs komponenst kihagyjuk a képből idő hiányában, és közvetlenül az Elasticsearch-be fog írni az alkalmazás.

> **Megj.:** Ha az alkalmazásunkat Azure PaaS szolgáltatásokra építjük, akkor az ajánlott megoldás az Azure Application Insights. ELK-t akkor érdemes használni, ha az architektúránkat felhő szolgáltató függetlennek szeretnénk tartani (ami viszonylag ritka).

Az (ASP).NET Core kiváló absztrakciós réteget nyújt nekünk a naplózáshoz az `ILogger<T>` és kapcsolódó interfészein keresztül. Egyszerűbb implementációi a keretrendszerben is megtalálhatóak, de ezek nem elégítik ki a szemantikus naplózáshoz kapcsolódó igényeket: mégpedig, hogy egyszerűen és egységesen parsolhatóak legyenek a naplóbejegyzések.

Használjuk a Serilog külső osztálykönyvtárat a naplózásra, ami a a fenti absztrakcióra épül rá. Seriloghoz több nyelő (Sink) implementáció is készült, és van kifejezetten Elasticsearch-be naplózó csomag is.

### Előkészület

Klónozzuk le a kiinduló projektet, ami ASP.NET Core 3.0-ra lett felmigrálva az előző alkalmakhoz képest.

```cmd
git clone TODO
```

Próbáljuk ki, hogy docker-compose-zal elindul-e az alkalmazásunk, és teszteljük az API GW-en keresztül a működést. **(TODO link)**

### ELK

Vegyünk fel a docker-compose.yml-be két új konténert az Elasticsearch-nek és a Kibana-nak. Fontos, hogy ne az OSS verziót használjuk az ES-ből, mert abban nincsenek alapértelmezetten a Kibana-hoz szükséges komponensek feltelepítve.

```yml
  logs:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.4.2
    container_name: logs
    environment:
      - cluster.name=logs # Settings to start Elasticsearch in a single-node development environment
      - node.name=logs
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms256m -Xmx256m"
    ports:
      - "9202:9200"
    volumes:
      - logs-elastic-data:/usr/share/elasticsearch/data
    networks:
      - todoapp-network

  kibana:
    image: docker.elastic.co/kibana/kibana:7.4.2
    container_name: kibana
    environment: 
      - ELASTICSEARCH_HOSTS=http://logs:9200
    ports:
      - "5602:5601"
    depends_on:
      - logs
    networks:
      - todoapp-network
```

A kötetek közé is fel kell vennünk egy bejegyzést.

```yml
volumes: # The volumes will store the database data; kept even after the containers are deleted
  todoapp-mongo-data:
    driver: local
  todoapp-elastic-data:
    driver: local
  logs-elastic-data:
    driver: local
```

### Serilog

Vegyük fel az alábbi csomagokat a Todos.Api projektbe.

```xml
<PackageReference Include="Serilog" Version="2.9.0" />
<PackageReference Include="Serilog.AspNetCore" Version="3.2.0" />
<PackageReference Include="Serilog.Exceptions" Version="5.3.1" />
<PackageReference Include="Serilog.Settings.Configuration" Version="3.1.0" />
<PackageReference Include="Serilog.Sinks.Elasticsearch" Version="8.0.1" />
```

A Program.cs-ben konfiguráljuk be a Serilog-ot.

```C#
public static IHostBuilder CreateWebHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .UseSerilog((hostingContext, loggerConfiguration) => loggerConfiguration
            .ReadFrom.Configuration(hostingContext.Configuration)
            .Enrich.FromLogContext()
            .Enrich.WithExceptionDetails()
            .WriteTo.Console()
            .WriteTo.Elasticsearch(new ElasticsearchSinkOptions(new Uri(hostingContext.Configuration.GetValue<string>("LogsUrl")))
            {
                AutoRegisterTemplate = true,
            }))
        .ConfigureWebHostDefaults(c =>
        {
            c.UseStartup<Startup>();
        });
```

> **Megj.:** A Serilog ajánlások szerint a Main függvényben lenne érdemes a Serilog-ot inicializálni, hogy az app indulása során fellépő kivételeket is le lehessen logolni. Most ettől az aspektustól eltekintünk az egyszerűség kedvéért.

* A kódrészletből láthatjuk, hogy a Serilog-ot konfigurálhatjuk az `IConfiguration`-ból is, de ezt most nem fogjuk kihasználni, és itt inline adjuk meg az alapértelmezéseket.
* Két Sink-et használunk most
    * A konzolra írunk, mivel ez egy általános elvárás a konténerizált alkalmazások esetében
    * Elasticsearchbe írjuk a logot.
* 2 Enrichert használunk
    * Az enricher-ek olyan általános komponensek, amik a logbejegyzéseket kontextusfüggő információkkal tudják kiegészíteni. pl.: időbélyeg, alkalmazás neve, gép neve, szál azonosítója stb. Most egy általános Enricher-t veszünk fel a `FromLogContext` személyében.
    * Exception-ök részletes adataival egészítjük ki a logot a `WithExceptionDetails`-szel

Fentebb is láthatjuk, hogy az Elasticsearch URL-jét  a konfigurációból nyerjük. Adjuk most ezt meg környezeti változóként a docker-compose.cs.debug.yml állományban.

```yml
      - ASPNETCORE_LogsUrl=http://logs:9200
```

> **Megj.:** A Serilog az appsetting.json és az appsettings.Development.json-ből nem használja fel a Logging szekciót. Ezeket az igényesség kedvéért törölhetjük. Ezek csak a Microsoft-os `ILogger` implementáció számára kellenek.

Logoljunk egy saját eseményt, most a példa kedvéért a TodosRepository-ban. Kérjünk el egy `ILogger<T>`-t

```C#
private readonly ILogger<TodosRepository> _logger;

public TodosRepository(ElasticClient elasticClient, ILogger<TodosRepository> logger)
{
    this.elasticClient = elasticClient;
    _logger = logger;
}
```

Naplózzuk info szinten a todo létrehozását.

```C#
public async Task<TodoItem> Insert(CreateNewTodoRequest value)
{
    // This operation is ***NOT*** idempotent!
    var result = await elasticClient.IndexDocumentAsync(value.ToDal());
    var todo = await FindById(result.Id);
    _logger.LogInformation("Todo with data {@todoitem} for user:{userid} has been created", todo, todo.UserId);
    return todo;
}
```

Figyeljük meg, hogy nem használtuk a C# string interpoláció funkcióját (`$"{userid}"`)! **Ez szándékos szemantikus naplózás esetén!** Ha szemantikus logolást akarunk megvalósítani, akkor ne csak a log bejegyzés szövegében gondolkodjunk, hanem minden kapcsolódó információban. A fenti esetben az úgynevezett log message template természetesen ki lesz értékelve és be lesz helyettesítve a placeholderekre a paraméterek, de így a logger komponensnek lehetősége van ezeket a paramétereket nem csak a log üzenetbe belerakni, hanem a mi esetünkben az Elasticsearch-ben kereshetően eltárolni.

A template-ek esetében csak nem egyszerű `ToString()` hívást lehet végezni, hanem a fenti példában a `{@todoitem}` placeholder esetében a `@` jelentése az objektum sorosítására vonatkozik. lásd: https://github.com/serilog/serilog/wiki/Structured-Data

Oda kell figyelni, hogy a template-ben lévő propertyknek a JSON típusa (int, string, obj, array stb) első beszúráskor fixálva lesznek az ES sémájában. Ha refaktoráljuk a template-et és mást próbálunk logolni, akkor szimplán nem fog beszúródni az ES-be.

### Kibana

Futtassuk az alkalmazásunkat és generáljunk egy bejegyzést a fenti funkcióval.

Nyissuk meg a Kibana-t a TODO URL-en.

A log nézetben még nem látunk semmit. A kibanának mondjuk meg, hogy milyen index-en keresse a bejegyzéseket. Esetünkben ez alapértelmezetten a `logstash-*` mintára fog illeszkedni. Második lépésként válasszuk ki a @timestamp mezőt a szűréshez.

Vizsgáljuk meg a logbejegyzéseket, kitüntetetten a saját TODO létrehozásával kapcsolatos bejegyzést. Figyeljük meg, hogy szinte minden mező kereshető és szépen, struktúráltan kerülnek a bejegyzés adatai lementésre.

