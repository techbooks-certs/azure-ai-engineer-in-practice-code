# Chapter 7 — Responsible AI, Security, and Production Operations

> These code blocks are reproduced from the book in the order they appear. Read the surrounding book section for setup, dependencies, interpretation, and caveats.


## Safety filters, guardrails, and risk detection — configured, not just called


### Code block 1


```text
# Prompt Shields current public API (REST shape)
import requests

shield_response = requests.post(
    "https://<content-safety-endpoint>/contentsafety/text:shieldPrompt?api-version=2024-09-01",
    headers={
        "Authorization": f"Bearer {credential.get_token('https://cognitiveservices.azure.com/.default').token}",
        "Content-Type": "application/json",
    },
    json={"userPrompt": user_message, "documents": []},
    timeout=30,
)
shield_response.raise_for_status()
is_jailbreak_attempt = shield_response.json()["userPromptAnalysis"]["attackDetected"]
```


## Responsible AI instrumentation: evaluators as an ongoing practice


### Code block 2


```python
from azure.ai.evaluation import ContentSafetyEvaluator, evaluate
```


### Code block 3


```text
evaluation_results = evaluate(
    data="production_conversation_sample.jsonl",
    evaluators={
```


### Code block 4


```text
    evaluator_config={"default": {"column_mapping": {"query": "${data.query}", "response": "${data.response}"}}},
```


## Auditing: trace logging, provenance, and approval workflows


### Code block 5


```python
from datetime import datetime, timezone
```


### Code block 6


```text
audit_record = {
```


## Security hardening end to end


### Code block 7


```bash
az network vnet create \
  --name vnet-ai103-labs --resource-group rg-ai103-labs \
  --address-prefix 10.0.0.0/16 \
```


### Code block 8


```bash
az cognitiveservices account update \
  --name aif-ai103-labs --resource-group rg-ai103-labs \
  --custom-domain aif-ai103-labs \
```


### Code block 9


```bash
az network private-endpoint create \
  --name pe-aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --vnet-name vnet-ai103-labs \
  --subnet snet-ai-services \
  --private-connection-resource-id "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.CognitiveServices/accounts/aif-ai103-labs" \
  --group-id account \
```


### Code block 10


```xml
<rate-limit calls="60" renewal-period="60" />
```


## Closing the loop: an actual deployment step, and drift detection


### Code block 11


```yaml
# .github/workflows/foundry-infra-check.yml (extends Chapter 1's file)
```


### Code block 12


```bash
          az deployment group validate \
            --resource-group rg-ai103-labs \
```


### Code block 13


```bash
          az deployment group create \
            --resource-group rg-ai103-labs \
            --template-file infra/foundry.bicep \
```


### Code block 14


```yaml
# .github/workflows/foundry-drift-check.yml
```


### Code block 15


```bash
          az deployment group what-if \
            --resource-group rg-ai103-labs \
            --template-file infra/foundry.bicep \
```


## Monitoring: performance, drift, and pipeline health together


### Code block 16


```bash
az monitor log-analytics workspace create \
```


### Code block 17


```bash
LAW_ID=$(az monitor log-analytics workspace show \
--workspace-name law-ai103-labs --resource-group rg-ai103-labs \
--query id -o tsv)

az monitor diagnostic-settings create \
  --name diag-aif-ai103-labs \
  --resource "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.CognitiveServices/accounts/aif-ai103-labs" \
--workspace "$LAW_ID" \
--logs '[{"categoryGroup":"allLogs","enabled":true}]' \
```


### Code block 18


```bash
az monitor scheduled-query create \
  --name alert-groundedness-degradation \
  --resource-group rg-ai103-labs \
  --scopes "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.OperationalInsights/workspaces/law-ai103-labs" \
--condition "avg 'GroundednessScore_d' from 'Eval' < 3.5" \
--condition-query Eval="EvaluationResults_CL | where TimeGenerated > ago(1h) | project GroundednessScore_d" \
  --window-size 1h --evaluation-frequency 15m \
```


## Cost, quota, and rate-limit management at scale


### Code block 19


```text
# Interpret utilization returned by your chosen Azure Monitor query.
# These are example review thresholds for this fictional workload, not Azure defaults.
LOW_UTILIZATION_REVIEW_THRESHOLD = 0.30
HIGH_UTILIZATION_REVIEW_THRESHOLD = 0.85

def interpret_ptu_utilization(avg_utilization: float) -> dict:
    if avg_utilization < LOW_UTILIZATION_REVIEW_THRESHOLD:
        return {"recommendation": "Review whether reserved PTU capacity is justified", "utilization": avg_utilization}
    if avg_utilization > HIGH_UTILIZATION_REVIEW_THRESHOLD:
        return {"recommendation": "Review capacity headroom before further traffic growth", "utilization": avg_utilization}
    return {"recommendation": "Within this workload's review band", "utilization": avg_utilization}
```


## Hands-on lab: hardening the claims-triage system


### Code block 20


```python
def harden_production_system() -> dict:
    results = {}
```


### Code block 21


```text
        resources=["aif-ai103-labs", "srch-ai103-labs"]
```


### Code block 22


```python
        deployment="chat-primary",
        policy_name="claims-intake-policy",
        jailbreak_detection=True,
        blocklists=["claims-domain-blocklist"],
```


### Code block 23


```text
        sample_rate=0.1,  # evaluate 10% of production conversations
        evaluators=["groundedness", "content_safety"],
```


### Code block 24


```text
        workspace="law-ai103-labs",
        alert_on="groundedness_degradation",
```


### Code block 25


```text
    return results
```
