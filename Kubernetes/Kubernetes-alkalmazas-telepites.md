# Alkalmazás telepítése Kubernetes klaszterbe

## Labor célja

A labor célja egy alkalmazás telepítése Kubernetes klaszterbe, valamint a frissítés módjának megismerése. A telepítéshez és frissítéshez részben Helm chartot, részben magunk által elkészített yaml erőforrás leírókat használunk.

## Előkövetelmények

- Kubernetes
  - Bármely felhő platform által biztosított klaszter
  - Linux platformon: [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube)
  - Windows platformon: Docker Desktop
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
  - A binárisa legyen elérhető PATH-on.
- [helm](https://helm.sh/docs/using_helm/#installing-helm)
  - A binárisa legyen elérhető PATH-on.
- Kubernetes dashboard
  - Telepítése és használata korábbi anyag szerint.
- Telepítendő alkalmazás
  - <https://github.com/bmeviauav42/todoapp/tree/megoldas/kubernetes>
  - `git clone -b megoldas/kubernetes https://github.com/bmeviauav42/todoapp.git`

## Feladatok

### Célok, tervezés

A célunk a [todo-kat kezelő konténeralapú mikroszolgáltatásokra épülő webalkalmazás](https://github.com/bmeviauav42/todoapp) telepítése Kubernetes-be. A rendszerünk alapvetően három féle komponensből épül fel:

- az általunk megvalósított mikroszolgáltatások (backend és frontend),
- az adatbázis rendszerek (MongoDB, Elasticsearch és Redis),
- valamint az api gateway.

Célunk nem csak az egyszeri telepítés, hanem az alkalmazás naprakészen tartásához a folyamatos frissítés lehetőségének megteremtése. A fenti komponensek azonban nem ugyanolyan frissítési ciklussal rendelkeznek: a saját komponenseink gyakran fognak változni, míg az adatbázisok és az api gateway ritkán frissül. A telepítést ennek megfelelően ketté vágjuk:

1. Az api gateway-t és az adatbázisokat egyszer telepítjük.
1. Az alkalmazásunk saját komponenseihez yaml alapú leírókat készítünk, amit `kubectl apply` segítségével fogunk telepíteni.

### Helm inicializálása

Mi a Helm 2-es verzióját fogjuk használni. (A 3-as verzió még csak release candidate állapotban van.) Ezen verzióban a Helm egy un. Tiller komponenst használt, amely a Kubernetes klaszterben fut. A Helm CLI ezzel a Tiller-rel beszélgetve éri el a klasztert. Ezért első feladatunk ezen Tiller telepítése. Ezt egy klaszter esetében csak egyetlen egyszer kell megtenni.

1. Ellenőrizzük, hogy a `helm` CLI elérhető-e: `helm version`

1. Inicializáljuk a Helm-et, ami egyben telepíti a Tiller komponenst a Kubernetes klaszterbe: `helm init`

   - Ez a parancs két dolgot végzett el.
     1. Inicializálta a kliens oldalon, a gépünkön a Helm környezetét.
     1. Telepítette a klaszterbe a Tiller-t.

1. Nézzük meg, hogy tényleg van a klaszterben egy Tiller pod: `kubectl get pod -n kube-system`

1. Kérdezzük le ismét a verziót: `helm version`

   - Most már nem csak a kliens oldali CLI, hanem a Kubernetes-ben futó szerver oldal verzióját is megkapjuk.

Megjegyzés: ha a Kubernetes klaszterben már fut Tiller, akkor is szükségünk lehet a Helm inicializálására egy új fejlesztői gépen. Ilyenkor a `--client-only` kapcsolóval kérhetjük csak a kliens oldal inicializálását. Új Tiller verzió telepítését pedig a `helm init --upgrade` paranccsal végezhetjük el.

### Ingress Controller (api gateway) telepítése Helm charttal

A Traefik-et Helm charttal fogjuk telepíteni. [Ezen Helm chart](https://github.com/helm/charts/tree/master/stable/traefik) **nem hivatalos**, harmadik féltől származik, ezért a használat előtt mindig érdemes alaposan megnézni a tartalmát. Nekünk most az egyszerűség végett praktikus lesz a Helm chart, mert a Traefik helyes működéséhez a Traefik konténer (Deployment) mellett egyéb elemekre is szükség lesz (klaszteren belüli hozzáférés szabályzás miatt).

(Jelen pillanatban a Traefik Helm chart-ja csak az 1.7-es verziót támogatja. Készül egy [hivatalos Helm chart is a Traefik 2-es verziójához](https://github.com/containous/traefik-helm-chart), de ez még nem production ready, ezért most maradunk az 1.7-es Traefik verziónál.)

1. Frissítsük be a Helm repository-nkat: `helm repo update`.

1. Telepítsük: `helm install stable/traefik --name traefik --version "1.78.3" --set rbac.enabled=true --set logLevel=debug --set dashboard.enabled=true --set service.nodePorts.http=30080 --set serviceType=NodePort`

   - A `name` a Helm release nevét adja meg. Ezzel tudunk rá hivatkozni a jövőben.
   - A `--set` kapcsolóval a chart változóit állítjuk be.

1. Ellenőrizzük, hogy fut-e: `kubectl get pod`

   - Látunk kell egy traefik kezdetű podot.

A Traefik jelen konfigurációban _NodePort_ service típussal van konfigurálva, ami azt jelenti, lokálisan, helyben a megadott porton lesz csak elérhető. Ha publikusan elérhető klaszterben dolgozunk, akkor tipikusan _LoadBalancer_ service típust fogunk kérni, hogy publikus IP címet is kapjon a Traefik.

Ha frissíteni szeretnénk később a Traefik-et, akkor azt a `helm upgrade traefik stable/traefik ...` paranccsal tudjuk megtenni.

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

   - Mindenhol, ahol _labels_ vagy _matchLabels_ szerepel, még egy sort fel kell vennünk: `app.kubernetes.io/instance: {{ .Release.Name }}` Ez egy implicit változót helyettesít be: a release nevét. Ezzel azonosítja a Helm a telepítés és frissítés során, hogy mely elemeket kell frissítenie, melyek tartoznak a fennhatósága alá. Mi most mindent egy névtérbe raktunk. A Helm telepítés ezt nem fogja tönkretenni, mert az eddigi elemeinket nem fogja érinteni ezen label miatt.

   - A pod-ban az image beállításához használjunk változót: `image: todoapp/todos:{{ .Values.todos.tag }}`

1. Definiáljuk az előbbi változó alapértelmezett értékét. A `values.yaml` fájlban (egy könyvtárral feljebb) töröljünk ki mindent, és vegyük fel ezt a kulcsot:

   ```yaml
   todos:
     tag: v1
   ```

1. A másik két komponens yaml leíróival is hasonlóan kell eljárnunk.

1. Nézzük meg a template-eket kiértékelve.

   - A chartunk könyvtárából lépjünk eggyel feljebb, hogy a `todoapp` chart könyvtár az aktuális könyvtárban legyen.
   - Futtassuk le csak a template generálást a telepítés nélkül: `helm install --debug --dry-run todoapp`
   - Konzolra megkapjuk a kiértékelt yaml-öket.

1. Töröljük ki a telepített alkalmazásunkat, mert összeakadna a Helm-mel. Eyt a parancsot a telepítéshez korábban használt `app` könyvtárban adjuk ki: `kubectl delete -f app`

1. Telepítsük újra az alkalmazást a chart segítségével: `helm upgrade todoapp-prod --install todoapp`

   - A release-nek _todoapp-prod_ nevet választottunk. Ez a Helm release azonosítója.

   - Az _upgrade_ parancs és az _--install_ kapcsoló telepít, ha nem létezik, ill. frissít, ha már létezik ilyen telepítés.

1. Nézzük meg, hogy a Helm szerint létezik-e a release: `helm list`

   - Azt, hogy létezik-e már egy release a klaszterben, és mi a verziója (saját belső verzió, tőlünk független), onnan tudja a Helm, hogy a telepítés végeztével elmenti az állapotát egy ConfigMap-be.

1. Próbáljuk ki az alkalmazást a <http://localhost:30080> címen.

1. Ezen chart segítségével a Docker image tag-et telepítési paraméterben adhatjuk át, pl. ha a "v2" az új tag, akkor egy paraccsal tudjuk frissíteni: `helm upgrade todoapp-prod --install todoapp --set todos.tag=v2`.
