# Kubernetes alapok

## Előadás

[Kubernetes bevezető](https://www.aut.bme.hu/Upload/Course/VIAUAV42/hallgatoi_jegyzetek/10-Kubernetes.pdf)

## Cél

A labor célja megismerni a Kubernetes használatának alapjait, a _podok_, _Deployment-ek_ és _ReplicaSet-ek_ létrehozását és kezelését, valamint a leggyakrabban használt `kubectl` parancsokat.

## Előkövetelmények

- Kubernetes
    - Bármely felhő platform által biztosított klaszter
    - Linux platformon: [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube)
    - Windows platformon: Docker Desktop
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
    - A binárisa legyen elérhető PATH-on.

## Feladatok

### Előkészület Docker Desktop-on

1. Praktikus, ha leállítunk minden futó konténert, amire nincs szükségünk. Használhatjuk a következő parancsot PowerShell-ben: `docker rm -f $(docker ps -aq)`

1. Nyissuk meg a Docker Desktop beállításait.

1. A _Kubernetes_ fülön pipáljuk be az _Enable Kubernetes_ opciót, és kattintsunk az _Apply_-ra.

1. Várjuk meg, amíg befejeződik a művelet.

### Kubectl csatlakozás a klaszterhez

- Ellenőrizzük, hogy a `kubectl` bináris elérhető-e, és tud-e csatlakozni a klaszterhez: `kubectl version`

    - A `kubectl` a CLI kliens a klaszter kezeléséhez. A Kubernetes API szerveréhez csatlakozik, annak REST API-ján keresztül végzi a műveleteket.
    - Láthatjuk mind a kliens, mind a klaszter verzió információit.

- A `kubectl` egy konkrét klaszterhez csatlakozik. Nézzük meg, milyen klasztereket ismer: `kubectl config get-contexts`

    - Ha több klaszterrel dolgoznánk, itt láthatnánk őket.
    - Ezek valójában egy konfigurációs fájlban vannak: `$HOME/.kube/config`
    - Váltani a `kubectl config use-context <név>` parancssal lehet.
    - Minden parancsnál külön megadhatjuk a kontextust a `--context` kapcsolóval, de inkább az implicit contextust szoktuk használni.
    - Kontextus beállításról részletesebben [itt](https://kubernetes.io/docs/reference/kubectl/cheatsheet/#kubectl-context-and-configuration).

### Podok és névterek listázása

- Listázzuk ki a futó podokat: `kubectl get pod -A`

    - A `-A` vagy `--all-namespaces` kapcsoló az összes névtérben levő podot listázza.

- Ismételjük meg a `-A` kapcsoló nélkül: `kubectl get pod`

    - Ez az alapértelmezett _default_ névtér podjait listázza.
    - (Az alapértelmezett névtér is a kontextus beállítása.)

- Nézzük meg, milyen névterek vannak: `kubectl get namespace`

- Listázzuk a podokat egy konkrét névtérben: `kubectl get pod -n kube-system`

### Pod létrehozása

A futtatás elemi egysége a pod. Indítsunk el egy podot.

1. A létrehozáshoz podot yaml leíróban definiáljuk, és a `kubectl create` parancsnak átadjuk stdin-ről olvasva (_Windows Command promptban_ adjuk ki a parancsot):

    ```bash
    kubectl create -f -
    apiVersion: v1
    kind: Pod
    metadata:
      name: counter
    spec:
      containers:
        - name: count
          image: ubuntu:16.04
          args: [bash, -c, 'for ((i=0; ;i++));do echo "$i: $(date)";sleep 5;done']
    ```

    A végén nyomjunk egy Ctrl-Z-t a leíró befejezésének jelzéséhez.

1. A pod létrejött. Ellenőrizzük: `kubectl get pod`

1. Nézzük meg a pod logjait: `kubectl logs counter`

    !!! tip ""
        Ha gondoljuk, tegyük hozzá a `-f` kapcsolót is (`kubectl logs -f counter`) a log követéséhez. Ctrl-C-vel léphetünk ki a log folyamatos követéséből.

1. Töröljük a podot: `kubectl delete pod counter`

1. Ellenőrizzük, hogy a pod tényleg eltűnik egy kis idő múlva: `kubectl get pod`

    !!! note ""
        A pod törlése nem azonnali. A benne futó konténerek leállás jelzést kapnak, és ők maguk terminálhatnak. Ha ez nem történik, meg, akkor kis idő múlva megszűnteti őket a rendszer.

### Yaml leíró fájl

A yaml leíró begépelése a fentiek szerint nem kényelmes. Tipikusan komplex pod és egyéb erőforrás definíciókkal dolgozunk. Tegyük inkább egy fájlba.

1. Hozzunk létre egy új yaml fájt `createpod.yml` néven.

    !!! tip ""
        Használhatjuk például Visual Studio Code-ot. Érdemes olyan szövegszerkesztővel dolgozni, amely ismeri a yaml szintaktikát.

1. Másoljuk be a yaml fájlba az alábbiakat.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: counter
    spec:
      containers:
        - name: count
          image: ubuntu:16.04
          args:
            [bash, -c, 'for ((i=0; ;i++));do echo "$i: $(date)";sleep 5;done']
    ```

1. A konzolunkban navigáljunk el abba a könyvtárba, ahol a yaml fájl van.

1. Hozzuk létre a podot: `kubectl create -f createpod.yml`

1. A pod létrejött. Ellenőrizzük: `kubectl get pod`, majd töröljük a podot: `kubectl delete pod counter`

### Deployment létrehozása

A podokat nem szoktuk közvetlenük létrehozni, hanem _Deployment_-re és _ReplicaSet_-re szoktunk bízni a kezelésüket és létrehozásukat.

1. Hozzunk létre egy új yaml fájl `createdepl.yml` néven az alábbi tartalommal.

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: counter
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: counter
      template:
        metadata:
          labels:
            app: counter
        spec:
          containers:
            - name: count
              image: ubuntu:16.04
              args:
                [bash, -c, 'for ((i=0; ;i++));do echo "$i: $(date)";sleep 5;done']
    ```

1. Hozzuk létre a Deployment-et: `kubectl apply -f createdepl.yml`

    !!! info ""
        Ezúttal nem `create`, hanem `apply` parancsot használunk. Az apply létrehozza, ha nem létezik, és módosítja az erőforrást, ha már létezik.

1. Listázzuk a Deployment-eket, ReplicaSet-eket és a podokat:

    - `kubectl get deployment`
    - `kubectl get replicaset`
    - `kubectl get pod`

    Vegyük észre, hogy a pod neve generált, a _Deployment_ és a _ReplicaSet_ alapján kap automatikusan egyet.

1. Változtassuk meg a program futását: ne 5, hanem 10 másodpercenként írjuk ki az időt. Ezt a _Deployment_ módosításával fogjuk megtenni. Gondolhatunk arra is, hogy a podot szerkesszük, de azt nem tehetjük meg. Egy futó pod nem cserélhető le. Ehelyett valójában egy új podot fogunk létrehozni indirekten.

    - Írjuk át a yaml fájlban a bash parancsban a sleep-et 10-re.
    - Mentsük a fájlt.
    - Alkalmazzuk a változtatásokat: `kubectl apply -f createdepl.yml`
    - Nézzük a podok változását: `kubectl get pod`
    - Időzítés függően jó eséllyel látni fogjuk a régi leálló podot, és az újat is.

1. "Lépjünk be" a futó podba egy új, interaktív shellben. Másoljuk ki a futó pod nevét, és adjuk ki a következő parancsot: `kubectl exec -it <podnév> /bin/bash`

    - Ahogy a docker-nél már láthattuk, egy új shell indul a pod konténerében, és ehhez csatlakozunk.
    - Ebben a shellben, ahogy natív docker esetében is, bármit megtehetünk.

1. Töröljük a _Deployment_-et: `kubectl delete deployment counter`

    - Ez a _ReplicaSet_-et és a podokat is törölni fogja.

!!! info ""
    A _Deployment_ szolgál az alkalmazás verziónak frissítésére, kiadására. Podok helyett leggyakrabban _Deployment_-eket definiálunk.

### Kubectl parancsok

A `kubectl` leggyakrabban használt parancsainak szerkezete: `kubectl <ige> <erőforrás> <attribútumok>`.

Az ige például:

- `get`: listázza az erőforrásokat
- `create`: létrehoz egy erőforrást
- `delete`: töröl egy erőforrást
- `describe`: lekérdezi az erőforrás részletes állapotát
- `edit`: letölti az erőforrás leíróját, és megnyitja szövegszerkesztőben; mentés és bezárás után frissíti a klaszterben az erőforrást a módosítások alapján

Az erőforrások a `pod`, `replicaset` vagy röviden `rs`, a `deployment`, stb.

A parancsokról `-h` kapcsolóval kaphatunk segítséget, pl. `kubectl describe -h`

### Dashboard és proxy-zás

A [_Web UI_ / _Dashboard_](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/) egy webalkalmazás, amely maga is Kubernetes alatt fut. Az alkalmazás a klaszter tartalmát jeleníti meg egy egyszerű, de könnyen áttekinthető webes felületen.

A dashboard alapvetően felhasználó authentikáció után érhető el. Az egyszerűség végett mi ezt most kikapcsoljuk, és authentikáció nélkül fogjuk használni.

1. Telepítsük a dashboard-ot:

    - Telepítsük alapbeállításokkal: `kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/alternative.yaml`

    - Szerkesszünk bele a konténer argumentumaiba, hogy engedélyezzük az anonim belépést: `kubectl edit deployment kubernetes-dashboard --namespace kubernetes-dashboard`

        A parancs letölti és megnyitja a _deployment_ leíróját. Keressük meg a konténer indítási argumentumait, és adjunk hozzá még két sort (ügyeljünk a megfelelő behúzásra):

        ```yaml hl_lines="4-5"
        - args:
          - --namespace=kubernetes-dashboard
          - --enable-insecure-login
          - --enable-skip-login
          - --authentication-mode=basic
        ```

        !!! warning "Csak fejlesztői gépen"
            Ez csak helyi működési módban javasolt! Most szándékosan kikerültük az authentikációt!

    !!! note ""
        Értelemszerűen a dashboard-ot csak egyszer kell egy klaszterbe telepíteni.

1. Adjunk egy pár másodpercet a felállásra. Nézzük meg, hogy rendben fut-e: `kubectl get pods -n kubernetes-dashboard`

    !!! success ""
        Akkor jó, ha a _kubernetes-dashboard-..._ nevű pod _running_ állapotban van.

1. A webalkalmazás csak a klaszteren belül érhető el egyelőre. Látni fogjuk később, hogyan tudunk a klaszteren kívülre "publikálni". Egyelőre azonban egy másik, fejlesztéshez kényelmesen használható megoldást alkalmazunk. Nyissunk egy **új** konzolt, és adjuk ki a `kubectl proxy` parancsot.

    - A futó proxy folyamatos kapcsolatot tart fenn a klaszterrel, és a Kubernetes Api szervert használja "belépési pontnak". Az proxy a helyi 8081 portra küldött kéréseket továbbítja a klaszteren belülre az URL alapján.

    - Ez a proxy nem csak helyben futtatott klaszterrel használható. Ha a felhőben, tőlünk távol fut a klaszter, ugyanígy tudunk vele kommunikálni.

1. Nyissuk meg böngészőben: <http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/kubernetes-dashboard:/proxy/>

    - Értelmezzük az URL-t:

        - `localhost:8001`: a proxy helyi portja
        - `/api/v1`: Kubernetes Api verzió, ez mindig így néz ki
        - `/namespaces/kubernetes-dashboard`: a _kubernetes-dashboard_ névtérben szeretnénk elérni egy elemet
        - `/services/kubernetes-dashboard:` a névtéren belül a service-ek közül a _kubernetes-dashboard_ nevű szolgáltatást szeretnénk elérni
        - `/proxy`: kérjük az erőforrás proxy-zását (ez mindig így néz ki)

1. Ismerkedjünk meg a dashboard webes felületével:

    - Nézzük meg a bal oldali menü elemeit.
    - Válasszunk névteret.
    - Nézzük meg a podokat.

### Pod - ReplicaSet - Deployment - Service kapcsolata

Használjuk a dashboard-ot a következőkhöz. A feladatban végig a _kubernetes-dashboard_ névtérben dolgozunk.

1. Keressük ki a "kubernetes-dashboard" nevű service-t.

1. Nézzük meg a service adatait, kifejezetten a "Pods" részt. Emlékezzünk arra, hogy a service teszi elérhetővé podok szolgáltatásait/végpontjait. A service-nek tehát tudnia kell, mely podok "állnak mögötte". Ezt látjuk a "Pods" listában.

1. Skálázzuk fel a dashboard-ot, futtassunk két példányt.

    - Keressük meg a "kubernetes-dashboard" deployment-et.
    - A webes felület tetején a kék színű sorban a jobb oldalon keressük meg a skálázás ikont.
    - Nyissuk meg, és kérjünk 3 példányt.

1. Nézzük meg a podok listáját. Lássuk, hogy most már 3 "kubernetes-dashboard-" névvel kezdődő pod lesz.

1. Térjünk vissza a "kubernetes-dashboard" service-hez. Nézzük meg, hogyan van összekötve a podokkal.

    - Nézzük meg a podokat. Vegyük észre, hogy a service "észrevette", hogy most már 3 pod felett rendelkezik.
    - Nézzük meg a service leíróját. A webes felületen a kék sárban a ceruza ikonra kattintsunk.
    - Keressük meg a `spec` leíróban a `selector`-t.

        ```yaml hl_lines="9"
        kind: Service
        apiVersion: v1
        metadata:
          name: kubernetes-dashboard # service elem neve
          ...
        spec:
          ...
          selector:
            k8s-app: kubernetes-dashboard # label, amelyet a podokon keres
        ```

        Ez a service azon podok felé osztja szét a kéréseket, amelyek a _k8s-app_ címkében _kubernetes-dashboard_ értékkel rendelkeznek.

    - Nézzük meg a podban ugyanezt. A service alatt bármelyik podra kattintva navigáljunk el a podhoz, és kérjük le a leíróját az előbbiek szerint. Keressük meg a pod metaadataiban a fenti labelt.

        ```yaml hl_lines="6"
        kind: Pod
        apiVersion: v1
        metadata:
          name: kubernetes-dashboard-57c9d8c7c4-z98z2 # pod saját, generált neve
          labels:
            k8s-app: kubernetes-dashboard # pod címkéje, amivel a service rátalál
        ```

1. A pod oldalán maradva keressük meg a "Controlled by" részt. Láthatjuk, hogy a podot egy ReplicaSet kezeli. Menjünk a ReplicaSet-hez. Nyissuk meg a ReplicaSet leíróját is.

      ```yaml
      kind: ReplicaSet
      apiVersion: extensions/v1beta1
      metadata:
        name: kubernetes-dashboard-57c9d8c7c4
        labels:
          k8s-app: kubernetes-dashboard # ReplicaSet címkéje, amivel a felette álló Deployment azonosítja őt
      spec:
        replicas: 3 # ennyi replikát kértünk
        selector:
          matchLabels:
            k8s-app: kubernetes-dashboard # Label, amivel a podjait azonosítja
        template: # pod definiálásához használt template, minden pod ez alapján jön létre
          metadata:
            labels:
              k8s-app: kubernetes-dashboard
              # amikor a ReplicaSet létrehoz egy podot, ezt a címkét teszi rá
              # meg kell egyezzen pár sorral feljebb a selector-ban definiálttak
          spec:
            containers: # leírja a podban a konténert
              - name: kubernetes-dashboard
                image: "kubernetesui/dashboard:v2.0.3"
                args:
                  - "--namespace=kubernetes-dashboard"
                ports:
                  - containerPort: 9090
                    protocol: TCP
      ```

    !!! info ""
        Ha még eggyel "feljebb" lépünk, a ReplicaSet-ből a Deployment-be, hasonló összekapcsolást találnánk.

### Pod "újraindítása"

1. Maradva a _kubernetes-dashboard_ névtérben listázzuk a podokat.

1. Válasszunk ki egy dashboard podot, és töröljük. A pod sor végén a ... ikon alatt válasszuk a _Delete_-et.

1. Várjunk egy pár másodpercet, és frissítsünk rá az oldalra.

1. Egy új pod született. Az _Age_ oszlop alapján láthatjuk. (Lehet, hogy a töröltet is látjuk még, leállás alatt.) Ez a ReplicaSet feladata: garantálni, hogy legyen 3 példány.

!!! note "Újraindítás"
    A pod törlése megfelel egy komponens újraindításának. Ha több példányunk van, akkor mindegyiket kézzel tudjuk így törölni. Ez persze csak akkor működik, ha a pod felett van egy controller, ami szükség szerint létrehozza az új podokat.

## További olvasnivaló

- [The twelve-factor app](https://12factor.net/): modern webszolgáltatások fejlesztése (nem csak Kubernetes)
- Bilgin Ibryam, Roland Huss - Kubernetes Patterns: Reusable Elements for Designing Cloud Native Applications, O'Reilly, 2019
