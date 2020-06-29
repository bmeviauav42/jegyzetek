---
title: Alkalmazás telepítése Kubernetes klaszterbe
---

## Labor célja

A labor célja egy alkalmazás telepítése Kubernetes klaszterbe, valamint a frissítés módjának megismerése. A telepítéshez és frissítéshez részben Helm chartot, részben magunk által elkészített yaml erőforrás leírókat használunk.

## Előkövetelmények

- Kubernetes
    - Bármely felhő platform által biztosított klaszter
    - Linux platformon: [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube)
    - Windows platformon: Docker Desktop
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
    - A binárisa legyen elérhető PATH-on.
- [helm v3](https://helm.sh/docs/intro/install/)
    - A binárisa legyen elérhető PATH-on.
- Kubernetes dashboard
    - Telepítése és használata korábbi anyag szerint.
- Telepítendő alkalmazás
    - <https://github.com/bmeviauav42/todoapp>
    - `git clone https://github.com/bmeviauav42/todoapp.git`

## Feladatok

### Célok, tervezés

A célunk a [todo-kat kezelő konténeralapú mikroszolgáltatásokra épülő webalkalmazás](https://github.com/bmeviauav42/todoapp) telepítése Kubernetes-be. A rendszerünk alapvetően három féle komponensből épül fel:

- az általunk megvalósított mikroszolgáltatások (backend és frontend),
- az adatbázis rendszerek (MongoDB, Elasticsearch és Redis),
- valamint az api gateway.

Célunk nem csak az egyszeri telepítés, hanem az alkalmazás naprakészen tartásához a folyamatos frissítés lehetőségének megteremtése. A fenti komponensek azonban nem ugyanolyan frissítési ciklussal rendelkeznek: a saját komponenseink gyakran fognak változni, míg az adatbázisok és az api gateway ritkán frissül. A telepítést ennek megfelelően ketté vágjuk:

1. Az api gateway-t és az adatbázisokat egyszer telepítjük.
1. Az alkalmazásunk saját komponenseihez yaml alapú leírókat készítünk, amit `kubectl apply` segítségével fogunk telepíteni.

### Helm

Ellenőrizzük, hogy a `helm` CLI elérhető-e: `helm version`

!!! note
    A feladat során a Helm 3-as verzióját fogjuk használni. A korábbi verziója koncepcióban azonos, de működésében eltérő.

### Ingress Controller (api gateway) telepítése Helm charttal

A Traefik-et [Helm charttal](https://github.com/containous/traefik-helm-chart) fogjuk telepíteni, mert a Traefik helyes működéséhez a Traefik konténer (Deployment) mellett egyéb elemekre is szükség lesz (klaszteren belüli hozzáférés szabályzás miatt).

!!! important
    A Helm chartok nagy része harmadik féltől származik, így a klaszterünbe való telepítés előtt a tartalmukat érdemes alaposan megnézni.

1. A Helm is repository-kkal dolgozik, ahonnan a chart-okat letölti. Ezeket regisztrálni kell. Regisztráljuk a Traefik hivatalos chart-ját tartalmazó repository-t, majd frissítsük az elérhető char-okat:

    ```bash
    helm repo add traefik https://containous.github.io/traefik-helm-chart
    helm repo update
    ```

1. Telepítsük: `helm install traefik traefik/traefik --set ports.web.nodePort=32080 --set service.type=NodePort`

     - A legelső `traefik` a Helm release nevét adja meg. Ezzel tudunk rá hivatkozni a jövőben.
     - A `traefik/traefik` azonosítha a telepítendő chartot (repository/chartnév).
     - A `--set` kapcsolóval a chart változóit állítjuk be.

    !!! info
        A Traefik jelen konfigurációban _NodePort_ service típussal van konfigurálva, ami azt jelenti, lokálisan, helyben a megadott porton lesz csak elérhető. Ha publikusan elérhető klaszterben dolgozunk, akkor tipikusan _LoadBalancer_ service típust fogunk kérni, hogy publikus IP címet is kapjon a Traefik.

1. Ellenőrizzük, hogy fut-e: `kubectl get pod`

     - Látunk kell egy traefik kezdetű podot.

1. A Traefik dashboard-ja nem elérhető "kívülről". A dashboard segít minket látni a Traefik konfigurációját és működését. Mivel ez a klaszter belső állapotát publikálja, production üzemben valamilyen módon authentikálnunk kellene. Ezt most megkerülve `kubect` segítségével egy helyi portra továbbítjuk a Traefik dashboard-ot. A következő parancsot egy **új** konzolban adjuk ki, és utána hagyjuk nyitva a konzolt:

    ```bash
    kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" --output=name) 9000:9000
    ```

!!! note
    Ha frissíteni szeretnénk később a Traefik-et, akkor azt a `helm upgrade traefik traefik/traefik ...` paranccsal tudjuk megtenni.

### Adatbázisok telepítése

Az adatbázisainkat saját magunk által megírt yaml leíróval telepítjük. Ez a leíró fájl már rendelkezésünkre áll a todoapp alkalmazás repositoryjában.

1. Navigáljunk el a forráskód repository-jában a _kubernetes/db_ könyvtárba. Nézzük meg a yaml leírókat.
1. Telepítsük az adatbázisokat: `kubectl apply -f db`
1. Kapcsolódjunk a Kubernetes Dashboard-hoz a korábban ismertetett módon (ill. telepítsük a klaszterbe, ha nem lenne).
1. Ellenőrizzük, hogy az adatbázis podok elindulnak-e. Minden a _default_ névtérbe kellett települjön.
1. Nézzük meg a _Persistent Volume_ és _Persistent Volumen Claim_-eket.

### Alkalmazásunk telepítése

Az alkalmazásunk telepítéséhez szintén yaml leírókat találunk a _kubernetes/app_ könyvtárban.

1. Nézzük meg a leírókat. Az előbb látott Deployment és Service mellet Ingress-t is látni fogunk.

1. Telepítsük az adatbázisokat: `kubectl apply -f app`

1. Kubernetes dashboard segítségével nézzük meg, hogy létrejöttek a Deployment-ek podok, stb. Viszont vegyük észre, hogy piros lesz még pár dolog. A hiba oka, hogy a hivatkozott image-eket nem találja a rendszer.

1. Navigáljunk el az `src/Docker` könyvtárba, és buildeljük le az alkalmazást docker-compose segítségével úgy, hogy a megfelelő taggel ellátjuk az image-eket. Ehhez a compose fájlunk az `IMAGE_TAG` környezeti változó beállítását várja.

    - Powershell-ben

        ```powershell
        $env:IMAGE_TAG="v1"
        docker-compose build
        ```

    - Windows Command Prompt-ban

        ```cmd
        setx IMAGE_TAG "v1"
        docker-compose build
        ```

1. A build folyamat végén előállnak helyben az image-ek, pl. `todoapp/web:v1` taggel. Ha távoli registry-be szeretnénk feltölteni őket, akkor taggelhetnénk őket a registry-nek megfelelően. A helyi fejlesztéshez nincs szükségünk ehhez, mert a helyben elindított Kubernetes "látja" a Docker helyi image-eit.

1. Menjünk vissza a Kubernetes dashboard-ra. Egy kicsit várjunk, és azt kell lássuk, hogy az eddig piros elemek kizöldülnek. A Kubernetes folyamatosan törekszik az elvárt állapot elérésére, ezért a nem elérhető image-einket újra és újra megpróbálta elérni, míg nem sikerült.

1. Próbáljuk ki az alkalmazást a <http://localhost:30080> címen

### Alkalmazás frissítése

Tegyük fel, hogy az alkalmazásunkból újabb verzió készül, és szeretnénk frissíteni. A fentebb használt yaml leírókat azért (is) a verziókezelőben tároljuk, mert így a telepítési "útmutatók" is verziózottak. Tehát nincs más dolgunk, mit a konténer image-ek elkészítése után a Deployment-ekben a megfelelő tag-ek lecserélése, és a `kubectl apply` paranccsal a telepítés frissítése.

### Helm chart készítése

Az alkalmazás fent ismertetett frissítéséhez a yaml fájlokba minden alkalommal bele kell írnunk. Jó lenne, ha az image tag-et mint egy változó tudnánk atelepítés során átadni. Erre szolgál a Helm. Készítsünk egy _chart_-ot a szolgáltatásainknak. A _chart_ a telepítés leíró neve, ami gyakorlatilag yaml fájlok gyűjteménye egy speciális szintaktikával kiegészítva.

1. Konzolban navigáljunk egy el tetszőleges munkakönyvtárba.

1. Készítsünk egy új, üres chart-ot: `helm create todoapp`. Ez létrehoz egy _todoapp_ nevű chartot egy azonos nevű könyvtárban.

1. Nézzük meg a chart fájljait.

    - `Chart.yaml` a metaadatokat írja le.
    - `values.yaml` írja le a változóink alapértelmezett értékeit.
    - `.helmignore` azon fájlokat listázza, amelyeket a chart értelmezésekor nem kell figyelembe venni.
    - `templates` könyvtárban vannak a template fájlok, amik a generálás alapjául szolgálnak.

    A Helm egy olyan template nyelvet használ, amelyben változó behelyettesítések, ciklusok, egyszerű szövegműveletek támogatottak. Mi most csak a változó behelyettesítést fogjuk használni.

1. Töröljük ki a `templates` könyvtárból az összes fájlt a `_helpers.tpl` kivételével. Másoljuk helyette be ide a korábban a telepítéshez használt yaml fájljainkat (3 darab).

1. Szerkesszük meg a `todos.yaml` fájl tartalmát. Leegyszerűsítve az alábbiakra lesz szükség:

    - Ha _Visual Studio Code_-ot használunk, akkor telepítsük a [`ms-kubernetes-tools.vscode-kubernetes-tools`](https://marketplace.visualstudio.com/items?itemName=ms-kubernetes-tools.vscode-kubernetes-tools) extension-t. Így kapunk némi segítséget a szintaktikával.

    - Mindenhol, ahol _labels_ vagy _matchLabels_ szerepel, még egy sort fel kell vennünk:

        `app.kubernetes.io/instance: {{ .Release.Name }}`

        Ez egy implicit változót helyettesít be: a _release_ nevét. Ezzel azonosítja a Helm a telepítés és frissítés során, hogy mely elemeket kell frissítenie, melyek tartoznak a fennhatósága alá.

    - A pod-ban az image beállításához használjunk változót:

        `image: todoapp/todos: {{ .Values.todos.tag }}`

1. Definiáljuk az előbbi változó alapértelmezett értékét. A `values.yaml` fájlban (egy könyvtárral feljebb) töröljünk ki mindent, és vegyük fel ezt a kulcsot:

    ```yaml
    todos:
      tag: v1
    ```

1. A másik két komponens yaml leíróival is hasonlóan kell eljárnunk.

1. A továbbiakhoz el kell távolítanunk az előbb telepített alkalmazásunkat, mert összeakadna a Helm-mel. Ezt a parancsot a telepítéshez korábban használt `app` könyvtárban adjuk ki: `kubectl delete -f app`

1. Nézzük meg a template-eket kiértékelve.

    - A chartunk könyvtárából lépjünk eggyel feljebb, hogy a `todoapp` chart könyvtár az aktuális könyvtárban legyen.
    - Futtassuk le csak a template generálást a telepítés nélkül: `helm install todoapp --debug --dry-run todoapp`
    - Konzolra megkapjuk a kiértékelt yaml-öket.

1. Telepítsük újra az alkalmazást a chart segítségével: `helm upgrade todoapp --install todoapp`

    - A release-nek _todoapp_ nevet választottunk. Ez a Helm release azonosítója.

    - Az _upgrade_ parancs és az _--install_ kapcsoló telepít, ha nem létezik, ill. frissít, ha már létezik ilyen telepítés.

1. Nézzük meg, hogy a Helm szerint létezik-e a release: `helm list`

1. Próbáljuk ki az alkalmazást a <http://localhost:30080> címen.

1. Ezen chart segítségével a Docker image tag-et telepítési paraméterben adhatjuk át, pl. ha a "v2" az új tag, akkor egy paranccsal tudjuk frissíteni: `helm upgrade todoapp --install todoapp --set todos.tag=v2`
