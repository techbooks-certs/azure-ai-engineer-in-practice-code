# Chapter 1 — Foundations

These examples follow the book in chapter order. Read the surrounding section for prerequisites, configuration, and security guidance. Some blocks show example output or partial commands rather than standalone scripts.


## Hands-on lab: provisioning the resource group and Foundry project

### Code block 1

```bash
az account set --subscription "<your-subscription-id>"
az group show --name rg-ai103-labs --output table
az configure --defaults group=rg-ai103-labs location=eastus2
```

### Code block 2

```text
Name           Location    Status
-------------  ----------  ---------
rg-ai103-labs  eastus2     Succeeded
```


## Deployment options and why "just deploy it" isn't a complete plan

### Code block 3

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
--sku-name GlobalStandard
```

### Code block 4

```json
{
  "name": "chat-primary",
  "properties": {
"model": { "name": "gpt-5-mini", "format": "OpenAI", "version": "2025-08-07" },
    "provisioningState": "Succeeded",
    "rateLimits": [
      { "key": "token", "renewalPeriod": 60, "count": 10000 }
    ]
  }
}
```


## Quotas, scaling, and cost footprints

### Code block 5

```bash
az cognitiveservices usage list \
  --location eastus2 \
  --output table
```

### Code block 6

```text
Name                         CurrentValue    Limit
---------------------------  --------------  -------
OpenAI.GlobalStandard.gpt-5-mini  10              300
OpenAI.GlobalStandard.text-embedding-3-large   0               50
```


## Identity and access: the RBAC and Key Vault pattern this book uses everywhere

### Code block 7

```bash
az identity create --name id-ai103-labs --resource-group rg-ai103-labs
 
IDENTITY_PRINCIPAL_ID=$(az identity show --name id-ai103-labs \
  --resource-group rg-ai103-labs --query principalId -o tsv)
 
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
```

### Code block 8

```bash
az keyvault create --name kv-ai103-labs --resource-group rg-ai103-labs
 
az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Key Vault Secrets User" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.KeyVault/vaults/kv-ai103-labs"
 
az keyvault secret set --vault-name kv-ai103-labs \
  --name demo-third-party-key --value "placeholder-rotate-before-use"
```


## Connecting Foundry projects to CI/CD

### Code block 9

```yaml
# .github/workflows/foundry-infra-check.yml
name: foundry-infra-check
on:
  pull_request:
    paths:
      - 'infra/**'
jobs:
  validate:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - name: Azure login (OIDC, no stored secret)
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Validate Bicep
        run: |
          az deployment group validate \
            --resource-group rg-ai103-labs \
            --template-file infra/foundry.bicep
```

### Code block 10

```bash
az ad app create --display-name app-ai103-labs-cicd
 
APP_ID=$(az ad app list --display-name app-ai103-labs-cicd --query "[0].appId" -o tsv)
 
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "gh-actions-main",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<your-org>/<your-repo>:ref:refs/heads/main",
  "audiences": ["api://AzureADTokenExchange"]
}'
# A second credential is needed when the same workflow authenticates on pull_request:
az ad app federated-credential create --id "$APP_ID" --parameters '{
  "name": "gh-actions-pr",
  "issuer": "https://token.actions.githubusercontent.com",
  "subject": "repo:<your-org>/<your-repo>:pull_request",
  "audiences": ["api://AzureADTokenExchange"]
}' 
```

### Code block 11

```bash
az ad sp create --id "$APP_ID"
 
RG_ID=$(az group show --name rg-ai103-labs --query id -o tsv)
az role assignment create \
  --assignee "$APP_ID" \
  --role "Contributor" --scope "$RG_ID"
```


## Closing the loop: budget, and the mistake that survives good planning

### Code block 12

```bash
az consumption budget create \
  --budget-name budget-ai103-labs \
  --amount 50 \
  --category cost \
  --time-grain monthly \
  --start-date $(date -u +%Y-%m-01) \
  --end-date 2027-01-01 \
--resource-group rg-ai103-labs
```

