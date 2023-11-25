# Play.Infra
Play Economy Infrastructure components

## Add the GitHub package source
```powershell
$owner="wallawastoo"
$gh_pat="[PAT HERE]"

dotnet nuget add source --username USERNAME --password $gh_pat --store-password-in-clear-text --name github "https://nuget.pkg.github.com/$owner/index.json"
```

## Creating the Azure resource group
```powershell
$groupname="playeconomy"
$appname="wa-playeconomy"
$appnamenew="waplayeconomy"
az group create --name $groupname --location eastus
```

## Creating the Cosmos DB account
```powershell
az cosmosdb create --name $appname --resource-group $groupname --kind MongoDB --enable-free-tier
```

## Creating the Service Bus namespace
```powershell
az servicebus namespace create --name $appname --resource-group $groupname --sku Standard
```

## Creating the Container Registry
```powershell
az acr create --name $appnamenew --resource-group $groupname --sku Basic
```

## Creating the AKS cluster
```powershell
az aks create -n $appnamenew -g $groupname --node-vm-size Standard_B2s --node-count 2 --attach-acr $appnamenew --enable-oidc-issuer --enable-workload-identity --generate-ssh-keys

az aks get-credentials --name $appnamenew --resource-group $groupname
```

## Creating the Azure Key Vault
```powershell
az keyvault create -n $appnamenew -g $groupname
```

## Installing Emissary-ingress
```powershell
helm repo add datawire https://app.getambassador.io
helm repo update

kubectl apply -f https://app.getambassador.io/yaml/emissary/3.9.0/emissary-crds.yaml
kubectl wait --timeout=90s --for=condition=available deployment emissary-apiext -n emissary-system

$emissarynamespace="emissary"

helm install emissary-ingress datawire/emissary-ingress --set service.annotations."service\.beta\.kubernetes\.io/azure-dns-label-name"=$appname -n $emissarynamespace --create-namespace 
 
kubectl rollout status deployment/emissary-ingress -n $emissarynamespace -w

```

## Kubernets Services and Pods
To check the pods and services
```powershell
kubectl get pods -n $emissarynamespace
kubectl get service emissary-ingress -n $emissarynamespace
```

## Configuring Emissary-ingress routing
```powershell
kubectl apply -f .\emissary-ingress\listener.yaml -n $emissarynamespace
kubectl apply -f .\emissary-ingress\mappings.yaml -n $emissarynamespace
```

## Installing cert-manager
```powershell
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager --version v1.13.2 --set installCRDs=true --namespace $emissarynamespace
```

## Creating the cluster issuer
```powershell
kubectl apply -f .\cert-manager\cluster-issuer.yaml -n $emissarynamespace
kubectl apply -f .\cert-manager\acme-challenge.yaml -n $emissarynamespace
```

## Creating the tls certificate
```powershell
kubectl apply -f .\emissary-ingress\tls-certificate.yaml -n $emissarynamespace

kubectl get certificate -n $emissarynamespace
kubectl describe certificate playeconomy-tls-cert -n $emissarynamespace
```

check the secret 
```powershell
kubectl get secret -n $emissarynamespace
kubectl get secret playeconomy-tls -n $emissarynamespace -o yaml
```

## Enabling TLS and HTTPS
```powershell
kubectl apply -f .\emissary-ingress\host.yaml -n $emissarynamespace
```

## Packaging and publishing the microservice Helm chart
```powershell
helm package .\helm\microservice

$helmUser=[guid]::Empty.Guid
$helmPassword=az acr login --name $appnamenew --expose-token --output tsv --query accessToken

helm registry login "$appnamenew.azurecr.io" --username $helmUser --password $helmPassword

helm push microservice-0.1.0.tgz oci://$appnamenew.azurecr.io/helm
```

## Create Github service principal
```powershell
$appId= az ad sp create-for-rbac -n "GitHub" --query appId --output tsv

az role assignment create --assignee $appId --role "AcrPush" --resource-group $groupname
az role assignment create --assignee $appId --role "Azure Kubernetes Service Cluster User Role" --resource-group $groupname
az role assignment create --assignee $appId --role "Azure Kubernetes Service Contributor Role" --resource-group $groupname
```