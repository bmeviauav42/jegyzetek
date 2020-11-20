# Naplózás, health check

## Előadás

[Naplózás, Health Checks](https://edu.vik.bme.hu/mod/resource/view.php?id=22408)

??? "Előző alkalmak cheatsheet"

    * Docker
        * `docker rm -f $(docker ps -aq)`
    * K8S dashboard
        * `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/alternative/kubernetes-dashboard.yaml`
        * `kubectl proxy`
        * http://localhost:8001/api/v1/namespaces/kube-system/services/kubernetes-dashboard:/proxy
    * Traefic
        * `helm version`
        * `helm init`
        * `helm install stable/traefik --name traefik --version "1.78.3" --set rbac.enabled=true --set logLevel=debug --set dashboard.enabled=true --set service.nodePorts.http=30080 --set serviceType=NodePort`
    * DB
        * `kubectl apply -f db`
    * App
        * `kubectl apply -f app`
        * `$env:IMAGE_TAG="v1"` vagy írjuk át a yml-ben latest-re ideiglenesen
        * `docker-compose build`
        * http://localhost:30080

## Naplózás

Az ASP.NET Core TODO webalkalmazásunkban implementáljunk strukturált naplózást. A naplóbejegyzéseket az úgynevezett [ELK](https://www.elastic.co/what-is/elk-stack) technológiai stackkel fogjuk feldolgozni.

* **E**: Elasticsearch
    * A naplóbejegyzések tárolásáért és indexeléséért/kereshetőség biztosításáért felelős
* **L**: Logstash
    * Transzformációs réteg az alkalmazás és a perzisztencia réteg között
* **K**: Kibana
    * Adatvizualizációért felelős komponens

A labor keretében most a Logstash transzformációs komponenst kihagyjuk a képből idő hiányában, és közvetlenül az Elasticsearch-be fog írni az alkalmazás.

!!! note "Azure Application Insights"
    Ha az alkalmazásunkat Azure PaaS szolgáltatásokra építjük, akkor az ajánlott megoldás az [_Azure Application Insights_](https://docs.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview) és [_Azure Monitor_](https://docs.microsoft.com/en-us/azure/azure-monitor/overview). ELK-t akkor érdemes használni, ha az architektúránkat felhő szolgáltató függetlennek szeretnénk tartani.

Az (ASP).NET Core kiváló absztrakciós réteget nyújt nekünk a naplózáshoz az `ILogger<T>` és kapcsolódó interfészein keresztül. Egyszerűbb implementációi a keretrendszerben is megtalálhatóak, de ezek nem elégítik ki a strukturált naplózáshoz kapcsolódó igényeket: mégpedig, hogy egyszerűen és egységesen parsolhatóak legyenek a naplóbejegyzések.

Használjuk a [Serilog](https://github.com/serilog/serilog) külső osztálykönyvtárat a naplózásra, ami a a fenti absztrakcióra épül rá. Seriloghoz több nyelő (Sink) implementáció is készült, és van kifejezetten Elasticsearch-be naplózó csomag is.

### Előkészület

Klónozzuk le a kiinduló projektet.

```cmd
git clone https://github.com/bmeviauav42/todoapp-logging-hc.git
```

Próbáljuk ki, hogy docker-compose-zal elindul-e az alkalmazásunk, és teszteljük az API GW-en keresztül a működést.

### ELK

Vegyünk fel a docker-compose.yml-be két új konténert az Elasticsearch-nek és a Kibana-nak.

```yaml
  logs:
    image: docker.elastic.co/elasticsearch/elasticsearch-oss:7.8.0
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
    image: docker.elastic.co/kibana/kibana-oss:7.8.0
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

```yaml
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
<PackageReference Include="Serilog.AspNetCore" Version="3.4.0" />
<PackageReference Include="Serilog.Exceptions" Version="6.0.0" />
<PackageReference Include="Serilog.Settings.Configuration" Version="3.1.0" />
<PackageReference Include="Serilog.Sinks.Elasticsearch" Version="8.4.1" />
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

!!! note "Hibakezelés az indulás közben"
    A Serilog ajánlások szerint a Main függvényben lenne érdemes a Serilog-ot inicializálni, hogy az app indulása során fellépő kivételeket is le lehessen logolni. Most ettől az aspektustól eltekintünk az egyszerűség kedvéért.

* A kódrészletből láthatjuk, hogy a Serilog-ot konfigurálhatjuk az `IConfiguration`-ból is, de ezt most nem fogjuk kihasználni, és itt inline adjuk meg az alapértelmezéseket.
* Két Sink-et használunk most
    * A konzolra írunk, mivel ez egy általános elvárás a konténerizált alkalmazások esetében
    * Elasticsearchbe írjuk a logot.
* 2 Enrichert használunk
    * Az enricher-ek olyan általános komponensek, amik a logbejegyzéseket kontextusfüggő információkkal tudják kiegészíteni. pl.: időbélyeg, alkalmazás neve, gép neve, szál azonosítója stb. Most egy általános Enricher-t veszünk fel a `FromLogContext` személyében.
    * Exception-ök részletes adataival egészítjük ki a logot a `WithExceptionDetails`-szel

Fentebb is láthatjuk, hogy az Elasticsearch URL-jét  a konfigurációból nyerjük. Adjuk most ezt meg környezeti változóként a docker-compose.cs.debug.yml állományban.

```yaml
      - ASPNETCORE_LogsUrl=http://logs:9200
```

!!! note ""
    A Serilog az appsetting.json és az appsettings.Development.json-ből nem használja fel a Logging szekciót. Ezeket az igényesség kedvéért törölhetjük. Ezek csak a Microsoft-os `ILogger` implementáció számára kellenek.

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

Figyeljük meg, hogy nem használtuk a C# string interpoláció funkcióját (`$"{userid}"`)! **Ez szándékos strukturált naplózás esetén!** Ha strukturált logolást akarunk megvalósítani, akkor ne csak a log bejegyzés szövegében gondolkodjunk, hanem minden kapcsolódó információban. A fenti esetben az úgynevezett log message template természetesen ki lesz értékelve, és be lesz helyettesítve a placeholderekre a paraméterek, de így a logger komponensnek lehetősége van ezeket a paramétereket nem csak a log üzenetbe belerakni, hanem a mi esetünkben az Elasticsearch-ben kereshetően eltárolni.

A template-ek esetében csak nem egyszerű `ToString()` hívást lehet végezni, hanem a fenti példában a `{@todoitem}` placeholder esetében a `@` jelentése az objektum sorosítására vonatkozik. lásd: https://github.com/serilog/serilog/wiki/Structured-Data

Oda kell figyelni, hogy a template-ben lévő propertyknek a JSON típusa (`int, string, obj, array` stb) első beszúráskor lesznek megkötve az ES sémájában (`AutoRegisterTemplate = true`). Ha refaktoráljuk a template-et és mást próbálunk logolni, akkor szimplán nem fog beszúródni az ES-be.

### Kibana

Futtassuk az alkalmazásunkat és generáljunk egy bejegyzést a fenti funkcióval.

Nyissuk meg a Kibana-t.

A Discover nézetben még nem látunk semmit. A kibanának mondjuk meg, hogy milyen index-en keresse a bejegyzéseket. Esetünkben ez alapértelmezetten a `logstash-*` mintára fog illeszkedni. Második lépésként válasszuk ki a `@timestamp` mezőt a szűréshez.

Vizsgáljuk meg a logbejegyzéseket, kitüntetetten a saját TODO létrehozásával kapcsolatos bejegyzést. Figyeljük meg, hogy szinte minden mező kereshető és szépen strukturáltan kerülnek a bejegyzés adatai lementésre. Próbáljunk meg az általunk készített bejegyzés message template-jére keresni.

## Health Checks

Implementáljunk a Todos.Api projektünkhöz health check-et, amit majd a kubernetes fog elsősorban felhasználni.

### Előkészület

Mivel Kubernetes-hez még nem konfiguráltuk fel a loggolás szolgáltatásait, ezért a jelenlegi munkánkat commitoljuk egy külön ágra, majd álljunk vissza a kiinduló ágra.

```bash
git branch logging
git commit -m "naplózás kész"
git checkout kiindulóTODO
```

Előző órák mintájára üzemeljük be a kubernetes verzióját az alkalmazásnak. Nézzük meg a dashboardon a rendszer állapotát és próbáljuk ki az alkalmazást.

### Health Check implementáció

Készítsünk readiness és liveness próbákat a kubernetes számára. Ehhez használjuk fel az ASP.NET Core 2.2 óta rendelkezésre álló beépített Health Check API-kat.

#### Liveness

Kezdjük az egyszerűbbel. Akkor lesz live egy szolgáltatás, ha az app felált és ki tudja szolgálni külső függőségek nélkül a liveness próbát. Ehhez vegyünk fel egy üres health checket a `/health/live` végpontra.

```C#
public void ConfigureServices(IServiceCollection services)
{
    // ...
    services.AddHealthChecks()
        .AddCheck("liveness", () => HealthCheckResult.Healthy());
}
```

```C#
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    //...
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
        endpoints.MapHealthChecks("/health/live", new HealthCheckOptions
        {
            Predicate = r => r.Name.Contains("liveness"),
            ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
        });
    }
```

A health check UI-hoz az alábbi NuGet csomagot kell felvegyük a projektbe.

```xml
<PackageReference Include="AspNetCore.HealthChecks.UI" Version="3.0.4" />
```

Mi most csak a sorosító komponensét fogjuk használni belőle (`UIResponseWriter`), idő hiányában a UI komponenst most nem üzemeljük be.

Próbáljuk ki az új végpontot (F5).

#### Readiness

Vegyünk fel HC-t a külső szolgáltatásainkhoz is (Elasticsearch, Redis), majd ezt publikáljuk ki egy külön végponton.

Vegyük fel az következő csomagokat a Todos.Api projekthez.

```xml
<PackageReference Include="AspNetCore.HealthChecks.Elasticsearch" Version="3.0.0" />
<PackageReference Include="AspNetCore.HealthChecks.Redis" Version="2.2.1" />
```

!!! important "Verzió fontos"
    `AspNetCore.HealthChecks.Redis` csomag direkt a 2.2.1-es mert összeakadna a `Microsoft.Extensions.Caching.Redis` csomaggal az újabb verzió.

Vegyük fel a csekkolásokat.

```C#
services.AddHealthChecks()
    .AddCheck("liveness", () => HealthCheckResult.Healthy())
    .AddRedis(Configuration.GetValue<string>("RedisUrl") ?? "redis:6379", tags: new[] { "readiness" })
    .AddElasticsearch(Configuration.GetValue<string>("ElasticsearchUrl") ?? "http://elasticsearch:9200", tags: new[] { "readiness" });
```

!!! note "`IOptions<T>` használata"
    Az ASP.NET Core-os konfigurációk kezelésére itt is célszerűbb lenne az `IOptions<T>` mintát használni, de most az egyszerűség kedvéért ettől eltekintünk.

Publikáljuk ki egy végponton őket.

```C#
endpoints.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = r => r.Tags.Contains("readiness"),
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});
```

Próbáljuk ki!

### Kubernetes probes

Vegyük fel a kubernetes konfigurációba a liveness és a readiness próbákat.

```yaml
    spec:
      containers:
        - name: todos
          livenessProbe:
            httpGet:
              path: /health/live
              port: 80
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 80
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 10
```

Ha mi kívülről is meg akarjuk hívni a `/health` végpontokat, akkor vegyük fel őket az ingress konfigurációba.

```yaml
spec:
  rules:
    - http:
        paths:
          - path: /health/live
            backend:
              serviceName: todos # A service neve
              servicePort: http # A service-ben a port neve (lehet a port szama is)
          - path: /health/ready
            backend:
              serviceName: todos # A service neve
              servicePort: http # A service-ben a port neve (lehet a port szama is)
```

Próbáljuk ki!

Rontsuk el a readiness próbát úgy, hogy elírjuk az elasticsearch connection string-jét a környezeti változóban. Azt tapasztalhatjuk, hogy az új konténer elindult, de mivel nem ready ezért a régi szerepét nem tudja átvenni addig, amíg ready nem lesz. Sajnos ez a hiba nem tud kijavulni magától, így a kijavított config után fog indulni a pod.

Készítsünk egy egyszerű módszert a liveness próba elrontására a `Startup` osztályban a healthcheck létrehozásakor.

```C#
public bool IsLive { get; private set; } = true;
```

```C#
services.AddHealthChecks()
    .AddCheck("liveness", () => IsLive ? HealthCheckResult.Healthy() : HealthCheckResult.Unhealthy())
```

```C#
endpoints.MapGet("/health/switch", async r =>
{
    IsLive = !IsLive;
    await r.Response.WriteAsync($"IsLive is now {IsLive}");
});
```

Engedélyezzük kívülről ezt a végpontot is.

```yaml
          - path: /health/switch
            backend:
              serviceName: todos # A service neve
              servicePort: http # A service-ben a port neve (lehet a port szama is)
```

Telepítsük ki, majd rontsuk el a működést a végpontunkkal. Közben figyeljük a podok állapotát. Egy idő után láthatjuk, hogy a próba sérült, és a k8s megpróbálja újraindítani a pod-ot, mivel ott már az `IsLive` property érteke igaz lesz.

Érdemes minden health check definiálása során megtervezni azt, hogy az most melyik próbába illik bele jobban. Ha van esély, hogy magától megjavuljon, akkor a readiness próbába érdemes rakni (pl. valamilyen külső szolgáltatás nem elérhető, persze ez lehet konfigurációs hiba is, ahogy láttuk), ha pedig újraindítás tud segíteni akkor a liveness próbába rakjuk.
