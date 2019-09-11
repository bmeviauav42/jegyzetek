# Docker alapok

## Labor célja

A labor célja megismerni a Docker konténerek használatának alapjait, és a leggyakrabban használt Docker CLI parancsokat.

## Előkövetelmények

- Docker Desktop
  - A laboron Windows platformot használunk, azonban a feladatok többnyire Linuxon és Mac-en is megoldhatóak.
- Alap Linux parancsok ismerete. Érdemes átnézni pl.:
  - <http://bmeaut.github.io/snippets/snippets/0700_LinuxBev/>
  - <https://maker.pro/linux/tutorial/basic-linux-commands-for-beginners>
  - <https://www.pcsuggest.com/basic-linux-commands/>

## Feladatok

### _Docker Desktop_

- Indítsuk el a _Docker Desktop_-ot.
- Keressük meg a tálcán az ikont.
- Nézzük meg a beállítási lehetőségeit.
- Keressük meg a _Switch to Linux containers / Switch to Windows containers_ parancsokat.
- Csak Windows-on: keressük meg a háttérben futó Hyper-V VM-et.

### Docker _hello world_

- Nyissunk egy _PowerShell_ konzolt, és adjuk ki a következő parancsokat.
- `docker --version`
- `docker run hello-world`
  - _hello-word_ egy image neve: <https://hub.docker.com/_/hello-world>
  - Image letöltődik, elindul, lefut a benne leírt program.

### Konténer futtatása interaktív módon

- Add ki a következő parancsot: `docker run -it ubuntu`
  - Nézzük meg a fájlrendszert: `ls`
  - Lépjünk ki az interaktív shell-ből: `exit`
- Konténer terminált, mert a bash folyamat megállt az `exit` hatására. **Konténer addig fut, amíg a benne levő alkalmazás fut.**
- Leállt konténer még meg van, tartalma még nem veszett el: `docker ps -a`. Nézzük meg a parancs eredményét. Keressük meg a konténerek id-ját.
- Távolítsuk el a két konténert, amit mi indítottunk: `docker rmi <id1> <id2>`

### Docker CLI parancsok struktúrája

- Adjuk ki a `docker` parancsot a help-hez.
- Adminisztratív parancsok: _mivel mit_, pl. `docker image ls`
- Kezelő parancsok: _parancs argumentumok_, pl. `docker rmi <id>`
- Gyakran használtak:
  - Konténerek kezelése
    - `docker container ls [-all]` avagy `docker ps [-a]`
    - `docker run [opciók] <image>`
    - `docker stop <id>`
    - `docker rm <id>`
  - Image-ek kezelése
    - `docker pull <image>`
    - `docker image ls` avagy `docker images`
    - `docker rmi <image>`
    - `docker tag <id> <tag>`
- Konkrét parancshoz segítség: `docker <parancs> --help`
- Tipp: minden konténer eltávolítása `docker rm -f $(docker ps -aq)` (PowerShell)

### _Volume_ csatolása (_bind mount_)

- Hozzunk létre egy munkakönyvtárat, pl. `c:\work\<neptun>\foo`
- Indítsunk el egy konténert úgy, hogy ezt a könyvtárat felcsatoljuk
  - `docker run -it --rm -v c:\work\<neptun>\foo:/bar ubuntu`
  - szintaktika: helyi teljes elérési útvonal _kettőspont_ konténeren belüli teljes elérési útvonal
  - Konténeren belül: `ls`, látjuk a /bar könyvtárat
  - Írjunk bele: `echo "hello" > /bar/a.txt`
  - `exit`
- Nézzük meg a munkakönyvtárunkat.
- Tipp: `--rm` opció törli a konténert a végén; pl. teszteléshez, próbálkozáshoz, mint most.

### Port mappelés

- Indítsunk el egy _nginx_ webszervert: `docker run -d -p 8085:80 nginx`
  - `-d` opció: háttérben fut, a konzolt "visszakaptunk", amint elindult a konténer, és kiírja az image id-t
  - `-p` hely port _kettőspont_ konténeren belüli port
- Nyissuk meg böngészőben ezt a címet: <http://localhost:8085>
- Nézzük meg a konténer logjait: `docker logs <id>`
- Állítsuk le a konténert: `docker stop <id>`

### _Docker registry_

- Korábban használt parancs: `docker run ubuntu` Az _ubuntu_ az image neve. Ez egy un. registry-ből jön.
- Tipikus registry: <https://hub.docker.com>
  - Tipikusan open-source szoftverek image-ei, és az általunk is használt alap image-ek.
  - Vannak továbbiak is (Azure, Google, stb.)
  - Analógia: NPM csomagkezelő, NuGet.org
- Image neve valójában nem `ubuntu`, hanem `index.docker.io/ubuntu:latest`
  - `index.docker.io` registry szerver elérési útvonala
  - `ubuntu` image neve (lehet többszintű is)
  - `:latest` tag neve
- Hasonló példa: `mcr.microsoft.com/dotnet/core/runtime:2.1`
- Két fajta registry: publikus (pl. Docker Hub) és privát
  - Privát registry esetén: `docker login <url>` és `docker logout <url>`
- Letöltés a registry-ből: `docker pull mcr.microsoft.com/dotnet/core/runtime:2.1`
  - Ugyan a _run_ parancs is letölti, de csak akkor, ha még nem létezik. Nem ellenőrzi viszont, hogy nincs-e újabb image verzió publikálva. A _pull_ mindig frisset szed le.

### Készítsünk saját image-et.

Kövesd az alábbi lépéseket egy saját image elkészítéséhez. (Megjegyzés: ennél lesz jobb megoldás, következő órán.)

1. Indíts el egy nginx image-et: `docker run -d -p 8085:80 nginx` Jegyezd meg az image id-t, alább többször is használni fogjuk.
1. "Lépj be" a futó konténerbe egy interaktív bash shellben: `docker exec -it <id> /bin/bash`
1. A bash shellben lépj be az nginx által kiszolgált index html-t tartalmazó mappába: `cd /usr/share/nginx/html/`
1. Nézd meg a mappa tartalmát: `ls`
1. Írd felül az `index.html` tartalmát: `echo "hello from nginx" > index.html`
1. Lépj ki a shellből: `exit`
1. Ellenőrizd meg, hogy a konténer még fut: `docker ps`
1. Nyisd meg böngészőből a <http://localhost:8085> címet, ellenőrizd, hogy megjelenik a saját tartalom
1. Készíts egy pillanatmentést a konténer jelenlegi állapotáról: `docker commit <id>`
1. Az előbbi parancs készített egy image-et, aminek kiírta a hash-ét. Ellenőrizd, hogy tényleg létezik-e ez az image: `docker images`
1. Állítsd le a háttérben futó konténert: `docker stop <id>`
1. Taggeld meg az image-et: `docker tag <imageid> mynginx` (itt már az image id-ja kell)
1. Indíts el egy új konténert az előbb létrehozott saját image-ből: `docker run -it --rm -p 8086:80 mynginx` (A portszám szándékosan más, hogy biztosan legyünk benne, nem a korábban futóhoz csatlakozunk - ha mégse állítottuk volna azt le.)
1. Nyisd meg böngészőből a <http://localhost:8086> címet. Látható, hogy ez a módosított tartalmat jeleníti meg. Tehát `mynginx` néven létrehoztunk egy saját image-et.

### Takarítás

Fejlesztés közben sok ideiglenes image keletkezik, és konténereket hagyunk hátra. Add ki a következő parancsot a nem futó konténerek törléséhez és az ideiglenes (címke nélküli) image-ek törléséhez: `docker system prune`

## További olvasnivaló

- Digest-ek: <https://engineering.remind.com/docker-image-digests/>
- Image layerek: <https://docs.docker.com/v17.09/engine/userguide/storagedriver/imagesandcontainers/>
