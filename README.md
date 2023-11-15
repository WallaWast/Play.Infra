# Play.Infra
Play Economy Infrastructure components

## Add the GitHub package source
```powershell
$owner="WallaWast"
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