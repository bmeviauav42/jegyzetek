---
title: Prometheus monitorozás Kubernetes-ben
---

## Cél

A labor célja egy példán keresztül megismerni a Prometheus-alapú monitorozást Kubernetes környezetben.

## Előkövetelmények

- Kubernetes
    - Bármely felhő platform által biztosított klaszter
    - Linux platformon: [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube)
    - Windows platformon: Docker Desktop
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
    - A binárisa legyen elérhető PATH-on.
- [helm v3](https://helm.sh/docs/intro/install/)
    - A binárisa legyen elérhető PATH-on.

## Prometheusról, röviden

A [Prometheus](https://prometheus.io) egy metrikák gyűjtésére és lekérdezésére alkalmas adatbázis. Feladata egy rendszer (nem csak Kubernetes!) komponenseitől azok által publikált metrika adatok összegyűjtése. Saját grafikus felhasználói felülettel rendelkezik ugyan, de az csak nagyon alapvető funkciókra használható, ezért a [Grafana](https://grafana.com) dashboard rendszerrel együtt szokták használni.

A Prometheus architektúrája a hivatalos dokumentációból:

![Prometheus architektúra](https://prometheus.io/assets/architecture.png)

## Prometheus telepítése

A Prometheus csak egy metrikagyűjtő szerver. Ahhoz, hogy Kubernetes alatt monitorozni tudjuk, számos további komponensre van szükség. Ezek telepítését fogja össze több projekt is (amelyek zavarbaejtően hasonló nevűek, ezért vigyázzunk, ha rákeresünk). Ezek közül a [prometheus-operator helm chart](https://github.com/helm/charts/tree/master/stable/prometheus-operator) telepítése a legegyszerűbb számunka.

Helm segítségével regisztráljuk a chart repository-ját, majd telepítsük a chart-ot:

```bash
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo update
helm install prometheus stable/prometheus-operator
```

## Prometheus webfelület és Grafana

A Prometheus saját webfelülete nem túl felhasználóbarát, de célratörő eszköz. A Grafana a már jól összeállított rendszerünk metrikáinak grafikus megjelenítéséért felel.

1. A Prometheus saját webfelületét a prometheus-t futtató pod 9090-es porján érjük el. Különösebb bonyolítás nélkül továbbítsuk ezt egy helyi portra egy **új** terminálban kiadott parancssal: `kubectl port-forward $(kubectl get pods --selector "app=prometheus" --output=name) 9090:9090`

1. Nyissuk meg böngészőben a <http://localhost:9090> címen. Nézzük meg a megtalált _target_-eket és az _alert_-eket is.

1. Tegyük elérhetővé a Grafana-t is az előzőhöz hasonlóan egy **új** terminálban kiadott paranccsal: `kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=grafana" --output=name) 3000:3000`

1. A Grafana a <http://localhost:3000> címen lesz elérhető. Nyissuk meg és lépjünk be az `admin / prom-operator` felhasználónévvel és jelszóval, majd nézzünk meg pár kész dashboard-ot. Ezeket mind a prometheus-operator helm chart telepítette számunkra.

## Saját komponensek monitorozása

A _prometheus-operator_ chart előre konfigurál számos monitorozott célt, ezek többnyire a Kubernetes saját működését segítik láthatóvá tenni. Ha a saját alkalmazásunkat szeretnénk monitorozni, három lehetőségünk van.

1. Több eszköz támogatja a Prometheus monitorozást, csak engedélyezni kell. Erre példa a Traefik, amelyben egy [kapcsolóval engedélyezhetjük](https://docs.traefik.io/observability/metrics/prometheus/), hogy a Prometheus számára scrapelhetővé tegye a metrikáit. Ezen metrikákra gyakran találunk kész [Grafana dashboard-ot](https://grafana.com/grafana/dashboards/10902) is, amit csak importálnunk kell.

1. Bizonyos komponensek önmagukban nem publikálnak Prometheus számára metrikákat, azonban saját belső metrikáik vannak, így azokat "csak" transzformálni kell a Prometheus számára. Erre egy példa az Elasticsearch, amely saját magáról részletes információkat nyújt http kéréseken keresztül, de nem direktben a Prometheus által várt formában. Ilyenkor un. [_exporter_](https://github.com/justwatchcom/elasticsearch_exporter)-t telepíthetünk, amely egy kis program, maga is konténer formájában, ami az Elasticsearch-től lekérdezett adatokat transzformálja a Prometheus-nak.

1. Ha pedig magunk által írt komponenseket akarunk monitorozni, akkor a forráskódunkat kell instrumentálni. Ez az adott környezet ismeretében egyszerűbb vagy bonyolultabb feladat. Például .NET Core esetében a [Prometheus-net](https://github.com/prometheus-net/prometheus-net) NuGet csomagon keresztük pár sor kód segítségével kivitelezhető.