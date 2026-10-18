# Chapter 5 — RAG and Information Extraction

These examples follow the book in chapter order. Read the surrounding section for prerequisites, configuration, and security guidance. Some blocks show example output or partial commands rather than standalone scripts.


## Chunking, embeddings, and vector stores: the ingestion foundation

### Code block 1

```python
def chunk_document(text: str, target_tokens: int = 400, overlap_tokens: int = 50) -> list[str]:
    paragraphs = text.split("\n\n")
    chunks, current = [], ""
    for para in paragraphs:
        if len(current) + len(para) > target_tokens * 4:  # ~4 chars/token, rough estimate
            if current.strip():  # guard against flushing an empty chunk when the very first
                chunks.append(current.strip())  # paragraph alone already exceeds the target size
            current = current[-(overlap_tokens * 4):] + para
        else:
            current += "\n\n" + para
    if current.strip():
        chunks.append(current.strip())
    return chunks
```

### Code block 2

```bash
az cognitiveservices account deployment create \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --deployment-name embed-primary \
  --model-name text-embedding-3-large \
  --model-version "1" \
  --model-format OpenAI \
  --sku-capacity 5 \
  --sku-name GlobalStandard
```

### Code block 3

```bash
az search service create \
  --name srch-ai103-labs \
  --resource-group rg-ai103-labs \
  --sku standard \
  --location eastus2 \
  --auth-options aadOrApiKey \
  --aad-auth-failure-mode http401WithBearerChallenge
 
az role assignment create \
--assignee "$IDENTITY_PRINCIPAL_ID" \
--role "Search Service Contributor" \
--scope "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.Search/searchServices/srch-ai103-labs"

az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Search Index Data Contributor" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.Search/searchServices/srch-ai103-labs"
```

### Code block 4

```python
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
    SearchIndex, SimpleField, SearchableField, SearchField, SearchFieldDataType,
    VectorSearch, HnswAlgorithmConfiguration, VectorSearchProfile,
    SemanticConfiguration, SemanticPrioritizedFields, SemanticField, SemanticSearch,
)
 
index_client = SearchIndexClient(endpoint="https://srch-ai103-labs.search.windows.net", credential=credential)
 
index = SearchIndex(
    name="claims-policy-docs",
    fields=[
        SimpleField(name="id", type=SearchFieldDataType.String, key=True),
        SearchableField(name="content", type=SearchFieldDataType.String),
        SearchField(
            name="content_vector", type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
            searchable=True, vector_search_dimensions=3072, vector_search_profile_name="default-vector-profile",
        ),
        SimpleField(name="source_document", type=SearchFieldDataType.String, filterable=True),
        SimpleField(name="doc_type", type=SearchFieldDataType.String, filterable=True),
        SimpleField(name="section_type", type=SearchFieldDataType.String, filterable=True),
        SimpleField(name="flagged", type=SearchFieldDataType.Boolean, filterable=True),
        SimpleField(name="contains_pii", type=SearchFieldDataType.Boolean, filterable=True),
    ],
    vector_search=VectorSearch(
        algorithms=[HnswAlgorithmConfiguration(name="default-hnsw")],
        profiles=[VectorSearchProfile(name="default-vector-profile", algorithm_configuration_name="default-hnsw")],
    ),
    semantic_search=SemanticSearch(configurations=[
        SemanticConfiguration(
            name="claims-docs-semantic-config",
            prioritized_fields=SemanticPrioritizedFields(content_fields=[SemanticField(field_name="content")]),
        )
    ]),
)
index_client.create_or_update_index(index)
```


## Configuring semantic, hybrid, and vector search

### Code block 5

```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
 
search_client = SearchClient(
    endpoint="https://srch-ai103-labs.search.windows.net",
    index_name="claims-policy-docs",
    credential=credential,
)
 
query_text = "what's the coverage limit for water damage from a burst pipe"
query_vector = client.embeddings.create(model="embed-primary", input=query_text).data[0].embedding
 
results = search_client.search(
    search_text=query_text,  # lexical half of hybrid search
    vector_queries=[VectorizedQuery(vector=query_vector, k_nearest_neighbors=5, fields="content_vector")],
    query_type="semantic",
    semantic_configuration_name="claims-docs-semantic-config",
    filter="flagged eq false and contains_pii eq false",
    top=5,
)
```

### Code block 6

```text
1. "Water damage from sudden and accidental discharge, including burst
   pipes, is covered up to the policy's dwelling coverage limit, subject to a
   $1,000 deductible..."
2. "Gradual water damage from long-term leaks or seepage is
   excluded from standard coverage..."
```


## Ingesting content: documents, images, audio, and video

### Code block 7

```python
from azure.ai.contentunderstanding.models import (
    ContentAnalyzer,
    ContentFieldDefinition,
    ContentFieldSchema,
    ContentFieldType,
)

policy_schema = ContentFieldSchema(
    name="policy_schema",
    description="Structured policy-document fields",
    fields={
        "section_heading": ContentFieldDefinition(type=ContentFieldType.STRING),
        "coverage_limits": ContentFieldDefinition(type=ContentFieldType.STRING),
        "exclusions": ContentFieldDefinition(type=ContentFieldType.STRING),
    },
)

policy_analyzer = ContentAnalyzer(
    base_analyzer_id="prebuilt-document",
    description="Policy-document analyzer for RAG ingestion",
    field_schema=policy_schema,
    models={
        "completion": "gpt-5.2",
        "embedding": "text-embedding-3-large",
    },
)

cu_client.begin_create_analyzer(
    analyzer_id="policy-doc-analyzer",
    resource=policy_analyzer,
).result()
import re
 
def sanitize_search_key(raw: str) -> str:
    # Azure AI Search document keys allow only letters, digits, -, _, and = —
    # a filename's "." and other punctuation must be stripped or replaced first.
    return re.sub(r"[^A-Za-z0-9_\-=]", "_", raw)
 
# Define enrich_chunk() below before invoking ingest_document(); Python resolves the name at call time.
def ingest_document(document_url: str, doc_type: str) -> list[dict]:
    analysis = cu_client.begin_analyze(analyzer_id="policy-doc-analyzer", inputs=[AnalysisInput(url=document_url)]).result()
    extracted_text = analysis.contents[0].markdown  # structured markdown output, not raw OCR text
    chunks = chunk_document(extracted_text)
    doc_key = sanitize_search_key(document_url.split("/")[-1])
    records = []
    for i, chunk in enumerate(chunks):
        vector = client.embeddings.create(model="embed-primary", input=chunk).data[0].embedding
        metadata = enrich_chunk(chunk, doc_type)
        records.append({
            "id": f"{doc_key}-chunk-{i}",
            "content": chunk,
            "content_vector": vector,
            "source_document": document_url,
            "doc_type": doc_type,
            "section_type": metadata["section_type"],
            "flagged": metadata["flagged"],
            "contains_pii": metadata["contains_pii"],
        })
    return records
```

### Code block 8

```python
def ingest_audio(audio_url: str, doc_type: str = "recorded_call") -> list[dict]:
    local_path = download_to_temp(audio_url)
    transcript = transcribe(local_path)  # Chapter 2's speech-to-text function
    chunks = chunk_document(transcript)
    doc_key = sanitize_search_key(audio_url.split("/")[-1])
    records = []
    for i, chunk in enumerate(chunks):
        vector = client.embeddings.create(model="embed-primary", input=chunk).data[0].embedding
        records.append({
            "id": f"{doc_key}-chunk-{i}",
            "content": chunk,
            "content_vector": vector,
            "source_document": audio_url,
            "doc_type": doc_type,
        })
    return records
```

### Code block 9

```python
from azure.ai.textanalytics import TextAnalyticsClient
 
language_client = TextAnalyticsClient(endpoint="https://aif-ai103-labs.cognitiveservices.azure.com/", credential=credential)
 
INGESTION_REVIEW_SEVERITY = 2  # Example policy threshold; not an Azure default.
def enrich_chunk(chunk: str, doc_type: str) -> dict:
    section_classification = client.chat.completions.create(
        model="chat-primary",
        messages=[
            {"role": "system", "content": "Classify this policy document excerpt's section as one of: coverage_terms, exclusions, definitions, procedures, other. Respond with only the label."},
            {"role": "user", "content": chunk},
        ],
    ).choices[0].message.content
 
    safety_result = safety_client.analyze_text(AnalyzeTextOptions(text=chunk))  # harm-category screening, not PII
    contains_flagged_content = any(cat.severity >= INGESTION_REVIEW_SEVERITY for cat in safety_result.categories_analysis)
 
    pii_result = language_client.recognize_pii_entities([chunk])[0]
    contains_pii = len(pii_result.entities) > 0
 
    return {"section_type": section_classification, "flagged": contains_flagged_content, "contains_pii": contains_pii}
```


## Implementing RAG in the application: prompts, citations, and hallucination reduction

### Code block 10

```python
retrieved_context = "\n\n".join(f"[Source {i+1}]: {r['content']}" for i, r in enumerate(results))
 
rag_response = client.chat.completions.create(
    model="chat-primary",
    messages=[
        {"role": "system", "content": (
            "Answer the question using only the provided sources. Cite the source "
            "number for every claim, like [Source 1]. If the sources don't contain "
            "enough information to answer confidently, say so explicitly rather than "
            "guessing or using outside knowledge."
        )},
        {"role": "user", "content": f"Sources:\n{retrieved_context}\n\nQuestion: {query_text}"},
    ],
)
```

### Code block 11

```text
Water damage from a burst pipe is covered up to your policy's dwelling coverage
limit, subject to a $1,000 deductible [Source 1]. Note that gradual leaks or
seepage — as opposed to a sudden burst — are excluded from standard coverage
[Source 2], so the claim's classification as sudden versus gradual damage
matters for coverage.
```


## Connecting retrieval pipelines to workflows and agent tools

### Code block 12

```python
def search_policy_documents(query: str) -> list[dict]:
    query_vector = client.embeddings.create(model="embed-primary", input=query).data[0].embedding
    results = search_client.search(
        search_text=query, vector_queries=[VectorizedQuery(vector=query_vector, k_nearest_neighbors=5, fields="content_vector")],
        query_type="semantic", semantic_configuration_name="claims-docs-semantic-config",
        filter="flagged eq false and contains_pii eq false", top=5,
    )
    return [{"content": r["content"], "source": r["source_document"]} for r in results]
```


## Evaluating RAG quality

### Code block 13

```python
import os
from azure.ai.evaluation import GroundednessEvaluator, RelevanceEvaluator

groundedness_eval = GroundednessEvaluator(
    model_config={
        "azure_endpoint": os.environ["AZURE_OPENAI_ENDPOINT"],
        "azure_deployment": "chat-primary",
    },
    credential=credential,
)

result = groundedness_eval(
    query=query_text,
    response=rag_response.choices[0].message.content,
    context=retrieved_context,
)
print(result)
```

### Code block 14

```json
{"groundedness": 4.5, "groundedness_reason": "The response's claims are directly supported by Source 1 and Source 2, with appropriate citation of each."}
```

### Code block 15

```python
fabrication_check = client.chat.completions.create(
    model="chat-primary",
    messages=[{
        "role": "system",
        "content": (
            "Given the sources and the answer below, list any specific factual "
            "claim in the answer that is NOT directly supported by the sources. "
            "If every claim is supported, respond with 'none'."
        ),
    }, {
        "role": "user",
        "content": f"Sources:\n{retrieved_context}\n\nAnswer:\n{rag_response.choices[0].message.content}",
    }],
)
```


## Hands-on lab: the full ingestion-to-answer pipeline

### Code block 16

```python
def build_and_query_rag_pipeline(document_urls: list[str], question: str, evaluate_inline: bool = False) -> dict:
    all_records = []
    for url in document_urls:
        all_records.extend(ingest_document(url, doc_type="policy"))
    search_client.upload_documents(documents=all_records)  # index the ingested, embedded chunks
 
    query_vector = client.embeddings.create(model="embed-primary", input=question).data[0].embedding
    results = search_client.search(
        search_text=question,
        vector_queries=[VectorizedQuery(vector=query_vector, k_nearest_neighbors=5, fields="content_vector")],
        query_type="semantic", semantic_configuration_name="claims-docs-semantic-config",
        filter="flagged eq false and contains_pii eq false", top=5,
    )
    context = "\n\n".join(f"[Source {i+1}]: {r['content']}" for i, r in enumerate(results))
 
    answer = client.chat.completions.create(
        model="chat-primary",
        messages=[
            {"role": "system", "content": "Answer using only the provided sources, citing each claim. Say so explicitly if the sources are insufficient."},
            {"role": "user", "content": f"Sources:\n{context}\n\nQuestion: {question}"},
        ],
    )
    groundedness = (
        groundedness_eval(query=question, response=answer.choices[0].message.content, context=context)
        if evaluate_inline else None
    )
    return {"answer": answer.choices[0].message.content, "groundedness": groundedness}
```


## Scaling and optimizing: what production adds

### Code block 17

```python
import hashlib
 
query_cache: dict[str, list[dict]] = {}  # illustrative in-memory cache; production uses Redis or similar
 
def cached_search(query: str) -> list[dict]:
    cache_key = hashlib.sha256(query.strip().lower().encode()).hexdigest()
    if cache_key in query_cache:
        return query_cache[cache_key]
    results = search_policy_documents(query)
    query_cache[cache_key] = results
    return results
```

### Code block 18

```python
rewrite_response = client.chat.completions.create(
    model="chat-primary",
    messages=[
        {"role": "system", "content": "Rewrite the user's follow-up question as a standalone search query, incorporating context from the conversation history."},
        {"role": "user", "content": "Previous question: What's the coverage limit for water damage?\nFollow-up: What about auto claims?"},
    ],
)
```

