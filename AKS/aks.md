# K8S DevOps Azure DevOps-ban és Azure Kubernetes Service-szel

# Témakörök
- Azure szolgáltatások
  - Azure DevOps
  - Azure Kubernetes Service (multitenant)
  - Azure SQL (multitenant)
- Terraform  
- DevOps műveletek Azure DevOps-ban
  - Forráskód kezelés Azure Repos-szal
  - CI+CD Azure Pipelines-szal
  - Monitorozás Azure Monitor-ral
 
 # A labor menete
 
 A hivatalos Azure DevOps labor anyagot követi: https://www.azuredevopslabs.com/labs/vstsextend/kubernetes/
 Itt csak a kiegészítéseket, ill. egy vázlatot írunk le.
 
 ## Előkészületek
 - Azure DevOps szervezet létrehozása
  - Be fog kérni nevet, emailt, országot (nem kell hozzájárulni az emailküldéshez)
  - Utána **Get started with Azure DevOps** dalógus jelenik meg -> Continue (nem kell hozzájárulni az emailküldéshez)
  - Ha bejön az új projekt készítő felület, akkor nem kell használatba venni
 - Helyette Azure DevOps projekt generálása a laboranyag alapján
 - DevOps elmélet
    https://docs.microsoft.com/en-us/azure/devops/learn/what-is-devops
 - Azure Repos: projekt felfedezése konténer szempontból
   - docker-compose.yml
   - mhc-aks.yaml
  
 ## 0. feladat - egyedi image név
  - az image neve elé tegyük a neptunkódunkat
  - docker-compose.yml-ban
    - `image: myhealth.web` -> `image: <neptunkod>.myhealth.web`
  - mhc-aks.yaml-ban 93.sor körül
    -  `image: __ACR__/myhealth.web:latest` -> `image: __ACR__/<neptunkod>.myhealth.web:latest`
 - ne felejtsünk el commitolni!
  
 
 
 
