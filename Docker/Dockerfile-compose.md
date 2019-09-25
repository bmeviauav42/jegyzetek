# Dockerfile és docker-compose

## Labor célja

A labor célja megismerni a Docker-alap fejlesztés módjait, a Dockerfile és docker-compose alapú megközelítéseket, valamint a Microsoft Visual Studio fejlesztőkörnyezet konténerfejlesztést támogató szolgáltatásait.

## Előkövetelmények

- Docker Desktop
- Docker-compose
  - Csak Linux esetén szükséges [külön telepíteni](https://docs.docker.com/compose/install/)
- Microsoft Visual Studio Code
  - Javasolt: [Docker extension](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-docker)
- Microsoft Visual Studio 2017/2019
  - 2017 esetén legalább 15.6 verzió
  - A _.NET Core cross-platform development_ nevű [workload](https://visualstudio.microsoft.com/vs/visual-studio-workloads/) szükséges

## Feladatok

### _Dockerfile_ készítése

Előző laboron készítettünk már egy saját image-et. Az image elkészítésére azonban nem az a módszer a javasolt megoldás, hanem a _Dockerfile_ alapú "recept készítés".

1. Készíts egy munkakönytárat, pl. `c:\work\<neptun>\dockerlab\pythonweb`
1. Nyisd meg a mappát Visual Studio Code-ban.
1. Készíts egy `Dockerfile` nevű fájt (kiterjesztés nélkül!) az alábbi tartalommal.

   ```Dockerfile
   FROM python:2.7-slim
   WORKDIR /app
   COPY . /app
   RUN pip install --trusted-host pypi.python.org -r requirements.txt
   EXPOSE 80
   ENV NAME NEPTUN # Ide a saját Neptun kódodat írd
   CMD ["python", "app.py"]
   ```

1. Készíts egy `requirements.txt` fájlt az alábbi tartalommal

   ```txt
   Flask
   Redis
   ```

1. Készíts egy `app.py` fájlt az alábbi tartalommal

   ```Python
   from flask import Flask
   from redis import Redis, RedisError
   import os
   import socket

   # Connect to Redis
   redis = Redis(host="redis", db=0, socket_connect_timeout=2, socket_timeout=2)

   app = Flask(__name__)

   @app.route("/")
   def hello():
       try:
           visits = redis.incr("counter")
       except RedisError:
           visits = "<i>cannot connect to Redis, counter disabled</i>"

       html = "<h3>Hello {name}!</h3><b>Visits:</b> {visits}"
       return html.format(name=os.getenv("NAME", "world"), visits=visits)

   if __name__ == "__main__":
       app.run(host='0.0.0.0', port=80)
   ```

1. Készítsd el a fenti fájlokból az image-et. _PowerShell_ konzolból a munkakönyvtárban add ki a következő parancsot: `docker build -t hellopython:v1 .` (a végén egy pont van, az is a parancs része!)
1. Ellenőrizd, hogy tényleg létrejött-e az image.
1. Indíts el egy új konténert ebből az image-ből: `docker run -it --rm -p 8085:80 hellopython:v1`
1. Nyisd meg böngészőben a <http://localhost:8085> oldalt.

### Dockerignore és build contextus

A `Dockerfile`-ban hivatkoztunk az aktuális könyvtárra a `.`-tal. Vizsgáljuk meg, hogy ez mit is jelent.

1. Készítsünk az aktuális könyvtárunkba, az `app.py` mellé egy nagy fájlt. PowerShell-ben addjuk ki a következő parancsot.

   ```powershell
   $out = new-object byte[] 134217728; (new-object Random).NextBytes($out); [IO.File]::WriteAllBytes("$pwd\file.bin", $out)
   ```

1. Buildeljük le ismét a fenti image-et: `docker build -t hellopython:v1 .`

   Menet közben látni fogjuk a következő sort a build logban: _Sending build context to Docker daemon 111.4MB_, és azt is tapasztalni fogjuk, hogy ez el tart egy kis ideig.

   A `docker build` parancs végén a `.` az aktuális könyvtár. Ezzel tudatjuk a Docker-rel, hogy a buildeléshez ezt a _kontextust_ használja, azon fájlok legyenek elérhetőek a build során, amelyek ebben a kontextusban vannak. A `Dockerfile`-ban a `COPY` így **relatív** útvonallal hivatkozik a kontextusban levő fájlokra.

   Ennek következménye az is, hogy csak a build kontextusban levő fájlokra tudunk hivatkozni. Tehát nem lehet pl. `COPY ..\..\file` használatával tetszőleges fájlt felmásolni a build közben.

1. Ha a build kontextusból szeretnénk kihagyni fájlokat, hogy a build ne tartson sokáig, akkor egy `.dockerignore` fájlra lesz szükségünk (a `.gitignore` mintájára). Ide szokás például a build környezet saját könyvtárait (obj, bin, .vs, node_modules, stb.) is felvenni.

   Készítsünk egy `.dockerignore`-t az alábbi tartalommal

   ```txt
   file.bin
   ```

1. Futtassuk ismét a buildet. Így már gyorsabb lesz.

### Docker-compose

A fenti alkalmazás egy része még nem működik. A python alkalmazás mellett egy redis-re is szükségünk lenne. Futtassunk több konténert egyszerre.

1. Visual Studio Code-ban nyisd meg a mappát, amiben az előző feladat során használt _pythonweb_ mappa van. Tehát nem az előzőleg használt mappa kell, hanem egy szinttel feljebb (pl. `c:\work\<neptun>\dockerlab`).

1. Készíts ide egy `docker-compose.yml` nevű fájlt az alábbi tartalommal.

   ```yaml
   version: "3"

   services:
     redis:
       image: redis:5.0.5-alpine
       networks:
         - mikroszolg_network
     web:
       build: pythonweb
       ports:
         - 5000:80
       depends_on:
         - redis
       networks:
         - mikroszolg_network

   networks:
     mikroszolg_network:
       driver: bridge
   ```

1. Nyiss egy _PowerShell_ konzolt ugyanebbe a mappába. Indítsd el az alkalmazásokat az alábbi paranccsal: `docker-compose up --build`
   - Két lépésben a parancs: `docker-compose build` és `docker-compose up`
1. Nyisd meg böngészőben a <http://localhost:8085> oldalt.
1. Egy új konzolban nézd meg a futó konténereket a `docker ps` parancs segítségével.

> A Docker-compose alkalmas üzemeltetésre is. A `docker-compose.yml` fájl nem csak fejlesztői környezetet ír le, hanem üzemeltetéshez szükséges környezetet is. Ha a compose fájlt megfelelően írjuk meg (pl. használjuk a [`restart` direktívát](https://docs.docker.com/compose/compose-file/#restart) is), az elindított szolgáltatások automatikusan újraindulnak a rendszer indulásakor.

### Több compose yaml fájl

A docker-compose parancsnak nem adtuk meg, hogy milyen yaml fájlból dolgozzon. Alapértelmezésként a `docker-compose.yaml` kiterjesztésű fájlt **és** ezzel összefésülve a `docker-compose.override.yaml` fájlt használja.

1. Készíts egy `docker-compose.override.yaml` fájlt a másik compose yaml mellé az alábbi tartalommal

   ```yaml
   version: "3"

   services:
     redis:
       command: redis-server --loglevel verbose
   ```

1. Indítsd el a rendszert `docker-compose up` paranccsal. A redis konténer részletesebben fog naplózni a `command` direktívában megadott utasítás szerint. Állítsd le a rendszert.

1. Nevezd át az előbbi override fájlt `docker-compose.debug.yaml`-re. Készíts egy új `docker-compose.prod.yaml` fájlt a többi yaml mellé az alábbi tartalommal

   ```yaml
   version: "3"

   services:
     redis:
       command: redis-server --loglevel warning
   ```

1. Indítsuk el a rendszert az alábbi paranccsal

   ```bash
   docker-compose -f docker-compose.yml -f docker-compose.prod.yml up
   ```

   A `-f` kapcsolóval tudjuk kérni a megadott yaml fájlok összefésülését.

> Általában a `docker-compose.yaml`-be kerülnek a közös konfigurációk, és a további fájlokba a környezet specifikus konfigurációk.

### Tipikusan használt image-ek

Konténer alapú fejlesztésnél tehát két féle image-et használunk:

- amit magunk készítünk egy adott alap image-re épülve,
- illetve kész image-eket, amiket csak futtatunk (és esetleg konfigurálunk).

Alap image-ek, amikre tipikusan saját alkalmazást építünk:

- Linux disztribúciók
  - [Ubuntu](https://hub.docker.com/_/ubuntu), pl. `ubuntu:18.04`
  - [Debian](https://hub.docker.com/_/debian), pl. `debian:stretch`
  - [Alpine](https://hub.docker.com/_/alpine), pl. `alpine:3.10`
- Futtató platformok
  - [.NET Core](https://hub.docker.com/_/microsoft-dotnet-core), pl. `mcr.microsoft.com/dotnet/core/runtime:2.1.12-bionic`
  - [OpenJDK](https://hub.docker.com/_/openjdk) pl. `openjdk:11-jre-stretch`
  - [NodeJS](https://hub.docker.com/_/node) pl. `node:12.7.0-alpine`
  - [Python](https://hub.docker.com/_/python) pl. `python:2.7-slim`
- [scratch](https://hub.docker.com/_/scratch): üres image, speciális esetek, pl. go

A kész image-ek, amiket pedig felhasználunk:

- Adatbázis szerverek, webszerverek, gyakran használt szolgáltatások
- MSSQL Server, redis, mongodb, mysql, nginx, ...
- Termérdek elérhető image: <https://hub.docker.com>

> Az image-ek verziózását minden esetben meg kell érteni! Minden image más-más megközelítést alkalmaz.

### Kész image testreszabása

Az előbb a _redis_ memória alapú adatbázist minden konfiguráció nélkül felhasználtuk. Gyakran az ilyen image-ek "majdnem" jók közvetlen felhasználásra, de azért szükség van egy kevés testreszabásra. Ilyen esetben a következő lehetőségeink vannak:

- Saját image-et készítünk kiindulva a számunkra megfelelő alap image-ből. A saját image-ben módosíthatunk a konfigurációs fájlokat, avagy további fájlokat adhatunk az image-be. Ezt a megoldást alkalmazhatjuk például tipikusan weboldal kiszolgálásánál, ahol is a kiinduló image a webszerver, viszont a kiszolgálandó fájlokat még mellé kell tennünk.

- Környezeti változókon keresztül konfiguráljuk a futtatandó szolgáltatást. Az alap image-ek általában elég jó konfigurációval rendelkeznek, csak keveset kell rajta módosítanunk. Erre tökéletesen alkalmas egy-egy környezeti változó. A jól felépített Docker image-ek előre meghatározott környezeti változókon keresztül testreszabhatóak.

  Erre egy jó példa a [Microsoft SQL Server Docker változata](https://hub.docker.com/_/microsoft-mssql-server). Az alábbi parancsban a `-e` argumentumokban adunk át környezeti változókat:

  ```bash
  docker run
     -e 'ACCEPT_EULA=Y'
     -e 'SA_PASSWORD=yourStrong(!)Password'
     -p 1433:1433
     mcr.microsoft.com/mssql/server:2017-CU8-ubuntu
  ```

- Becsatolhatjuk a saját konfigurációs fájljainkat a konténerbe. Korábban láttuk, hogy lehetőségünk van egy mappát a host géptről a konténerbe csatolni. Ha elkészítjük a testreszabott konfigurációs fájlt, akkor a `docker-compose.yml` leírásban a következő módon tudjuk ezt a fájlt becsatolni a konténer indulásakor.

  ```yaml
  services:
  ---
  redis:
    image: redis:5.0.5-alpine
    volumes:
      - my-redis.conf:/usr/local/etc/redis/redis.conf
  ```

  Ezen megoldás előnye, hogy nincs szükség saját image-et készíteni, tárolni, kezelni. Amikor a környezeti változó már nem elegendő a testreszabáshoz, ez a javasolt megoldás.

### Microsoft Visual Studio támogatás Docker-alapú fejlesztéshez

A Microsoft Visual Studio a 2017-es verzió óta támogatja és megkönnyíti a konténer alapú szoftverfejlesztést. Segíti a fejlesztőt a fejlesztett alkalmazás konténerizálásában, és támogatja a konténerben való debuggolást is.

1. Engedélyezzük Docker-ben a volume sharing-et!
1. Indítsuk el a Visual Studio-t (nem a Code-ot!).
1. Készítsünk egy új _ASP.NET Core Web Application_ típusú projektet a _Web application_ sablonnal, és engedélyezzük a Linux-alapú konténer támogatást a projektben. (A pontos lépések Visual Studio verzió függőek, lásd [itt](https://docs.microsoft.com/en-us/aspnet/core/host-and-deploy/docker/visual-studio-tools-for-docker).)
1. Nézzük meg és értsük meg az elkészült solution struktúrát, valamint a _Dockerfile_-t és a _docker-compose_ fájlokat is.
1. Indítsuk el az alkalmazást Docker-compose használatával: legyen a _docker-compose_ a startup projektünk, és F5-tel indítsuk debug módban az alkalmazást.
1. Egy konzolból listázzuk ki a futó konténereket: `docker ps`

> A Visual Studio Code is támogatja a konténerek debuggolását. Erről részletesen lásd [itt](https://code.visualstudio.com/docs/remote/containers).

## További olvasnivaló

- _Dockerfile_ szintaktika: <https://docs.docker.com/engine/reference/builder/>
- `.dockerignore` fájl szintaktika: <https://docs.docker.com/engine/reference/builder/#dockerignore-file>
- _Dockerfile_ best practice-ek: <https://docs.docker.com/develop/develop-images/dockerfile_best-practices/>
- _compose_ fájl szintaktika: <https://docs.docker.com/compose/compose-file/>
- Több compose fájl használata: <https://docs.docker.com/compose/extends/#multiple-compose-files>
- Multistage build-ek: <https://docs.docker.com/develop/develop-images/multistage-build/>
