# Házi feladat

A házi feladat otthon, önállóan elkészítendő mikroszolgáltatások architektúrára épülő és konténertechnológiát használó szoftverrendszer elkészítése és **működőképes állapotban való bemutatása**.

A rendszer egyetlen alkalmazást kell megvalósítson, ahol az alkalmazás funkciók szerint több szolgáltatásra van darabolva. Az alkalmazás nem feltétlenül kell felhasználói felülettel rendelkezzen, de enélkül is tudni kell demonstrálni a működését (pl. REST API-n keresztül).

Az elkészített rendszer egyes képességeire az alábbiak szerint pontok kaphatóak. Részpontszám csak különleges esetben kapható. A végső jegy az összpontszámból adódik.

Meglévő (pl. Microsoft-os) demók, mintaalkalmazások (elemei) felhasználhatók, de ezt külön jelezni kell bemutatáskor. A nem jelzett, de átvett részletek plágiumnak számítanak. Az órai demókban vagy mintaalkalmazásban megvalósított funkciók átvételéért pont nem adható, de azok tovább átdolgozhatóak saját implementációnak.

A pontrendszer általatok is módosítható/bővíthető. Ezt Pull Request formájában egy megfelelő indoklással nyújthatjátok be, de a PR benyújtása nem jelenti annak az automatikus elfogadását, arról minden esetben a tárgy oktatói döntenek.

### Jegy számítás

- 0-24: nem teljesítette
- 25-29: elégséges
- 30-34: közepes
- 35-40: jó
- 41-: jeles

### Pontok az alábbiakért kaphatóak

#### Mikroszolgáltatatások architektúra: 5+3 pont

A rendszer több, független mikroszolgáltatásból épül fel. Ebbe beleértendő a frontend is, amennyiben azt a többitől független webszerver szolgálja ki, de az adatbázis szerver(ek) külön nem számítanak bele.

- Minimum 3 szolgáltatással: 5 pont
- Minimum 4 szolgáltatással: +3 pont

A pont a többnyire korrekt rendszer partícionálás esetében jár. (Egyszerű többrétegű architektúrára nem jár a pont, akkor se, ha a rétegek külön szolgáltatásokban kapnak helyet!)

#### Több implementációs nyelv használata: 5 pont

A backend szolgáltatások legalább 2 különböző nyelven készültek. (A frontend ebbe nem számít bele!)

#### Konténerekben történő futtatás: 7 pont

_Minden_ szolgáltatás konténerben fut, beleértve a használt adatbázis rendszereket is. Részpontszám kapható, ha nem minden szolgáltatás konténerizált.

#### Legalább két fajta adatbázis használata: 5 pont

Két eltérő technológiájú adatbázis használata perzisztenciára. Egyik lehet relációs is. (Redis csak cache-ként való használata ebbe nem számít bele.)

#### Redis alapú cache használata: 4 pont

Redis használata kifejezetten cache-elésre legalább 1 művelet esetén.

#### Http/gRPC alapú kommunikáció mikroszolgáltatások között: 4+4 pont

- Legalább egy olyan művelet, amelyben egy mikroszolgáltatás egy másikkal http alapon kommunikál: 4 pont
- Legalább egy olyan művelet, amelyben egy mikroszolgáltatás egy másikkal gRPC alapon kommunikál: 4 pont

#### Polly vagy hasonló technológia alkalmazása a mikroszolgáltatások közötti kommunikációra: 3 pont

Az előzővel együtt teljesíthető. A mikroszolgáltatások közötti kommunikáció során Backoff/Retry/Throttling/... minták alkalmazása.

#### Üzenetsor alapú kommunikáció mikroszolgáltatások között: 5 pont

Legalább egy olyan művelet, amelyben egy mikroszolgáltatás egy másikkal üzenetsor alapon kommunikál (mind a termelő, mind a fogyasztó oldalt beleértve). Kommunikációs platform lehet Redis, RabbitMQ, ill. bármely ismert message queue szolgáltatás.

#### API Gateway használata: 5+3 pont

- Traefik használata útvonalválasztásra: 5 pont
- Más API gateway használata: +3 pont

#### Forward authentikáció: 5 pont

Az előzőn felül teljesíthető. API Gateway esetén az authentikációt _forward authentication_ módszerrel az API Gateway biztosítja. (Nem szükséges teljes, valódi authentikáció, mint például JWT használata. Elég, ha a működés valamilyen módon demonstrálható.)

#### Futtatás docker-compose alapokon: 5 pont

Az _összes_ szolgáltatás egyetlen compose parancs segítségével elindítható.

#### Futtatás Kubernetes-ben: 10 pont

A bemutatás során az alkalmazás Kubernetes-ben fut. (Akár helyben, akár felhőben.)

#### Futtatás publikus felhőben Kubernetes-ben: 5 pont

Az előzőn felül teljesíthető. Az alkalmazás publikus felhőben hostolt Kubernetes platformon fut, érvényes (nem self-signed) HTTPS tanusítvánnyal rendelkezik, és publikus domain néven keresztül elérhető.

#### Helm chart Kubernetes-hez: 7 pont

A Kubernetes-be történő telepítést helm chart végzi. Szükséges demonstrálni a rendszer frissítését a chart segítségével.

#### Kubernetes-ben Job, CronJob, ConfigMap, Secret használata: 5 pont

Legalább egy Job, CronJob, ConfigMap, vagy Secret használata az alkalmazásban.

#### OpenTracing (pl. Jaeger) beüzemelése: 5 pont

Az alkalmazásban követhetőek a kérések pl. Jaeger használatával.

#### Continuous Integration beüzemelése: 5 pont

Az alkalmazás _teljes egésze_ CI rendszerben lefordul és konténerek készülnek belőle.

#### Web/mobil felhasználói felület: 5 pont

Az alkalmazás rendelkezik modern webes / mobilos felhasználói felülettel. Teljesség és igényesség függvényében részpontszám is kapható.

#### Eventually Consistent elosztott működés: 5+3 pont

- A mikroszolgáltatások között eventually consistent állapotkezelés megvalósítása a kapcsolódó tervezési mintákkal. (5 pont)
- Gondoljunk a műveletek idempotens mivoltára is a tervezés során. (3 pont)

#### Tervezési minták használata az implementáció során: 5 pont

Domain Driven Design (DDD), Command Query Segregation Principle (CQRS). Ezeken felül előzetes egyeztetéssel a tárgy oktatóival.

#### Visszacsatolás: 0-1 pont, összesen max. 3 pont

A véglegesített pontrendszer vagy tananyag javítása, bővítése, módosítása pull request-tel. Helyesírási hiba is lehet, de az oktatók döntenek, hogy pontot ér-e a módosítás. Többször is megszerezhető.
