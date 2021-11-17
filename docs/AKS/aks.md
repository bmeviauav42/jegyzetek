# K8S Azure-ban platformszolgáltatásként (PaaS) - AKS

## A labor menete

A hivatalos [Azure AKS oktatóanyagot](https://docs.microsoft.com/en-us/learn/paths/intro-to-kubernetes-on-azure/) követi. Itt csak a kiegészítéseket, ill. egy vázlatot írunk le.

## Telepítés AKS-be

https://docs.microsoft.com/en-us/learn/modules/aks-deploy-container-app/

Változók beállítása

```bash
$RESOURCE_GROUP="autaks"
$CLUSTER_NAME="aks-contoso-video"
```

Alapértelmezett előfizetés beállítása

```bash
az account show
az account set -s <id>
```

Alapértelmezett régió

```bash
az configure --defaults location=westeurope
```

Új resource group

```bash
az group create -l westeurope -n $RESOURCE_GROUP
```

AKS létrehozás

```bash
az aks create --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --node-count 1 --enable-addons http_application_routing --generate-ssh-keys --node-vm-size Standard_A2_v2 --network-plugin azure
```

`az aks create`-nél **ha** hibás RSA kulcs: a windows user profil *.ssh* könyvtárából az id_rsa-t mozgassuk el. A *System* mode node pool létrehozása kb. 5 perc. A *User* mode node pool létrehozása kb. 3 perc

```bash
kubectl --kubeconfig ./.kube/config get nodes
```

Opcionális a cluster létrejötte után: Log Analytics Workspace létrehozása és összekötése

## Extrák Azure Portálon

- Node pool scaling
- Analytics Workspace létrehozás `az monitor log-analytics workspace create -g $RESOURCE_GROUP -n aksaw`
- naplók vizsgálata


## Takarítás

https://docs.microsoft.com/en-us/learn/modules/aks-deploy-container-app/8-summary
