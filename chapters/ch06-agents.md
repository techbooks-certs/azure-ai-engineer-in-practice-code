# Chapter 6 — Agentic AI Patterns: Orchestration, Tools, and Multi-Agent Systems

> These code blocks are reproduced from the book in the order they appear. Read the surrounding book section for setup, dependencies, interpretation, and caveats.


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


```python
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

TOOL_FUNCTIONS = {
    "search_policy_documents": search_policy_documents,
    "process_claim_intake": process_claim_intake,
}

def run_response_with_tools(openai_client, agent, user_input: str):
    conversation = openai_client.conversations.create()
    agent_ref = {
        "agent_reference": {"name": agent.name, "id": agent.id, "type": "agent_reference"}
    }
    response = openai_client.responses.create(
        conversation=conversation.id,
        extra_body=agent_ref,
        input=user_input,
    )

    tool_outputs = []
    for item in response.output:
        if item.type != "function_call":
            continue
        fn = TOOL_FUNCTIONS[item.name]
        result = fn(**json.loads(item.arguments))
        tool_outputs.append({
            "type": "function_call_output",
            "call_id": item.call_id,
            "output": json.dumps(result),
        })

    if not tool_outputs:
        return response

    return openai_client.responses.create(
        conversation=conversation.id,
        extra_body=agent_ref,
        input=tool_outputs,
    )
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

For small, static document sets, File Search removes the need to build a separate Search ingestion pipeline. The OpenAI-compatible client owns the file/vector-store operations; the resulting vector-store ID is then attached to the versioned prompt agent.
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


```text
research_agent = project_client.agents.create_version(
    agent_name="claims-research-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions="Research policy coverage for the supplied claim. Do not make routing decisions.",
        tools=[search_tool],
    ),
)

summarization_agent = project_client.agents.create_version(
    agent_name="claims-summarization-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions="Produce a concise compliance-formatted summary of the supplied research.",
    ),
)

action_agent = project_client.agents.create_version(
    agent_name="claims-action-agent",
    definition=PromptAgentDefinition(
        model="chat-primary",
        instructions="Recommend a routing queue from the supplied summary. Do not execute the route.",
    ),
)

def invoke_named_agent(agent, payload: dict) -> str:
    conversation = openai.conversations.create()
    response = openai.responses.create(
        conversation=conversation.id,
        extra_body={"agent_reference": {"name": agent.name, "id": agent.id, "type": "agent_reference"}},
        input=json.dumps(payload),
    )
    # If this agent emits a function_call, use the same function-call-output
    # round trip shown earlier before consuming response.output_text.
    return response.output_text

def supervisor_orchestrate(claim_details: dict) -> dict:
    research = invoke_named_agent(research_agent, claim_details)
    summary = invoke_named_agent(summarization_agent, {"research": research})
    routing = invoke_named_agent(action_agent, {"summary": summary})
    return {"research": research, "summary": summary, "decision": routing}

This is genuine multi-agent orchestration because three separately versioned agents have distinct roles; the research agent additionally has policy-search access. The supervisor is ordinary code because the sequence is fixed. A dynamic LLM supervisor is justified only when the route itself varies by case.
```


## Autonomous workflows, safeguards, and approval checkpoints


### Code block 7


```python
import uuid
```


### Code block 8


```python
def queue_for_human_approval(action: dict) -> str:
```


### Code block 9


```text
    approval_request_id = str(uuid.uuid4())
```


### Code block 10


```text
    return approval_request_id
```


### Code block 11


```python
def execute_action_with_approval(action: dict, requires_approval: bool) -> dict:
    if requires_approval:
        approval_request_id = queue_for_human_approval(action)
        return {"status": "pending_approval", "approval_request_id": approval_request_id}
    return execute_action(action)
```


## Monitoring, evaluation, and error analysis


### Code block 12


```text
Current Foundry agents emit server-side traces that can be viewed in the Foundry portal once tracing is enabled. For application-side visibility, instrument your agent code with OpenTelemetry and export spans to Application Insights. Traces capture model calls, tool invocations, latency, token usage, and errors, which is the production-grade replacement for depending on classic list_run_steps(thread_id, run_id) calls.

# Package direction for current client-side tracing:
# pip install azure-ai-projects azure-identity opentelemetry-sdk azure-core-tracing-opentelemetry
#
# Configure the project's tracing / Application Insights connection, then run
# the same Responses calls shown above. Foundry and OpenTelemetry record the
# model/tool spans; inspect them in Foundry > Traces or Application Insights.

The operational lesson is unchanged: use the trace to locate the slow or failing step before optimizing. The SDK surface changed; the observability principle did not.
```


### Code block 13


```text
Current production evaluation works from observable evidence: response text, tool calls and outputs, citations, structured state, and OpenTelemetry traces. Evaluate whether the final decision follows from those observable artifacts, and use Foundry trace-based evaluation when you need a repeatable post-deployment dataset.

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


### Code block 14


```python
def process_claim_end_to_end(intake_audio_path: str) -> dict:
    intake_result = process_claim_intake(intake_audio_path)  # Chapter 2's tool
    if intake_result["status"] == "escalated":
        approval_request_id = queue_for_human_approval({"reason": intake_result["reason"], "priority": "high"})
        return {"status": "pending_approval", "approval_request_id": approval_request_id}
```


### Code block 15


```text
    routing_result = supervisor_orchestrate(intake_result["extraction"])  # research -> summarize -> route
```


### Code block 16


```text
    requires_approval = intake_result["extraction"]["requires_human_review"]
    return execute_action_with_approval(routing_result, requires_approval)
```
