# Chapter 4 — Azure OpenAI / Foundry Model Deep Dive: Selection, Prompting, and Deployment

> These code blocks are reproduced from the book in the order they appear. Read the surrounding book section for setup, dependencies, interpretation, and caveats.


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
```


## Prompt engineering fundamentals, done properly


### Code block 3


```text
few_shot_messages = [
```


### Code block 4


```python
tools = [{
```


### Code block 5


```python
response = client.chat.completions.create(
    model="chat-primary",
    messages=[{"role": "user", "content": "What's the coverage limit for policy PL-4471-9902's property claim?"}],
    tools=tools,
    tool_choice="auto",
```


## Token economics: the chapter's opening scenario, formalized


### Code block 6


```python
def estimate_monthly_cost(calls_per_day: int, system_prompt_tokens: int,
```


### Code block 7


```text
    input_tokens_per_call = system_prompt_tokens + avg_user_tokens
    daily_input_cost = calls_per_day * (input_tokens_per_call / 1000) * input_price_per_1k
    daily_output_cost = calls_per_day * (avg_output_tokens / 1000) * output_price_per_1k
    return (daily_input_cost + daily_output_cost) * 30
```


### Code block 8


```text
# Illustrative pricing, not a current or guaranteed rate — check the Azure pricing page for real figures.
print(estimate_monthly_cost(10_000, 4000, 150, 300, 0.005, 0.015))
```


## Latency and throughput: the tradeoff cost estimation doesn't capture


### Code block 9


```python
stream = client.chat.completions.create(
    model="chat-primary",
    messages=[{"role": "user", "content": "Summarize this claim for the adjuster."}],
    stream=True,
```


### Code block 10


```python
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```


## Connecting an application to a Foundry project via SDK


### Code block 11


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


### Code block 12


```bash
az cognitiveservices account rai-policy create \
  --resource-group rg-ai103-labs \
  --name aif-ai103-labs \
  --rai-policy-name claims-intake-policy \
  --mode Default \
```


### Code block 13


```bash
az cognitiveservices account deployment create \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --deployment-name chat-primary \
--model-name gpt-5-mini \
--model-version "2025-08-07" \
  --model-format OpenAI \
  --sku-capacity 10 \
  --sku-name Standard \
```
