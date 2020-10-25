# Elosztott nyomkövetés

## Előadás

[Elosztott nyomkövetés](https://edu.vik.bme.hu/mod/resource/view.php?id=22406)

## Cél

* OpenTracing és Jaeger alapú elosztott nyomkövetés eszköztárának megismerése
* OpenTracing és Jaeger alapú nyomkövetés alapok .NET Core alapú mikroszolgáltatások esetén

## Előkövetelmények

* Docker Desktop
* Visual Studio 2019
    * min v16.3
    * ASP.NET Core 3.1 SDK

## 1. Feladat - OpenTracing alapú nyomkövetés Jaeger környezetben

### Bevezető

A feladat keretében egy olyan mikroszolgáltatás alapú mintaalkalmazást vizsgálunk meg, mely az OpenTracing használatát illusztrálja, Jaeger alapú backenddel és megjelenítéssel. A feladat a [Take OpenTracing for a HotROD ride](<https://medium.com/@YuriShkuro/take-opentracing-for-a-hotrod-ride-f6e3141f7941>) blogbejegyzésen alapul, de néhány olyan Jaeger szolgáltatást is megvizsgálunk, melyet a cikk nem taglal.

A mintaalkalmazás neve "Hot R.O.D". Go nyelven íródott, kódja OpenTracing instrumentált. A forráskódja a Jaeger GitHub repository példák között található meg: <https://github.com/jaegertracing/jaeger/tree/master/examples/hotrod>. Itt találunk leírást arról, hogyan lehet a HotROD és Jaeger alkalmazásokat docker alapokon futtatni:

* **A Jeager futtatására** a `jaegertracing/all-in-one` docker image-et használjuk: ez valamennyi Jaeger backend komponenst tartalmaz (agent, collector, ingester, stb.), beleértve az adatmegjelenítő frontendet is. Ez az image ismerkedéshez, helyi környezetben teszteléshez ajánlott, production környezethez nem ajánlják. Bővebb leírást az image-ről többek között itt találhatunk: <https://www.jaegertracing.io/docs/getting-started>. A számtalan portból számunkra kettő érdekes, a 6831 (Jaeger agent, span-eket itt fogadja UDP-n a kliens könyvtáraktól) és a 16686 (Jaeger UI frontend). Megjegyzések:
    * A "jaegertracing/all-in-one" image alapértelmezésben egy in-memory tárolót használ, így újraindítást követően elvesznek a korábban rögzített trace/span-ek.
    * Leírás a további Jaeger image-ekről és konfigurációs lehetőségekről itt található: <https://www.jaegertracing.io/docs/deployment>
* A **HotROD futtatására** a `jaegertracing/example-hotrod` docker image használható. Ez az image egyben tartalmazza valamennyi HotROD szolgáltatás kódját, futattásakor minden szükséges szolgáltatás elindul ugyanabban a konténerben, csak különböző portokon.

A HotRod és a Jaeger konténerek egyszerre történő indítására docker-compose-t használunk:

1. Töltsük le a `docker-compose.yml`-t innen <https://github.com/jaegertracing/jaeger/blob/master/examples/hotrod/docker-compose.yml>, ide mentsük: `c:\work\<sajátnév>`
2. Nyissuk meg a fájlt VS Code-ban, tekintsük át a tartalmát
    * A `jaeger` szolgáltatás a 6831 (Jaeger agent spanfeltöltő) és a 16686 (Jaeger UI frontend) portokat mappeli.
    * A `hotrod` szolgáltatás számára környezeti változókban mondjuk meg, milyen host néven és milyen porton éri az a Jaeger agentet (span-ek felöltéséhez szükséges).
3. Indítsunk új command promptot, navigáljunk az első lépésben letöltött docker-compose.yml (c:\work\<sajátnév>) mappába, majd indítsuk a konténereket: `docker-compose up`. Ha nem indul a HotRod a 8080-as port ütközés miatt, akkor:
    * `docker-compose down`-nal állítsuk le a konténereket
    * A `docker-compose.yml`-ben mappeljünk más külső portra, pl.: 8090:8080
    * Indítsuk újra `docker-compose up`
4. A HotRod frontend a <http://localhost:8080>, a Jaeger frontend a <http://localhost:16686> címen érhető el.
5. Később a szolgáltatások leállítása a következő paranccsal lesz lehetséges (most még ne futtassuk): `docker-compose down`

### Ismerkedés a HotROD alkalmazással

A HotRod egy "Ride Sharing" alkalmazás. Uber-hez hasonló, személyek vagy egyéb kézbesítendő tárgyak járművekkel történő célba juttatásához. Egy Go alkalmazás, több szolgáltatásból áll. A docker image az "all" paraméterrel futtatja az alkalmazást, mely valamennyi szolgáltatását elindítja.

* Nézzünk rá a command promptban a docker-compose up által megjelenített logokra, látszik, hogy négy szolgáltatásból áll, melyek a 8080-8083 portokon érhetők el:
    * frontend
    * customer
    * driver
    * route

* Kapcsolódjuk a HotROD frontendhez böngészőből, ha még nem tettük meg (<http://localhost:8080>).
  
Ismerkedjünk meg a felület működésével:

* Egy gomb megnyomásával az alkalmazás egy adott ügyfélhez (aki egy adott helyen tartózkodik) keres egy a közelében tartózkodó járművet, pl. egy csomag felvételéhez vagy egy személy felvételéhez.
* Bal felső sarokban egy a JavaScript frontend által generált véletlen session azonosítót látunk (F5-re más lesz).
* Gombnyomásra új keresést indít, mely kapcsán egy új sorban a következő adatokat naplózza a felület:
    * A keresés során a kéréshez rendelt jármű rendszáma és várható érkezési ideje
    * Kérés azonosító (session id + sorszám)
    * Kiszolgálási idő, a JavaScript könyvtár méri
  
### Architektúra, szolgáltatásfüggőségek felderítése

Lépések:

* Kattintsunk valamelyik gombon a HotROD frontenden (ha még nem tettük meg)
* A Jaeger frontenden System Architecture menü, majd DAG tab kiválasztása

A Jaeger a hívások során begyűjtött trace/span adatok alapján a következőket jeleníti meg:

* a szolgáltatásokat
* a szolgáltatások közötti függőségeket
* a szolgáltatások közötti hívások számát
  
Az ábra az egérrel pan-elhető, illetve az egérgörgővel nagyítható (azon pontra vonatkozóan, ahol az egér éppen áll).

Force Directed Graph tab: hasonló az előzőhöz, az egérrel az adott csomópontra állva az adott csomópont függőségeit emeli ki.

Látjuk, hogy bár a HotROD egyetlen konténerben fut,  hat szolgáltatásból áll. Megjegyzés: a Mysql és a Redis nem valós szolgáltatások, a HotROD ezeket csak "szimulálja", ezek nem valódi adatkezelők.

### Trace-ek és spanek megjelenítése

A Jaeger frontenden válaszuk ki a "Search" menüt, itt van lehetőségünk a Jaeger által rögzített trace-ek listázására a megadott szűrési feltételeknek megfelelően. Lépések:

* A HotRod frontenden kattintsunk még egyszer-kétszer a gombokon, hogy több, mint egy trace-ünk legyen a rendszerben.
* A Jaeger frontend szűrőpaneljén a "Service" dropdown-ban válasszuk ki a "Frontend" elemet. Ha nincs a listában, akkor az F5 gombban frissítsük a böngésző oldalát.
* Kattintsunk a "Find traces" gombbon

A jobb oldali panel felső részén az x időtengely mentén a trace-ek jelennek meg (az y tengely a végrehajtási idő). Minden kliens oldali kéréshez külön trace születik. Itt is lehetőségünk van az adott körre kattintva a trace részleteinek megjelenítésére. A diagram alatt a trace-ek listás nézete jelenik meg, **minden trace egy külön sor**, számos hasznos információval (többek között a trace-ben levő span-ek száma, ugyanez szolgáltatásonkénti lebontásban, hibaesemények száma, stb.) Lehetőség van a tételek sorrendezésére (pl. végrehajtási idő szerint is hasznos lehet). Kattintsunk az egyik trace-re a részletes nézetének megjelenítéséhez.

Trace részletes nézet:

* A trace spanjeinek **egymásba ágyazási hierarchiáját** látjuk a vízszintes időtengely mentén.
* Egy span hossza arányos a végrehajtási idejével.
* Minden span külön sor, a legfelső sor a root span: a böngészőből kiadott JavaScript kérést fogadó `frontend` szolgáltatáshoz tartozik, a művelet neve `HTTP GET/dispatch`, először a `customer` szolgáltatást hívja, mely fogadja a kérést és a `mysql` szolgáltatásba hív tovább.
* Sokat segít az átláthatóságban a **színkódolás**: minden szolgáltatás adott színt kap, az összes spanje ugyanolyan színnel jelenik meg!

A spanek alapján látjuk, hogy az alkalmazás hogyan szolgál ki egy kérést:

1. A frontend szolgáltatás a külső HTTP GET kérést a `/dispatch` végpontjánál kapja meg.
2. A frontend szolgáltatás HTTP GET kérést küld az `customer` szolgáltatás  `customer` végpontjának (ha összecsukjuk a szülő span sorát, akkor ezt sor elején a `frontend -> customer` nyíl is mutatja).
3. Az `customer` végrehajt egy SQL SELECT utasítást a MySQL hívásával. Az eredmények visszakerülnek a frontend szolgáltatáshoz.
4. Ezután a frontend szolgáltatás RPC kérést indít (`Driver :: findNearest` művelet) a `driver` szolgáltatáshoz. Anélkül, hogy alaposabban belemélyednénk a nyomkövetési részletekbe, nem tudjuk megmondani, hogy mely RPC keretrendszert használják a kérések, de feltehetjük, hogy nem HTTP (valójában TChannel útján készül).
5. A `driver` szolgáltatás számos hívást kezdeményez a Redis felé. A hívások közül néhány piros felkiáltójel ikonnal van megjelölve, jelezve a hibákat.
6. Ezután a frontend szolgáltatás egy sor HTTP GET kérést intéz a `route` szolgáltatás `route` végpontjához.
7. Végül, a frontend szolgáltatás visszaadja az eredményt a külső hívónak.

!!! hint ""
    Az oldal tetején levő időablakban egérrel egy időszeletet kijelölve az adott időszeletre "nagyíthatunk" rá.

Span részletes nézet:

* Egy sorra kattintva az adott span, "kinyílik", magassága megnő, további adatokat mutatva a spanről.
* Nyissunk le az első spant: látjuk a hozzárendelt **Tag**-eket. Ezek kulcs-érték párok a spanhez rögzítve (bármilyen kulcs-érték pár lehet). Pl. különösen informatív  a `http.url`, `http.method`, és a `http.status_code`.
    * `span.kind`: ha a "hívott" oldalon vagyunk, akkor a "server" értékkel kerül rögzítésre. A hívó oldalon "client" értékkel rögzített.
    * `sampler.type`: "const" - jelzi, hogy minden művelet/span rögzítendő, a mintavételezés során nem dobandó el semmi.
* A mysql lekérdezéshez tartozó span (5. sor) tag-ben tartalmazza az SQL parancsot.
* Nyissunk le egy piros felkiálltójellel dekorált spant. A tag-ek között szerepel az `error = true`: ez a tag használandó hibák jelzésére.
* Bármilyen egyéni adatot hozzáfűzhetünk tag-ként a span-ekhez. Erre példa a redis FindDriverIDs esetén a `param.location = 728,326`, vagy Redis GetDriver esetén a `param.driverID=T707027C`.
A "szabványos" span tag-ekről, illetve a rövidesen tárgyalásra kerülő log "szabványos" mezőiről itt találunk leírást: <https://github.com/opentracing/specification/blob/master/semantic_conventions.md#span-tags-table> (nyissuk meg az oldalt és nézzünk rá a táblázatra). Az OpenTracing nem köti ki a tag neveket, de mindenképpen célszerű a táblázat ajánlásait követni: ha nem tesszük, a trace/span megjelenítő eszközök nem tudnak szemantikát társítani hozzá (pl. error esetén speciális megjelentés).

!!! note ""
    Megjegyzés: ha megnézzük, a mysql és redis spanek kliens odaliak (span.kind = client), ezek nem az adattárolóból erednek. A demóalkalmazásban az adatkezelők szimuláltak, de jellemzően a valós adattárolók sem instrumentáltak: ez esetben tárolóhoz való hozzáférést kliens oldalon vegyük körbe egy új spannel, és ehhez csapjuk hozzá az informatív tag-eket (pl. SQL parancs, Redis művelet  és paraméterek, stb.).

A spanekhez **Log** bejegyzések is tartozhatnak. Keressünk meg párat:

* A span részletes nézetében a Logs szekció alatt találhatók.
* A span csíkján is megjelennek a logbejegyzések vékony függőleges vonallal, az egeret fölé húzva tooltipben részletes információt kapunk az adott logbejegyzésről.
* A Log bejegyzéseknek van időbélyege, plusz tetszőleges, debugolást segítő kulcs-érték párok tartozhatnak hozzá.  A "szabványosakat" itt tekinthetjük meg: <https://github.com/opentracing/specification/blob/master/semantic_conventions.md#log-fields-table> (nézzünk rá a táblázatra). A legfontosabb az `event` kulcs.
* Nézzük meg az egyik hibás redis GetDriver művelet logját. Látjuk, hogy redis timeout történt, a `driver_id` is naplózásra került.

### Kontextusba helyezett logok

Egész jól látjuk, hogyan épül fel az alkalmazás. További kérdésekre keressük a választ, pl.: miért hívja a frontend a customer szolgáltatás /customer végpontját?
Próbálhatnánk a szolgáltatások logjaiból megtudni:

* Ez sokszor nagyon nehéz
* Ha sok felhasználói kérés kiszolgálása történik egyszerre párhuzamosan, szinte lehetetlen kihámozni, mi tartozik egy adott felhasználói kéréshez.

Helyette nézzük a trace rendszer által begyűjtött (span-ekhez kapcsolódó) logbejegyzéseket. Nézzük meg a `HTTP GET /dispatch` spanhez kapcsolódó logokat (18 bejegyzés lesz). **A többi kéréstől izoláltan, jól áttekinthető módon, az adott trace és span kontextusába helyezve látjuk a logbejegyzéseket!**

Megjegyzés: Valami furcsa okból kifolyóan a mintaalkalmazás több helyen is elég szegényesen naplózza az információkat. Pl. a `Found customer` nem írja ki, milyen adatok kerültek lekérdezésre (customer koordináták): pedig itt pont segítené a megértést, mert ezen koordináták által meghatározott ponthoz képest keresi a közelben levő vezetőket a FindNearestDriver műveletben.

### Span Tasgs vs. Logs

Mikor használjunk tageket, és mikor logokat? Alapelv:

* A tag olyan információ rögzítésére való, mely a span egészéhez tartozik.
* A log időbélyeggel rendelkező események rögzítésére való.

### Késleltetések okainak felderítése

Vizsgáljuk meg az alkalmazás teljesítmény karakterisztikáit. Trace-ek alapán ezt látjuk:

1. Az ügyféladatok lekérdezése (`customer` szolgáltatás) a kritikus útvonalon van, mert ez adja vissza az ügyfél koordinátáit.
2. A `driver` szolgáltatás lekérdezi az ügyfél közelében levő 10 járművezetőt, majd egyesével SORBAN EGYMÁS UTÁN lekérdezi ezek adatait a Redistől: ezt mutatja a `redis GetDriver` lépcsőzetes mintája.
3. Ezt követően a 10 útvonalszámítás (`route` szolgáltatás) művelete nem is szekvenciális és nem is teljesen párhuzamos. Azt látjuk, hogy maximum három kérés tud párhuzamosan futni, és amint ezekből egy véget ér, akkor indul a következő. Ez arra utal, hogy itt egy hármas végrehajtó pool futtatja a műveleteket.

Vizsgáljuk meg, mi történik, ha számos párhuzamos kérést futtatunk:

1. A HotROD alkalmazásban gyorsan kattintsunk kb. 20-szor valamelyik gombon.
2. A HotROD alkalmazás felületén is jól látható, hogy a kérések kiszolgálása belassul, a latency 800 ms helyett 2000 ms környéke lesz.
3. A Jaeger alkalmazásban navigáljunk vissza a nyitóoldalra, frissítsük a trace listát (Find traces gomb), keressük ki és nyissuk meg az egyik leghosszabb lefutású trace-t: a trace listában ehhez tartozik a leghosszabb ciánkék színű sáv.
4. A trace-t másként is kikereshetjük: a HotROD oldal logjában látszik, hogy mely járművezetőt rendelte a rendszer a legnagyobb duration paraméterű sornál a kéréshez (T716217C vagy egy hasonló formátumú sztring). A Jaeger nyitóoldalán a szűrőpaneleben a "Tags" mezőbe írjuk be ezt: "driver=T716217C" és aktiváljuk a szűrést. Azért találja meg a trace-t, mert az egyik span egyik logjában szerepel a driver=T716217C kulcs-érték pár. (Ha meg akarjuk nézni: a legelső spant lenyitva a spanhez tartozó utolsó log tartalmazza).
5. Vizsgáljuk meg a trace-t: a legszembeötlőbb különbség a korábbi, gyors trace-ekhez képest, a mysql kérés lassú kiszolgálása. Próbáljuk meg kitalálni, miért:
6. Nyissuk le a `mysql SQL SELECT` spant. Két logbejegyzés tartozik hozzá.
7. Ezekből egyértelműen kiderül, hogy az idő nagy részét lock-ra várakozással töltöte a kérés.
8. Az első log-ból még az is kiderül, mely kérések tartották fel (Waiting for lock behind 4 transactions"), ezeket meg is nevesíti, pl. : `[8221-10 8221-11 8221-12 8221-13]`. Ezek a kérés azonosítók a HotROD frontendről jönnek, nézzük meg őket a felületen!
9. Ami itt igazán izgalmas: honnan tudja a mysql "kliens", hogy mik a kérés azonosítók, hogy jut el hozzá? Paraméterben NEM passzolja végig a rendszer a hívási láncon (pl. a HTTP GET customer kérések paraméterében sem szerepel, megnézhetjük a span-eket).
10. A "varázslat" mögött az OpenTracing **baggage** koncepciója és mechanizmusa áll.
       * Maga az elosztott trace azért tud megvalósulni, mert a OpenTracing instrumentálás gondoskodik arról, hogy a kérésekhez kapcsolódó bizonyos metaadatok szálak, folyamatok és számítógépek között propagálódjanak és minden a kérés kiszolgálásában részt vevő szereplőhöz eljussanak. Ilyen a trace id és a span id is. Egy másik ilyen metainformáció a baggage. Ez egy általános kulcs-érték pár tároló. 
       * A példánkban a JavaScript UI beteszi a kérés azonosítót a baggagebe, és ez a kérés kiszolgálása során minden szereplőhöz eljut, anélkül, hogy explicit paraméterekben kellene kezelni. Az OpenTracing instrumentáció gondoskodik róla, pl. HTTP fejlécbe teszi a HTTP kérések kiszolgálása során). Nagyon hatékony és hasznos eszköz!
       * A példánkban lehetővé teszi, hogy a kérésazonosító alapján kikeressük és analizáljuk azokat a trace-eket, melyek feltartják a kérésünk kiszolgálását. A való életben is gyakori probléma: egy ügyfél kérés feltart/belassít számos másikat. Lehetőségünk van ezen kérések megtalálására és analizálására.
11. A HotRod példában nincs igazi Mysql adatbázis, csak szimulálja a rendszer. Itt látható a mesterségesen szigorított zárolás (pl. egy rosszul konfigurált DB connection poolt szimulálva): <https://github.com/jaegertracing/jaeger/blob/master/examples/hotrod/services/customer/database.go>, ezen belül az `if !config.MySQLMutexDisabled` sor környékét nézzük.
12. Szimuláljuk a detektált probléma javítását. A forráskódhoz nem fogunk nyúlni, a HotROD alkalmazást kell speciális command line argumentummal futtatni ahhoz, hogy kikapcsoljuk a mesterségesen indukált szigorú zárolást, illetve ezen felül a mesterséges késleltetés mértékét is csökkenteni fogjuk:
    1. A docker-compose.yml-ben a hotrod szolgáltatás command line paramétereit bővítsük a "-M" kapcsolóval, ehhez egy sort kell módosítani, a `command:` után [] között ez kell álljon: `"-M", "-D", "100ms", "all"`
    2. Mentsük el a fájlt.
    3. Tegyük fel, hogy production környezetben vagyunk, csak a módosult szolgáltatást (esetünkben HotROD) kívánjuk újraindítani, a többit (esetünkben Jaeger) nem. Így nem adjuk ki a `docker compose down` parancsot!
    4. Egy új command line ablakban navigáljunk el a docker-compose.yml-t tartalmazó mappába (`c:\work\<sajátnév>`). Futtassuk a következő parancsot: `docker-compose up -d`. A '-d háttérben futtat, az up parancs pedig csak a módosult konténereket állítja le és indítja újra. A kimeneten ez jól követhető. Közben a korábbi, a konténerekre még mindig rácsatolt command promptunkban látszik, hogy újraindult a HotROD szolgáltatás, 'fix: disabling db connection mutex' üzemmódban.
13. Teszteljük a javított megoldást: generáljunk kb. 20 párhuzamos kérést, ellenőrizzük a duration-t: egyik sem megy 2000ms környékére (kb. 900 ms környékére kúszik csak fel), és a trace-ekben látszódik, hogy a mysql span-ek hossza 100 ms körül marad.

#### További teljesítmény optimalizáció

1. Azt látjuk, hogy míg korábban a route szolgáltatás hívások közül három futott párhuzamosan, most már általában csak egy fut egyszerre. Sőt, olyan időszakok is vannak, amikor egy sem fut. Ebből arra következtetünk, hogy a végrehajtó goroutine más goroutine-okkal verseng közös erőforrásért (vagyis közös a végrehajtó pool). A probléma helye a frontend szolgáltatásban valószínű, mivel a route spanek a frontend spanek gyerekei, így azt valószínűsítjük, hogy a route műveleteket a frontend művelete hívja. Ha van időnk, nézzünk rá a frontend kódjára:
   1. <https://github.com/jaegertracing/jaeger/blob/master/examples/hotrod/services/frontend/best_eta.go>, itt a getRoutes művelet az érdekes.
   2. Azt látjuk, hogy egy poolt használ az útvonalak megtervezéséhez (a legkorábbi várható érkezési idejű útvonalat keressük).
   3. Azt is látjuk, hogy ha a pool mérete kellően nagy, akár minden FindRoute művelet futhatna párhuzamosan.
   4. Arra következtetünk, hogy a pool mérete nem kellően nagy (a korábbi tesztjeink alapján a pool méretét 3-asnak véljük).
2. Növeljük meg a pool méretét 100-ra:
   1. A docker-compose.yml fájlban változtassunk a command line paramétereken: `"-M", "-D", "100ms", "-W", "100", "all"` legyen az új halmaz, mentsük el a változtatást
   2. Indítsuk újra a hotrod szolgáltatást: `docker-compose up -d`
3. Teszteljük a javított megoldást: generáljunk kb. 20 párhuzamos kérést és ellenőrizzük az egyik friss trace-t, a route spanek párhuzamosan kell fussanak.

A driver szolgáltatáson is lehetne optimalizálni, a Redis `GetDriver` lekérdezések szekvenciálisak, ezzel most nem foglalkozunk.

### Erőforráshasználat attributált mérése baggage segítségével

Gyakran merül fel arra **üzleti igény, hogy az erőforrás (pl. CPU) használatot valamilyen magasabb szintű paraméter alapján mérjük és attributáljuk**. A példánkban a route szolgáltatásban az útvonalkeresés relatíve CPU intenzív művelet, jó lenne mérni, hogy **ügyfelenként** (customer) mennyi CPU időt használ. Ugyanakkor a route szolgáltatás a függőségi gráfban mélyen van, itt már nincs információ az ügyfélről, akinek kapcsán az útvonalkeresés történik. Csak a mérés érdekében egy explicit customer id paramétert bevezetni a route szolgáltatás API-ján rossz tervezői döntés lenne. Itt lép képbe a tracing, illetve annak **baggage** mechanizmusa. Egy adott trace kontextusában már tudjuk, mely ügyfél számára történik az útvonalkeresés. A baggage pedig lehetőséget nyújt arra, hogy a **szolgáltatások kódjának módosítása nélkül** transzparens módon továbbítsunk a trace-hez kapcsolódó metaadatokat a szolgáltatások között. Esetünkban ez a customer id lesz, de ilyen lehet egy kérés/session/felhasználó azonosító is.

A fenti koncepció demonstrálására a `route` szolgáltatás olyan kódot tartalmaz, mely méri az útvonalszámítás idejét és a Go `expvar` metrika "kezelő" standard library package segítségével összegyűjti, nyilvántartja és lekérdezhetővé teszi azt. A kód itt található: <https://github.com/jaegertracing/jaeger/blob/master/examples/hotrod/services/route/stats.go>. Alapelve:

* A `routeCalcByCustomer` és `routeCalcBySession` változók egy-egy `map` típusú metrikát definiálnak, adott kulcsokkal.
* A `stats` egy struktúra tömb, két elemű. A struktúra objektumokban egy expvar metrika és egy baggage kulcs található.
* Az `updateCalcStats` lekérdezi az aktuális spant, egy ciklusban végigmegy a `stats`-ban tárolt struktúra elemeken: minden elemre lekérdezi a baggage-ből a struktúrában tárolt kulcs alapján a baggage elemet ("customer" és "session"), majd ez és a futási idő alapján frissíti a struktúrában tárolt metrikát.

Az expvar metrikák lekérdezhetők a futó szolgáltatástól a 8083-as porton:

1. A `docker-compose.yml` fájlban a hotrod szolgáltatásnál publikáljuk a 8083-os portot (vegyük fel a "8083:8083"-at a "ports:" alá), mentsük el, a `docker-compose up -d`-vel frissítsük a futó konténert.
2. A HotROD frontendden generáljunk jópár kérést: az első gombon kattintsunk sokat, a másodikon párat, a többin ne kattintsunk.
3. Böngészőből kapcsolódjunk a hotrod konténer expvar szolgáltatásához és kérdezzük le a metrikákat: <http://localhost:8083/debug/vars>
4. Keressünk rá szöveg szerint a `route.calc.by.customer.sec` és az alatta levő `route.calc.by.session.sec` kulcsokra és tekintsük meg a metrikák értékét. Az alábbihoz hasonló kimenetet kapunk:

```
"route.calc.by.customer.sec": {"Amazing Coffee Roasters": 1.0790000000000002, "Japanese Desserts": 1.5280000000000002, "Rachel's Floral Designs": 19.54500000000001, "Trom Chocolatier": 0.46199999999999997},
"route.calc.by.session.sec": {"8221": 22.61399999999999},
```

### Kódinstrumentálás

Felmerül a kérdés, mennyit kellett a kód instrumentálásán dolgozni a HotROD alkalmazás fejlesztése során. Meglepően keveset:

* Be kell konfigurálni, össze kell kötni Jaegerrel.
* Az intrumentálás a kommunikációt végző librarykben van: a Go hoz rendelkezésre álló REST és TChannel kommunikációt támogató open source könyvtárak instrumentáltak, ezzel nekünk nem kell foglalkozni.
* A kódban a Mysql és Redis szimuláció esetén található explicit instrumentálás.

## Feladat 2 - OpenTracing és Jaeger alapú nyomkövetés .NET Core alapú alkalmazás esetén

Kiinduló projekt és megoldás: <https://github.com/bmeviauav42/nyomkovetes>

A feladat keretében .NET Core 3.1 környezetben dolgozunk, de az OpenTracing és Jaeger .NET kliens könyvtárak korábbi .NET Core verziókkal is használhatók, nincs bennük semmi .NET Core 3.1 specifikus.

Vannak kliens könyvtárak, melyek azt várják, hogy a `jaeger-agent` a helyi gépen (pl. ugyanabban a docker containerben) fut, az agenttel UDP protokollon történik a kommunikáció. A tapasztalatok szerint számos kliens könyvtár tud más gépen/containerben futó agenttel kommunikálni, konfigurálható a kliensben az agent címe (host és a port). Ez lehetővé teszi a `jaegertracing/all-in-one` kényelmes használatát, a szolgáltatásokat futtató konténetben nem kell jaeger agentet telepíteni. Éles környezetben ez a megoldás korlátozottan ajánlott: az UDP megbízhatóan működik localhost esetén, de egy valódi hálózaton veszhetnek el csomagok. A legtöbb kliens könyvtár azt is támogatja, hogy az agent kihagyásával, közvetlen a collectornak történjen az adatküldés, akár HTTP protokollon.

A kommunikációról szóló gyakorlat némiképpen továbbfejlesztett Order+Customer kiindulási példáját kötjük össze Jaegerrel és instrumentáljuk OpenTracing alapokon. Miben más a kiinduló kód, mint a korábbi kommunikációs órán szereplő kiinduló gyakorlat?

* Repository mintát használ.
* A Catalog szolgáltatás Sqlite adatbázisban tárolja az adatokat (in-memory üzemmódra konfiguráltan), induláskori seed-et, vagyis tesztadat generálást követően. Ez éles környezetben nem használható, a célja mindössze a nyomkövetés demonstrálása. A választás azért esett az Sqlite-ra az Entity Framework In Memory Database-zel szemben, mert részletesebb nyomkövetési információt szolgáltat.

### Előkészítés, konfiguráció

Kiinduló lépések:

* GitHub-ról klónozzuk ki a kiinduló solution-t:
    * Hozzunk létre egy `tracing` mappát a `c:\work\<sajátnév>` munkakönyvtárunkban és indítsunk egy command promptot innen, futtassuk az alábbi parancsot:
`git clone https://github.com/bmeviauav42/nyomkovetes`

!!! warning "Ékezetes elérési út"
    Fontos, hogy ne legyen az elérési útban speciális (és ékezetes) karakter, különben nem tud a VS a docker-compose-hoz Debuggerrel csatlakozni.

* Indítsuk el VS alatt a szolgáltatásokat
* A böngészőben hibaoldal jelenik meg. Frissítsük (ha kell többször is), a hiba eltűnik. Az oka: az OrderService hívja a CatalogService-t, de az először még nem állt fel teljesen, az Sqlite adatbázis seed-elése időt igényel.

Ha kevés az idő, nézhetjük a feladat megoldását: ez a `megoldas/1-konfig-es-beepitett-instrumentalas` ágon található, az ág checkout lépései:

```bash
git fetch
git checkout megoldas/1-konfig-es-beepitett-instrumentalas
```

VS alatt startup projektnek állítsuk be a docker-compose projektet.

#### Projekt referenciák felvétele

A solutionünk több projektből/szolgáltatásból áll. Mindegyiket hasonló módon kell OpenTracing és Jaeger vonatkozásában inicializálni, illetve ugyanarra a Jaeger "kiszolgálóra" kell rákötni. Ezt a kódot ne copy-paste-tel szaporítsuk, hanem tegyük ki egy külön megosztott könyvtárba. Ez a kiinduló megoldásban elő van készítve, már van egy ilyen projekt (Msa.Comm.Lab.Shared), csak a projekt referenciák nincsenek felvéve. Tegyük ezt meg minden projektnél, ahol a nyomkövetést használni akarjuk:

* `Msa.Comm.Lab.Services.Catalog` -> `Msa.Comm.Lab.Shared` projekt referencia felvétele
* `Msa.Comm.Lab.Services.Order` -> `Msa.Comm.Lab.Shared` projekt referencia felvétele

### OpenTracing és Jaeger kliens könyvtárak

.NET Core környezetben az alábbi NuGet csomagok használatára van szükség:

* **"OpenTracing"**
    * OpenTracing API a kód instrumentálásához.
    * <https://github.com/opentracing/opentracing-csharp>
* **"OpenTracing.Contrib.NetCore"**
    * <https://github.com/opentracing-contrib/csharp-netcore>
    * Nem kötelező, de .NET Core környezetben javasolt: anélkül tudunk a segítségével pl. REST hívásokat trace-elni, hogy a kódunkat instrumentálni kellene.
    * Beépül az ASP.NET-be, Entity Framework-be, bizonyos .NET Core BCL típusokba,melyek közül a legfontosabb a HttpClient.
    * Minden .NET könyvtárat, keretrendszert, stb.-t  támogat, ami a  .NET DiagnosticSource-t használja (minden Activity-hez spant készít és span log-ot az egyéb eseményekhez).
    * Beregisztrálja magát a Microsoft.Extensions.Logging rendszerbe, és minden logba írás esetén span log-ot készít, de csak akkor, ha van aktív span.
    * Függőségként felteszi az OpenTracing package-et is!
* **"Jaeger"**
    * <https://github.com/jaegertracing/jaeger-client-csharp>
    * Jaeger .NET kliens könyvtár

Az `Msa.Comm.Lab.Shared` könyvtárba már fel vannak véve ezek a NuGet függőségek (az `OpenTracing` csak közvetve, az `OpenTracing.Contrib.NetCore` alatt), így nekünk nem kell megtennünk.
A szolgáltatás projektekben nem fogunk első körben közvetlen OpenTracing instrumentálást végezni, csak használjuk a `Msa.Comm.Lab.Shared` könyvtár egyetelen osztályát/műveletét a szolgáltatások konfigurálásakor: vegyük fel a `Msa.Comm.Lab.Services.Catalog` projekt `Startup.ConfigureServices` műveletének elejére:

```csharp
// Registers and starts Jaeger (see Shared.JaegerServiceCollectionExtensions)
// Also registers OpenTracing
services.AddJaeger(currentEnvironment);
```

A szolgáltatás projektekbe így (egyelőre legalábbis) egyetlen OpenTracing/NuGet package-et sem vettünk/veszünk fel, csak közvetett függés van!

Tekintsük át a `JaegerServiceCollectionExtensions.AddJaeger` műveletet, a legfontosabbak:

* A DI konténerbe egy `ITrace`r implementációt regisztrálunk be singletonként.
* Ez esetünkben egy Jaeger tracer lesz, amit a példában környezeti változók alapján inicializálunk (docker környezetben praktikus megközelítés): `Jaeger.Configuration.FromEnv(loggerFactory)`
* A környezeti változók közül a `JAEGER_SERVICE_NAME`, `JAEGER_AGENT_HOST`, `JAEGER_AGENT_PORT` és `JAEGER_SAMPLER_TYPE` a fontosabbak.
* A `GlobalTracer` egy klasszikus sigleton hozzáférést biztosít bárhol a kódban a tracer-hez, ezt is állítsuk be: `GlobalTracer.Register(tracer)`. Ez azon kód számára fontos, mely nem tud DI alapokon működni.
* A végén fontos az OpenTracing szolgáltatások regisztrálása is, eddig csak Jaegerrel foglalkoztunk: `services.AddOpenTracing();`

#### Jaeger szolgáltatás beépítése docker-compose.yml-be

Egészítsük ki a `docker-compose.yml` fájlt (időhiány esetén másoljuk be az egészet):

* jaeger szolgáltatás felvétele
* a már meglévő két szolgáltatásnál depends_on alatt `jaeger` megadása
* környezeti változók felvétele (ez esetünkben nem kötelező, mert ha nem adjuk meg, az `Msa.Comm.Lab.Shared` által beállított alapértékek megfelelők)

```yaml hl_lines="9-14 20-23 26 27-31"
version: '3.7'

services:
  msa.comm.lab.services.catalog:
    image: ${DOCKER_REGISTRY-}msacommlabservicescatalog
    build:
      context: .
      dockerfile: Msa.Comm.Lab.Services.Catalog/Dockerfile
    environment:
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
      - JAEGER_SAMPLER_TYPE=const
    depends_on:
      - jaeger
  msa.comm.lab.services.order:
    image: ${DOCKER_REGISTRY-}msacommlabservicesorder
    build:
      context: .
      dockerfile: Msa.Comm.Lab.Services.Order/Dockerfile
    environment:
      - JAEGER_AGENT_HOST=jaeger
      - JAEGER_AGENT_PORT=6831
      - JAEGER_SAMPLER_TYPE=const
    depends_on:
      - msa.comm.lab.services.catalog
      - jaeger
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # For Jaeger web UI
```

Teszteljük a működést:

* Indítsuk el a szolgáltatásokat VS alatt, frissítsük párszor a böngészőablakot
* Jelenítsük meg Jaeger UI-t: <http://localhost:16686/>
* A szűrőpanelen válasszuk ki a `Msa.Comm.Lab.Services.Order` szolgáltatást, frissítsük a megjelenített trace-eket (Find traces gomb), válasszunk ki jobboldalt egy hibával nem rendelkező trace-t és nyissuk meg.
* A részteles trace megjelenítőben látjuk span hierarchiát: Http hívások és DB ExecuteReader a mélyén. Az `Action Msa.Comm.Lab.Services.Catalog.Controllers.ProductController/Get` spanben számos log van, többek között EF-höz kapcsolódók is.
* **Mindezt úgy értük el, hogy semmiféle OpenTracing instrumentálást nem végeztünk a saját kódunkban**.

### Kód instrumentálás

Checkoutoljuk ki Git-ben kész megoldást, a `megoldas/2-kod-instrumentalas` ágon található:

```bash
git checkout megoldas/2-kod-instrumentalas
```

#### Egyszerű naplózás (##Instr_Log)

Itt még nem használunk OpenTracing specifikus instrumentálást, az `Microsoft.Extensions.Logging ILogger` segítségével naplózunk.

* Az `Order` szolgáltatás `TestController` osztályt nézzük
* DI-vel kap egy `ILogger<TestController>` objektumot. Ezt a `Microsoft.Extensions.Logging` beépítve támogatja.
* A `Get()` műveletben a `log.LogInformation` hívással naplózunk. Strukturált naplózást használunk, az első paraméterben a `{ProdCount}` definiál egy kulcsot, az értéke a count paraméter lesz. Ez nemcsak mint string, hanem egy kulcs-érték párként is megjelenik a naplózás során: a spanhez fűzött log-ban lesz egy ilyen kulcs-érték pár.
* Futtassuk az alkalmazást, generáljunk pár nem hibás hívást.
* Jaeger UI frontenden nyissunk meg egy nem hibás trace-t. A felső Find keresőbe írjuk be: ProdCount és keressünk rá. Azon spanek, ahol van találat, sárga színnel jelennek meg. Egy találatunk van. Nyissuk le a spant, és keressük ki szemmel a számunkra érdekes logbejegyzést. Itt látjuk, hogy a logon megjelenik a `ProdCount = 3` kulcs-érték pár, így lehet(ne) erre is értelmesen keresni: a nyitóoldalon a trace keresőben igen (pl. ProdCount=3 a  Tags mezőben), a trace részletes oldalon a span keresőben nem (itt, ha beírjuk, hogy ProdCount, akkor sárgával kiemeli az adott spant/logokat, lenyitogatva a böngésző Ctrl+F funkciójával lehet próbálkozni).
* Tipp: A span kereső egyelőre elég béna: ha valamit nem találunk, a jobb felső sarokban levő gombbal váltsunk JSON nézetre és keressünk abban szöveg szerint.

#### Saját span készítése, taggelés (##Instr_CreateSpan)

Saját spant készítünk. Itt már explicit OpenTracing API instrumentálást végzünk. Ehhez "logikailag" fel kellene vegyük az érintett projektben az **`OpenTracing`** NuGet package hivatkozást (`Jager` és `OpenTracing.Contrib.NetCore` nem kell, hiszen mi csak az API-t használjuk). Esetünkben nem kell megtenni, mert az `Msa.Comm.Lab.Shared` projektre van referencia, aminek már van (közvetett) `OpenTracing`  függése, a .NET Core 3.1 esetében ez már elég a megfelelő működéshez.

**Feladat**: a `Catalog` szolgáltatás `ProductController` osztályban a repository-hoz való hozzárés előtt cache-ben való keresést szimulálunk, ezt egy **új span** hatókörében trace-eljük.

* A `ProductController` függőséginjektálással kap egy `ITracer` objektumot
* A `Get(int id)`-ban levő kódot értelmezzük
    * Új span létrehozása, paraméterben műveletnév
    * Tag hozzáfűzése aktív spanhez
    * Log esemény felvétele aktív spanhez
* Futtassuk az alkalmazást, böngészőben egymás után kérjük le az egyes termékek adatait a `Test` szolgáltatás segítségével, a különböző kód ágak teszteléséhez:
    * <https://localhost:44385/api/test/1>, cache hibát generál
    * <https://localhost:44385/api/test/1>, nincs cache találat
    * <https://localhost:44385/api/test/1>, van cache találat
* A Jaeger UI segítségével vizsgáljuk meg a három trace-t

#### Baggage használata (##Instr_Baggage)

A `Catalog` szolgáltatás `ProductRepository` osztályban írjuk ki a felhasználónevet és kérést azonosítót Log-ba, melyet a példánkban az `Order` szolgáltatás generál.

A megoldás elve:

* Nem szennyezzük az API-t, nem vesszük fel explicit paraméterként
* Helyette **baggage**-ben továbbítjuk. Pontosabban rábízzuk az OpenTracing instrumentált HttpClient-re (az bepakolja http headerbe a baggage tartalmát).
* Lépések
    * `OrderService.TestController`-t nézzük meg, itt történik az aktív span baggage-ébe a kulcs-érték párok felvétele (string-string).
    * `CatalogService.ProductRepository`-t nézzük meg, itt olvassuk ki az aktív span baggage-éből az értékeket. Ezeket logban hozzáírjuk az aktív span-hez. (Nagyobb értelme lenne pl. valamilyen countert/merikát nyilvántartani ez alapján).
* Futtassuk (böngészőben `https://localhost:44385/api/test`)
* Jaeger UI-n a trace részletes nézetben az ablak tetején a keresőbe írjuk be: `Msa.Comm.Lab.Services.Catalog.Controllers.ProductController/Get`. A sárgával kiemelt spant nyissuk le, kb. a 10. logbejegyzés a `ProductRepository.GetProducts is executed`, látjuk a `username` és `requestid` kulcsokat és azok értékét.
