# K8S DevOps Azure DevOps-ban és Azure Kubernetes Service-szel

## Témakörök

- Azure szolgáltatások
    - Azure DevOps
    - Azure Kubernetes Service (multitenant)
    - Azure SQL (Sandbox előfizetésből)
- Terraform
- DevOps műveletek Azure DevOps-ban
    - Forráskód kezelés Azure Repos-szal
    - CI+CD Azure Pipelines-szal
    - Monitorozás Azure Monitor-ral

## A labor menete

A hivatalos Azure DevOps labor anyagot követi: <https://www.azuredevopslabs.com/labs/vstsextend/kubernetes/>. Itt csak a kiegészítéseket, ill. egy vázlatot írunk le.

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
- Azure portálra lépjünk be az edu.bme.hu-s címünkkel, válasszuk az autsoft.hu tenantot. Nézzünk szét a resource group-ban. Az AKS namespace-ek közé vegyük fel az edu.bme.hu-s címünk szerintit (`@` előtti rész, pontok nélkül)  
    ```javascript
    {
      "apiVersion": "v1",
      "kind": "Namespace",
      "metadata": { "name": "<edu-s azonosító>" },
    }
    ```

## -1. feladat - egyedi image név és adatbázis név

- az image neve elé tegyük a neptunkódunkat
- docker-compose.yml-ban
    - `image: myhealth.web` -> `image: <neptunkod>.myhealth.web`
- mhc-aks.yaml-ban 93. sor körül
    - `image: __ACR__/myhealth.web:latest` -> `image: __ACR__/<neptunkod>.myhealth.web:latest`
- src/MyHealth.Web/appsettings.json a connection string-ben
    - `Initial Catalog=mhcdb` helyett `Initial Catalog=__SQLDB__`
- ne felejtsünk el commitolni!

## Kitérő: terraform és multitenant infrastruktúra

- [terraform file](https://autsoft.sharepoint.com/:f:/g/shared/AUT/EumyvuEMcWVBlSvpxxtcnL4BThMYJ8D1yyfXQQAv1DjzAQ?e=UN9eiY) main.tf
    - Jelszó: a laborgép jelszava
    - Azure SQL Serverless változatot nem támogat még :(

## 0. feladat Azure és Azure DevOps összekötése

- Project Settings -> Pipelines részen belül Service Connection -> New Service Connection -> Azure Resource Manager -> alul váltsunk a linkkel a teljes verzióra (full version)
- Töltsük ki a terraform file alapján
    - Subscription Name: **VSEai**

## 1. feladat

A laboranyag alapján, továbbá

- Mindenhol, ahol Container Registry-t kell megadni, adjuk meg a teljes login server címet
- A release pipeline-ban klónozzuk a `Create Deployments & Services in AKS` lépést, nevezzük át `Create Namespace`-re, a `Command` rész alatt állítsunk be inline configuration-t a következőre:


- A release pipeline utolsó (`Update image in AKS`) lépésében az `Arguments` részen `image deployments/mhc-front mhc-front=$(ACR)/myhealth.web:$(Build.BuildId)` -> `image deployments/mhc-front mhc-front=$(ACR)/<neptunkód>.myhealth.web:$(Build.BuildId)`

## 2. feladat

A laboranyag alapján, továbbá

- a kubectl parancsok végére mindig írjuk `--namespace=<saját namespace-ünk neve>`
