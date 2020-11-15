# K8S DevOps Azure DevOps-ban és Azure Kubernetes Service-szel

## Témakörök

- Azure szolgáltatások
    - Azure DevOps
    - Azure Kubernetes Service (multitenant)
    - Azure SQL (multitenant)
- DevOps műveletek Azure DevOps-ban
    - Forráskód kezelés Azure Repos-szal
    - CI+CD Azure Pipelines-szal
    - Monitorozás Azure Monitor-ral

## A labor menete

A hivatalos [Azure DevOps k8s labor anyagot](https://www.azuredevopslabs.com/labs/vstsextend/kubernetes/) követi. Itt csak a kiegészítéseket, ill. egy vázlatot írunk le.

## Előkészületek

- Azure DevOps
    - szervezet létrehozása
    - be fog kérni nevet, emailt, országot (nem kell hozzájárulni az emailküldéshez)
    - Utána **Get started with Azure DevOps** dialógus jelenik meg -> Continue (nem kell hozzájárulni az emailküldéshez)
    - Ha bejön az új projekt készítő felület, akkor nem kell használatba venni
    - ... helyette Azure DevOps projekt generálása a laboranyag alapján
    - Azure Repos: projekt felfedezése konténer szempontból
        - `docker-compose.yml`
        - `mhc-aks.yaml`
- legyen az **edu-s azonosító** ez: a `@edu.bme.hu` előtti rész pontok nélkül
- Azure portálra lépjünk be az edu.bme.hu-s címünkkel, válasszuk az autsoft.hu tenantot. Nézzünk szét a resource group-ban. Az AKS namespace-ek közé vegyük fel az edu-s azonosító szerintit
    ```javascript
    {
      "apiVersion": "v1",
      "kind": "Namespace",
      "metadata": { "name": "<edu-s azonosító>" }
    }
    ```

## -1. feladat - egyedi image név és adatbázis név

- az image neve elé tegyük az edu-s azonosítónkat
- docker-compose.yml-ban
    - `image: myhealth.web` -> `image: <edu-s azonosító>.myhealth.web`
- mhc-aks.yaml-ban 93. sor körül
    - `image: __ACR__/myhealth.web:latest` -> `image: __ACR__/<edu-s azonosító>.myhealth.web:latest`
- src/MyHealth.Web/appsettings.json a connection string-ben
    - `Initial Catalog=mhcdb` helyett `Initial Catalog=__SQLDB__`
- ne felejtsünk el commitolni!

## 0. feladat Azure és Azure DevOps összekötése

- Project Settings -> Pipelines részen belül Service Connection -> New Service Connection -> Azure Resource Manager -> Service principal (manual)

## 1. feladat

A laboranyag alapján, továbbá
- A build pipeline változók közé vegyük fel az SQLDB-t is
- Mindenhol, ahol Container Registry-t kell megadni, adjuk meg a teljes login server címet. Ugyanígy, ahol Azure SQL szerver címet kell megadni, adjuk meg a teljes FQDN címet, de `tcp:` nélkül
- Minden docker compose taszknál legyen kiválasztva a container registry (különben nem taggeli be a push-hoz az image-et)
- Az agent legyen hosted (Azure Pipelines), a database deployment lépésnél windows agent kell, a többi lehet linuxos
- A kubectl paramcsoknál töltsük ki a névteret: legyen a saját névterünk
- A release pipeline utolsó (`Update image in AKS`) lépésében az `Arguments` részen 

`image deployments/mhc-front mhc-front=$(ACR)/myhealth.web:$(Build.BuildId)` helyett

**`image deployments/mhc-front mhc-front=$(ACR)/<edu-s azonosító>.myhealth.web:$(Build.BuildId)`**

## 2. feladat

A laboranyag alapján, viszont ne a k8s dashboard-ot használjuk
