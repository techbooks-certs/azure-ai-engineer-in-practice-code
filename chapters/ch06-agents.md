# Chapter 6 — Agents

These examples follow the book in chapter order. Read the surrounding section for prerequisites, configuration, and security guidance. Some blocks show example output or partial commands rather than standalone scripts.


## Defining an agent: roles, goals, and tool schemas

### Code block 1

```python
from azure.ai.projects.models import PromptAgentDefinition, FunctionTool

def policy_search_tool(query: str) -> list[dict]:
    """Search current policy documents for clauses relevant to a claim question."""
    return search_policy_documents(query)

search_tool = FunctionTool(
    name="search_policy_documents",
    description="Search current insurance policy documents for coverage, exclusions, and limits.",
    parameters={
        "type": "object",
        "properties": {
            "query": {"type": "string", "description": "Standalone policy-search query"}
        },
        "required": ["query"],
        "additionalProperties": False,
    },
    strict=True,
)

triage_agent = project_client.agents.create_version(
    agent_name="claims-triage-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions=(
            "You triage incoming insurance claims. Extract claim details, check "
            "policy coverage, and flag claims requiring human review. Never approve "
            "or deny a claim yourself."
        ),
        tools=[search_tool],
    ),
)
```


## Conversation tracking: Conversations and Responses

### Code block 2

```text
openai = project_client.get_openai_client()
conversation = openai.conversations.create()

agent_ref = {
    "agent_reference": {"name": triage_agent.name, "id": triage_agent.id, "type": "agent_reference"}
}

response = openai.responses.create(
    conversation=conversation.id,
    extra_body=agent_ref,
    input="I need to report water damage from a burst pipe, happened yesterday.",
)
print(response.output_text)

openai.conversations.items.create(
    conversation_id=conversation.id,
    items=[{"type": "message", "role": "user", "content": "Actually, the estimate is closer to $20,000."}],
)
response = openai.responses.create(
    conversation=conversation.id,
    extra_body=agent_ref,
)
print(response.output_text)
```


## Function calling with the Responses API

### Code block 3

```python
import json

# This illustrative dispatcher deliberately allows only a read-only tool.
# `caller` comes from verified application authentication, never model text.
def run_response_with_tools(agent_client, user_input: str, caller):
    conversation = agent_client.conversations.create()
    response = agent_client.responses.create(
        conversation=conversation.id, input=user_input
    )
    for _ in range(8):  # bounded tool-round-trip loop
        outputs = []
        for item in response.output:
            if item.type != "function_call":
                continue
            if item.name != "search_policy_documents":
                raise PermissionError("Tool not in the application allowlist")
            args = json.loads(item.arguments)
            if (not isinstance(args, dict) or set(args) != {"query"}
                    or not isinstance(args["query"], str)
                    or not 1 <= len(args["query"].strip()) <= 500):
                raise ValueError("Invalid search arguments")
            if "policy:read" not in caller.permissions:
                raise PermissionError("Caller cannot search policies")
            result = search_policy_documents(args["query"])
            outputs.append({"type": "function_call_output",
                            "call_id": item.call_id,
                            "output": json.dumps(result)})
        if not outputs:
            return response
        response = agent_client.responses.create(
            conversation=conversation.id, input=outputs
        )
    if any(i.type == "function_call" for i in response.output):
        raise RuntimeError("Agent exceeded tool-call iteration limit")
    return response
```


## Integrating agent tools: beyond function calling

### Code block 4

```python
def analyze_intake_document(document_url: str) -> dict:
    """Run the claims-intake Content Understanding analyzer on a document URL."""
    analysis = cu_client.begin_analyze(
        analyzer_id="claims-intake-form-extractor",
        inputs=[AnalysisInput(url=document_url)],
    ).result()
    content = analysis.contents[0]
    return {
        "fields": {k: v.value for k, v in (content.fields or {}).items()},
        "markdown": content.markdown,
    }

from azure.ai.projects.models import (
    PromptAgentDefinition,
    FileSearchTool,
    CodeInterpreterTool,
    AutoCodeInterpreterToolParam,
)

# Prepare a small static reference set for File Search.
underwriting_guidelines_store = openai.vector_stores.create(name="underwriting-guidelines")
with open("underwriting-guidelines.pdf", "rb") as fh:
    openai.vector_stores.files.upload_and_poll(
        vector_store_id=underwriting_guidelines_store.id,
        file=fh,
    )

reference_agent = project_client.agents.create_version(
    agent_name="claims-estimator-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions=(
            "Use file search for underwriting guidance and code interpreter "
            "for arithmetic over claim line items."
        ),
        tools=[
            FileSearchTool(vector_store_ids=[underwriting_guidelines_store.id]),
            CodeInterpreterTool(container=AutoCodeInterpreterToolParam()),
        ],
    ),
)
```


## Orchestrating hybrid LLM-and-rules-engine systems

### Code block 5

```python
def check_coverage_limit_deterministic(claim_type: str, policy_tier: str) -> float:
    # A direct table lookup, not a model call — deterministic, fast, and cheap.
    return COVERAGE_LIMIT_TABLE[claim_type][policy_tier]
```


## Multi-agent orchestration: supervisor/worker, done for real

### Code block 6

```python
 
research_agent = project_client.agents.create_version(
    agent_name="claims-research-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions="Research policy evidence; never decide claim outcomes.",
        tools=[search_tool],
    ),
)
summarization_agent = project_client.agents.create_version(
    agent_name="claims-summarization-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions="Summarize supplied policy research for compliance review.",
    ),
)
action_agent = project_client.agents.create_version(
    agent_name="claims-action-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions="Recommend a routing queue; never execute the route.",
    ),
)

def invoke_named_agent(agent, payload: dict, caller) -> str:
    # Pre-bind the client to the named agent (documented Foundry pattern).
    agent_client = project_client.get_openai_client(agent_name=agent.name)
    response = run_response_with_tools(
        agent_client, json.dumps(payload), caller
    )
    return response.output_text

def supervisor_orchestrate(claim_details: dict, caller) -> dict:
    research = invoke_named_agent(research_agent, claim_details, caller)
    summary = invoke_named_agent(
        summarization_agent, {"research": research}, caller
    )
    routing = invoke_named_agent(action_agent, {"summary": summary}, caller)
    return {"research": research, "summary": summary, "decision": routing}
```


## Autonomous workflows, safeguards, and approval checkpoints

### Code block 7

```python
import uuid
 
def queue_for_human_approval(action: dict) -> str:
    """Write the pending action to a durable queue an adjuster's review
    dashboard polls — the specific queue technology (Storage Queue, Service
    Bus, a database table) is an implementation detail; what matters here is
    that the action does not execute until a person acts on this record."""
    approval_request_id = str(uuid.uuid4())
    approval_queue_client.send_message(json.dumps({"id": approval_request_id, "action": action, "status": "pending"}))
    return approval_request_id
 
def execute_action_with_approval(action: dict, requires_approval: bool) -> dict:
    if requires_approval:
        approval_request_id = queue_for_human_approval(action)
        return {"status": "pending_approval", "approval_request_id": approval_request_id}
    return execute_action(action)
```


## Monitoring, evaluation, and error analysis

### Code block 8

```text
# Package direction for current client-side tracing:
# pip install azure-ai-projects azure-identity opentelemetry-sdk azure-core-tracing-opentelemetry
#
# Configure the project's tracing / Application Insights connection, then run
# the same Responses calls shown above. Foundry and OpenTelemetry record the
# model/tool spans; inspect them in Foundry > Traces or Application Insights.
```

### Code block 9

```python
def evaluate_routing_decision(routing_result: dict) -> str:
    evidence = {
        "research": routing_result["research"],
        "summary": routing_result["summary"],
        "decision": routing_result["decision"],
    }
    check = client.responses.create(
        model="chat-primary",
        instructions=(
            "Review the routing decision against the supplied observable evidence. "
            "Flag any claim in the decision that is unsupported by the research or summary."
        ),
        input=json.dumps(evidence),
    )
    return check.output_text
```


## Hands-on lab: the full multi-agent workflow with an approval checkpoint

### Code block 10

```python
def process_claim_end_to_end(intake_audio_path: str) -> dict:
    intake_result = process_claim_intake(intake_audio_path)  # Chapter 2's tool
    if intake_result["status"] == "escalated":
        approval_request_id = queue_for_human_approval({"reason": intake_result["reason"], "priority": "high"})
        return {"status": "pending_approval", "approval_request_id": approval_request_id}
 
    routing_result = supervisor_orchestrate(intake_result["extraction"])  # research -> summarize -> route
 
    requires_approval = intake_result["extraction"]["requires_human_review"]
    return execute_action_with_approval(routing_result, requires_approval)
```

