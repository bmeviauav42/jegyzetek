# K8S Azure-ban platformszolgáltatásként (PaaS) - AKS

## A labor menete

A hivatalos [Azure AKS oktatóanyagot](https://docs.microsoft.com/en-us/learn/paths/intro-to-kubernetes-on-azure/) követi. Itt csak a kiegészítéseket, ill. egy vázlatot írunk le.

## Telepítés AKS-be

https://docs.microsoft.com/en-us/learn/modules/aks-deploy-container-app/

```bash
az group create -l westeurope -n $RESOURCE_GROUP
```

```bash
az aks create --resource-group $RESOURCE_GROUP --name $CLUSTER_NAME --node-count 1 --enable-addons http_application_routing --generate-ssh-keys --node-vm-size Standard_A2_v2 --network-plugin azure
```

`az aks create`-nél ha hibás RSA kulcs: a windows user profil *.ssh* könyvtárából az id_rsa-t mozgassuk el. A *System* mode node pool létrehozása kb. 5 perc. A *User* mode node pool létrehozása kb. 3 perc

`kubectl` WSL bash-ben a windows user könyvtárában állva
```bash
kubectl --kubeconfig ./.kube/config get nodes
```

## Extrák Azure Portálon

- Node pool scaling
- Log Analytics Workspace létrehozása, naplók vizsgálata
