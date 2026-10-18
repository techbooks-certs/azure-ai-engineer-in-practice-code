# Chapter 7 — Security and Operations

These examples follow the book in chapter order. Read the surrounding section for prerequisites, configuration, and security guidance. Some blocks show example output or partial commands rather than standalone scripts.


## Safety filters, guardrails, and risk detection — configured, not just called

### Code block 1

```python
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
 
evaluation_results = evaluate(
    data="production_conversation_sample.jsonl",
    evaluators={
        "groundedness": groundedness_eval,       # from Chapter 5
        "content_safety": ContentSafetyEvaluator(credential=credential, azure_ai_project=PROJECT_ENDPOINT),
    },
    evaluator_config={"default": {"column_mapping": {"query": "${data.query}", "response": "${data.response}"}}},
)
```


## Auditing: trace logging, provenance, and approval workflows

### Code block 3

```python
from datetime import datetime, timezone
gate_result = execute_action_with_approval(routing_result, requires_approval)  # sole invocation in this flow; never repeat for audit
audit_record = {
    "response_id": response.id,
    "conversation_id": conversation.id,
    "model_version": "gpt-5-mini-2025-08-07",
    "prompt_version": "claims-triage-v3",
    "retrieved_sources": [r["source"] for r in retrieved_results],
    "decision": routing_result["decision"],
    "gate_status": gate_result.get("status"),
    "approval_request_id": gate_result.get("approval_request_id"),
    "human_approved": None,  # no reviewer decision recorded (pending/not required)
    "timestamp": datetime.now(timezone.utc).isoformat(),
}
audit_sink.write(audit_record)  # application abstraction: route to your durable audit store
```


## Security hardening end to end

### Code block 4

```bash
az network vnet create \
  --name vnet-ai103-labs --resource-group rg-ai103-labs \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-ai-services --subnet-prefix 10.0.1.0/24
```

### Code block 5

```bash
az cognitiveservices account update \
  --name aif-ai103-labs --resource-group rg-ai103-labs \
  --custom-domain aif-ai103-labs \
  --public-network-access Disabled
 
az network private-endpoint create \
  --name pe-aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --vnet-name vnet-ai103-labs \
  --subnet snet-ai-services \
  --private-connection-resource-id "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.CognitiveServices/accounts/aif-ai103-labs" \
  --group-id account \
  --connection-name conn-aif-ai103-labs
```

### Code block 6

```text
<!-- APIM inbound policy fragment -->
<rate-limit calls="60" renewal-period="60" />
<validate-content unspecified-content-type-action="prevent" max-size="102400" />
<authentication-managed-identity resource="https://cognitiveservices.azure.com" />
```


## Closing the loop: an actual deployment step, and drift detection

### Code block 7

```yaml
# .github/workflows/foundry-infra-check.yml (extends Chapter 1's file)
name: foundry-infra-check
on:
  pull_request:
    paths:
      - 'infra/**'
  push:
    branches: [main]
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
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Validate Bicep
        run: |
          az deployment group validate \
            --resource-group rg-ai103-labs \
            --template-file infra/foundry.bicep
 
  deploy:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Deploy infrastructure
        run: |
          az deployment group create \
            --resource-group rg-ai103-labs \
            --template-file infra/foundry.bicep \
            --mode Incremental
```

### Code block 8

```yaml
# .github/workflows/foundry-drift-check.yml
name: foundry-drift-check
on:
  schedule:
    - cron: '0 6 * * *'
jobs:
  drift-check:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - name: Check for configuration drift
        run: |
          az deployment group what-if \
            --resource-group rg-ai103-labs \
            --template-file infra/foundry.bicep \
            --mode Incremental
```


## Monitoring: performance, drift, and pipeline health together

### Code block 9

```bash
az monitor log-analytics workspace create \
  --workspace-name law-ai103-labs --resource-group rg-ai103-labs
```

### Code block 10

```text
LAW_ID=$(az monitor log-analytics workspace show \
--workspace-name law-ai103-labs --resource-group rg-ai103-labs \
--query id -o tsv)

az monitor diagnostic-settings create \
  --name diag-aif-ai103-labs \
  --resource "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.CognitiveServices/accounts/aif-ai103-labs" \
--workspace "$LAW_ID" \
--logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'
```

### Code block 11

```bash
az monitor scheduled-query create \
  --name alert-groundedness-degradation \
  --resource-group rg-ai103-labs \
  --scopes "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.OperationalInsights/workspaces/law-ai103-labs" \
--condition "avg 'GroundednessScore_d' from 'Eval' < 3.5" \
--condition-query Eval="EvaluationResults_CL | where TimeGenerated > ago(1h) | project GroundednessScore_d" \
  --window-size 1h --evaluation-frequency 15m \
  --action-groups "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.Insights/actionGroups/ag-ai103-oncall"
```


## Cost, quota, and rate-limit management at scale

### Code block 12

```python
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

### Code block 13

```python
def harden_production_system() -> dict:
    results = {}
    results["networking"] = disable_public_access_and_add_private_endpoints(
        resources=["aif-ai103-labs", "srch-ai103-labs"]
    )
    results["content_filter"] = attach_content_filter_policy(
        deployment="chat-primary",
        policy_name="claims-intake-policy",
        jailbreak_detection=True,
        blocklists=["claims-domain-blocklist"],
    )
    results["evaluation"] = schedule_continuous_evaluation(
        sample_rate=0.1,  # evaluate 10% of production conversations
        evaluators=["groundedness", "content_safety"],
    )
    results["monitoring"] = configure_diagnostics_and_alerts(
        workspace="law-ai103-labs",
        alert_on="groundedness_degradation",
    )
    results["audit"] = enable_audit_logging(table="ClaimsAuditLog")
    return results
```

