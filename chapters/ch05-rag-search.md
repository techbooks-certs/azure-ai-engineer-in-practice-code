# Chapter 5 — RAG and Information Extraction Pipelines

> These code blocks are reproduced from the book in the order they appear. Read the surrounding book section for setup, dependencies, interpretation, and caveats.


## Chunking, embeddings, and vector stores: the ingestion foundation


### Code block 1


```python
def chunk_document(text: str, target_tokens: int = 400, overlap_tokens: int = 50) -> list[str]:
    paragraphs = text.split("\n\n")
```


### Code block 2


```text
    for para in paragraphs:
        if len(current) + len(para) > target_tokens * 4:  # ~4 chars/token, rough estimate
            if current.strip():  # guard against flushing an empty chunk when the very first
```


### Code block 3


```text
            current = current[-(overlap_tokens * 4):] + para
        else:
```


### Code block 4


```text
    if current.strip():
```


### Code block 5


```text
    return chunks
```


### Code block 6


```bash
az cognitiveservices account deployment create \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --deployment-name embed-primary \
  --model-name text-embedding-3-large \
```


### Code block 7


```text
  --model-format OpenAI \
  --sku-capacity 5 \
```


### Code block 8


```bash
az search service create \
  --name srch-ai103-labs \
  --resource-group rg-ai103-labs \
  --sku standard \
```


### Code block 9


```bash
az role assignment create \
--assignee "$IDENTITY_PRINCIPAL_ID" \
--role "Search Service Contributor" \
--scope "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.Search/searchServices/srch-ai103-labs"

az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Search Index Data Contributor" \
```


### Code block 10


```python
from azure.search.documents.indexes import SearchIndexClient
from azure.search.documents.indexes.models import (
```


### Code block 11


```text
index_client = SearchIndexClient(endpoint="https://srch-ai103-labs.search.windows.net", credential=credential)
```


### Code block 12


```python
index = SearchIndex(
    name="claims-policy-docs",
    fields=[
```


### Code block 13


```python
            name="content_vector", type=SearchFieldDataType.Collection(SearchFieldDataType.Single),
            searchable=True, vector_search_dimensions=3072, vector_search_profile_name="default-vector-profile",
```


### Code block 14


```text
    vector_search=VectorSearch(
        algorithms=[HnswAlgorithmConfiguration(name="default-hnsw")],
        profiles=[VectorSearchProfile(name="default-vector-profile", algorithm_configuration_name="default-hnsw")],
```


### Code block 15


```text
    semantic_search=SemanticSearch(configurations=[
```


### Code block 16


```text
            name="claims-docs-semantic-config",
            prioritized_fields=SemanticPrioritizedFields(content_fields=[SemanticField(field_name="content")]),
```


## Configuring semantic, hybrid, and vector search


### Code block 17


```python
from azure.search.documents import SearchClient
from azure.search.documents.models import VectorizedQuery
```


### Code block 18


```python
search_client = SearchClient(
    endpoint="https://srch-ai103-labs.search.windows.net",
    index_name="claims-policy-docs",
    credential=credential,
```


### Code block 19


```text
query_text = "what's the coverage limit for water damage from a burst pipe"
query_vector = client.embeddings.create(model="embed-primary", input=query_text).data[0].embedding
```


### Code block 20


```text
results = search_client.search(
    search_text=query_text,  # lexical half of hybrid search
    vector_queries=[VectorizedQuery(vector=query_vector, k_nearest_neighbors=5, fields="content_vector")],
    query_type="semantic",
    semantic_configuration_name="claims-docs-semantic-config",
    filter="flagged eq false and contains_pii eq false",
    top=5,
```


## Ingesting content: documents, images, audio, and video


### Code block 21


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
```


### Code block 22


```python
def sanitize_search_key(raw: str) -> str:
    # Azure AI Search document keys allow only letters, digits, -, _, and = —
    # a filename's "." and other punctuation must be stripped or replaced first.
    return re.sub(r"[^A-Za-z0-9_\-=]", "_", raw)
```


### Code block 23


```text
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
```


### Code block 24


```text
    return records
```


### Code block 25


```python
def ingest_audio(audio_url: str, doc_type: str = "recorded_call") -> list[dict]:
    local_path = download_to_temp(audio_url)
    transcript = transcribe(local_path)  # Chapter 2's speech-to-text function
    chunks = chunk_document(transcript)
    doc_key = sanitize_search_key(audio_url.split("/")[-1])
    records = []
    for i, chunk in enumerate(chunks):
        vector = client.embeddings.create(model="embed-primary", input=chunk).data[0].embedding
```


### Code block 26


```text
    return records
```


### Code block 27


```python
from azure.ai.textanalytics import TextAnalyticsClient
```


### Code block 28


```text
language_client = TextAnalyticsClient(endpoint="https://aif-ai103-labs.cognitiveservices.azure.com/", credential=credential)
```


### Code block 29


```text
INGESTION_REVIEW_SEVERITY = 2  # Example policy threshold; not an Azure default.
def enrich_chunk(chunk: str, doc_type: str) -> dict:
    section_classification = client.chat.completions.create(
        model="chat-primary",
        messages=[
```


### Code block 30


```text
    safety_result = safety_client.analyze_text(AnalyzeTextOptions(text=chunk))  # harm-category screening, not PII
    contains_flagged_content = any(cat.severity >= INGESTION_REVIEW_SEVERITY for cat in safety_result.categories_analysis)
```


### Code block 31


```text
    pii_result = language_client.recognize_pii_entities([chunk])[0]
    contains_pii = len(pii_result.entities) > 0
```


### Code block 32


```text
    return {"section_type": section_classification, "flagged": contains_flagged_content, "contains_pii": contains_pii}
```


## Implementing RAG in the application: prompts, citations, and hallucination reduction


### Code block 33


```python
retrieved_context = "\n\n".join(f"[Source {i+1}]: {r['content']}" for i, r in enumerate(results))
```


### Code block 34


```python
rag_response = client.chat.completions.create(
    model="chat-primary",
    messages=[
```


## Connecting retrieval pipelines to workflows and agent tools


### Code block 35


```python
def search_policy_documents(query: str) -> list[dict]:
    query_vector = client.embeddings.create(model="embed-primary", input=query).data[0].embedding
    results = search_client.search(
        search_text=query, vector_queries=[VectorizedQuery(vector=query_vector, k_nearest_neighbors=5, fields="content_vector")],
        query_type="semantic", semantic_configuration_name="claims-docs-semantic-config",
        filter="flagged eq false and contains_pii eq false", top=5,
```


### Code block 36


```text
    return [{"content": r["content"], "source": r["source_document"]} for r in results]
```


## Evaluating RAG quality


### Code block 37


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


### Code block 38


```text
fabrication_check = client.chat.completions.create(
    model="chat-primary",
    messages=[{
```


## Hands-on lab: the full ingestion-to-answer pipeline


### Code block 39


```python
def build_and_query_rag_pipeline(document_urls: list[str], question: str, evaluate_inline: bool = False) -> dict:
    all_records = []
    for url in document_urls:
```


### Code block 40


```python
    query_vector = client.embeddings.create(model="embed-primary", input=question).data[0].embedding
    results = search_client.search(
        search_text=question,
        vector_queries=[VectorizedQuery(vector=query_vector, k_nearest_neighbors=5, fields="content_vector")],
        query_type="semantic", semantic_configuration_name="claims-docs-semantic-config",
        filter="flagged eq false and contains_pii eq false", top=5,
```


### Code block 41


```text
    context = "\n\n".join(f"[Source {i+1}]: {r['content']}" for i, r in enumerate(results))
```


### Code block 42


```text
    answer = client.chat.completions.create(
        model="chat-primary",
        messages=[
```


### Code block 43


```python
    groundedness = (
        groundedness_eval(query=question, response=answer.choices[0].message.content, context=context)
        if evaluate_inline else None
    )
    return {"answer": answer.choices[0].message.content, "groundedness": groundedness}
```


## Scaling and optimizing: what production adds


### Code block 44


```python
import hashlib
```


### Code block 45


```python
def cached_search(query: str) -> list[dict]:
    cache_key = hashlib.sha256(query.strip().lower().encode()).hexdigest()
    if cache_key in query_cache:
        return query_cache[cache_key]
    results = search_policy_documents(query)
```


### Code block 46


```text
    return results
```


### Code block 47


```text
rewrite_response = client.chat.completions.create(
    model="chat-primary",
    messages=[
```
