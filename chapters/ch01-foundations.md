# Chapter 1 — Foundations of Azure AI Engineering and the AI-103 Landscape

> These code blocks are reproduced from the book in the order they appear. Read the surrounding book section for setup, dependencies, interpretation, and caveats.


## Hands-on lab: provisioning the resource group and Foundry project


### Code block 1


```bash
az account set --subscription "<your-subscription-id>"
az group show --name rg-ai103-labs --output table
az configure --defaults group=rg-ai103-labs location=eastus2
```


## Deployment options and why "just deploy it" isn't a complete plan


### Code block 2


```bash
az cognitiveservices model list \
--location eastus2 \
--query "[?model.name=='gpt-5-mini'].{version:model.version,skus:join(',',model.skus[].name)}" \
--output table

az cognitiveservices account deployment create \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --deployment-name chat-primary \
--model-name gpt-5-mini \
--model-version "2025-08-07" \
  --model-format OpenAI \
  --sku-capacity 10 \
```


## Quotas, scaling, and cost footprints


### Code block 3


```bash
az cognitiveservices usage list \
  --location eastus2 \
```


## Identity and access: the RBAC and Key Vault pattern this book uses everywhere


### Code block 4


```bash
az identity create --name id-ai103-labs --resource-group rg-ai103-labs
```


### Code block 5


```bash
IDENTITY_PRINCIPAL_ID=$(az identity show --name id-ai103-labs \
```


### Code block 6


```bash
PROJECT_ID=$(az cognitiveservices account project show \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --project-name proj-claims-demo \
  --query id -o tsv)

# Foundry User role ID (use the GUID while the role-name rename rolls out)
az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "53ca6127-db72-4b80-b1b0-d745d6d5456d" \
  --scope "$PROJECT_ID"

Foundry User is the current least-privilege developer role for building and testing inside a Foundry project. Microsoft explicitly recommends it instead of the older Azure AI Developer role for current Foundry projects. The role is scoped to proj-claims-demo here so the identity does not gain development rights across unrelated projects.

This is the pattern, not the complete role list: id-ai103-labs will still need separate, narrowly scoped data-plane roles when later chapters add services that are not covered by Foundry project RBAC — for example, Cognitive Services Speech User on the Speech resource and Search Service Contributor plus Search Index Data Contributor on Azure AI Search.
```


### Code block 7


```bash
az keyvault create --name kv-ai103-labs --resource-group rg-ai103-labs
```


### Code block 8


```bash
az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Key Vault Secrets User" \
```


### Code block 9


```bash
az keyvault secret set --vault-name kv-ai103-labs \
```


## Connecting Foundry projects to CI/CD


### Code block 10


```yaml
# .github/workflows/foundry-infra-check.yml
```


### Code block 11


```bash
          az deployment group validate \
            --resource-group rg-ai103-labs \
```


### Code block 12


```bash
az ad app create --display-name app-ai103-labs-cicd
```


### Code block 13


```bash
APP_ID=$(az ad app list --display-name app-ai103-labs-cicd --query "[0].appId" -o tsv)
```


### Code block 14


```bash
az ad app federated-credential create --id "$APP_ID" --parameters '{
# A second credential is needed when the same workflow authenticates on pull_request:
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "gh-actions-pr",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<your-org>/<your-repo>:pull_request",
  "audiences": ["api://AzureADTokenExchange"]
}'
```


### Code block 15


```bash
az ad sp create --id "$APP_ID"
```


### Code block 16


```bash
az role assignment create \
  --assignee "$APP_ID" \
  --role "Contributor" \
```


## Closing the loop: budget, and the mistake that survives good planning


### Code block 17


```bash
az consumption budget create \
  --budget-name budget-ai103-labs \
  --amount 50 \
  --category cost \
  --time-grain monthly \
  --start-date $(date -u +%Y-%m-01) \
  --end-date 2027-01-01 \
```
