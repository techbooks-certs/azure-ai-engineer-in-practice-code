# Chapter 4 — Models and Prompting

These examples follow the book in chapter order. Read the surrounding section for prerequisites, configuration, and security guidance. Some blocks show example output or partial commands rather than standalone scripts.


## The Foundry model catalog, chosen deliberately

### Code block 1

```text
# Current Foundry Models use the generally available OpenAI v1-compatible route.
# Use the deployment name in the model field; the exact model provider can vary.
code_client = OpenAI(
    base_url="https://aif-ai103-labs.openai.azure.com/openai/v1/",
    api_key=token_provider,
)

code_completion = code_client.chat.completions.create(
model="code-specialist",
    messages=[
        {"role": "system", "content": "Generate a Bicep parameter file. Output only valid Bicep, no explanation."},
        {"role": "user", "content": "A bicepparam file setting foundryName to 'aif-ai103-labs' and location to 'eastus2'."},
    ],
)
```


## Deployment types and regional/capacity considerations, revisited

### Code block 2

```bash
az cognitiveservices account deployment create \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --deployment-name chat-ptu-pilot \
--model-name gpt-5-mini \
--model-version "2025-08-07" \
  --model-format OpenAI \
  --sku-name ProvisionedManaged \
  --sku-capacity 100
```


## Prompt engineering fundamentals, done properly

### Code block 3

```text
few_shot_messages = [
    {"role": "system", "content": "Summarize claims notes in the house style: one sentence, past tense, no adjectives about severity."},
    {"role": "user", "content": "Customer called upset about a delayed roof repair, water still leaking into the attic."},
    {"role": "assistant", "content": "Customer reported ongoing attic water intrusion due to delayed roof repair."},
    {"role": "user", "content": "Claimant says the contractor never showed up for the scheduled inspection, third time this month."},
]
```

### Code block 4

```python
tools = [{
    "type": "function",
    "function": {
        "name": "lookup_policy_coverage_limit",
        "description": "Look up the coverage limit for a given claim type and policy number.",
        "parameters": {
            "type": "object",
            "properties": {
                "policy_number": {"type": "string"},
                "claim_type": {"type": "string", "enum": ["property", "auto", "liability", "other"]},
            },
            "required": ["policy_number", "claim_type"],
        },
    },
}]
 
response = client.chat.completions.create(
    model="chat-primary",
    messages=[{"role": "user", "content": "What's the coverage limit for policy PL-4471-9902's property claim?"}],
    tools=tools,
    tool_choice="auto",
)
```

### Code block 5

```text
finish_reason: "tool_calls"
tool_calls: [{
  "function": {
    "name": "lookup_policy_coverage_limit",
    "arguments": "{\"policy_number\": \"PL-4471-9902\", \"claim_type\": \"property\"}"
  }
}]
```


## Token economics: the chapter's opening scenario, formalized

### Code block 6

```python
def estimate_monthly_cost(calls_per_day: int, system_prompt_tokens: int,
                           avg_user_tokens: int, avg_output_tokens: int,
                           input_price_per_1k: float, output_price_per_1k: float) -> float:
    input_tokens_per_call = system_prompt_tokens + avg_user_tokens
    daily_input_cost = calls_per_day * (input_tokens_per_call / 1000) * input_price_per_1k
    daily_output_cost = calls_per_day * (avg_output_tokens / 1000) * output_price_per_1k
    return (daily_input_cost + daily_output_cost) * 30
 
# Illustrative pricing, not a current or guaranteed rate — check the Azure pricing page for real figures.
print(estimate_monthly_cost(10_000, 4000, 150, 300, 0.005, 0.015))
```


## Latency and throughput: the tradeoff cost estimation doesn't capture

### Code block 7

```python
stream = client.chat.completions.create(
    model="chat-primary",
    messages=[{"role": "user", "content": "Summarize this claim for the adjuster."}],
    stream=True,
)
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```


## Connecting an application to a Foundry project via SDK

### Code block 8

```python
import os
from azure.ai.projects import AIProjectClient

project_client = AIProjectClient(
    endpoint=os.environ["AZURE_AI_PROJECT_ENDPOINT"],
    credential=credential,
)
openai = project_client.get_openai_client()

response = openai.responses.create(
    model="chat-primary",
    input="Summarize this claim for the adjuster.",
)
print(response.output_text)
```


## Content filtering on the way in and out

### Code block 9

```text
# Model deployment only; default guardrail remains until claims-intake-policy is assigned in Foundry or ARM.
```

### Code block 10

```bash
az cognitiveservices account deployment create \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --deployment-name chat-primary \
  --model-name gpt-5-mini \
  --model-version "2025-08-07" \
  --model-format OpenAI \
  --sku-capacity 10 \
  --sku-name Standard
```

