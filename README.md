# Play.Infra
Play Economy Infrastructure components

## Add the GitHub package source
```powershell
$owner="WallaWast"
$gh_pat="[PAT HERE]"

dotnet nuget add source --username USERNAME --password $gh_pat --store-password-in-clear-text --name github "https://nuget.pkg.github.com/$owner/index.json"
```