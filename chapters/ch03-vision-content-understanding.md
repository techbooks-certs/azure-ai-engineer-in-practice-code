# Chapter 3 — Vision and Multimodal Generation

> These code blocks are reproduced from the book in the order they appear. Read the surrounding book section for setup, dependencies, interpretation, and caveats.


## Generating images from prompts and reference media


### Code block 1


```bash
az cognitiveservices account deployment create \
  --name aif-ai103-labs \
  --resource-group rg-ai103-labs \
  --deployment-name image-gen-primary \
```


### Code block 2


```text
  --model-format OpenAI \
  --sku-capacity 1 \
```


### Code block 3


```python
import base64

image_response = client.images.generate(
    model="image-gen-primary",
    prompt=(
        "A clean, professional product photograph of a kitchen cabinet with "
        "visible water-damage staining along the lower edge, neutral studio "
        "lighting, no people, no text or watermarks."
    ),
    size="1024x1024",
    quality="medium",
)
image_bytes = base64.b64decode(image_response.data[0].b64_json)
with open("cabinet_reference.png", "wb") as f:
    f.write(image_bytes)
```


## Editing: inpainting, masks, and prompt-driven modification


### Code block 4


```text
edited = client.images.edit(
    model="image-gen-primary",
    image=open("cabinet_original.png", "rb"),
    mask=open("cabinet_damage_mask.png", "rb"),
    prompt="Add visible water staining and swelling to the cabinet's lower edge, consistent with pipe-burst water damage.",
    size="1024x1024",
```


### Code block 5


```text
# Current Sora video workflows are asynchronous jobs. Use the documented REST
# video endpoint for the deployed Sora model, then poll until the job reaches
# a terminal state before downloading the result. The exact request body
# depends on whether the input is text-only, image+text, or video+text.
#
# Pseudocode:
# job = create_sora_video_job(prompt=..., input_video="intake_walkthrough.mp4")
# while job.status in {"queued", "running"}:
#     time.sleep(2)
#     job = get_sora_video_job(job.id)
# download_video(job.id, "edited_walkthrough.mp4")
```


## Multimodal understanding: captions, visual Q&A, and accessibility


### Code block 6


```text
caption_response = client.chat.completions.create(
    model="chat-primary",
    messages=[
```


### Code block 7


```text
alt_text_response = client.chat.completions.create(
    model="chat-primary",
    messages=[{
```


## Identifying objects, components, and regions


### Code block 8


```text
identification_schema = {
```


### Code block 9


```text
identification = client.chat.completions.create(
    model="chat-primary",
    messages=[{"role": "user", "content": [
```


### Code block 10


```python
    response_format={"type": "json_schema", "json_schema": {"name": "component_id", "schema": identification_schema, "strict": True}},
```


## Azure Content Understanding: extraction from images and video


### Code block 11


```python
from azure.ai.contentunderstanding import ContentUnderstandingClient
from azure.ai.contentunderstanding.models import (
    ContentAnalyzer,
    ContentFieldDefinition,
    ContentFieldSchema,
    ContentFieldType,
    ContentAnalyzerConfig,
)

cu_client = ContentUnderstandingClient(
    endpoint="https://aif-ai103-labs.cognitiveservices.azure.com/",
    credential=credential,
)

field_schema = ContentFieldSchema(
    name="claims_intake_schema",
    description="Structured fields from a claims intake form",
    fields={
        "policy_number": ContentFieldDefinition(type=ContentFieldType.STRING),
        "claimant_name": ContentFieldDefinition(type=ContentFieldType.STRING),
        "incident_date": ContentFieldDefinition(type=ContentFieldType.DATE),
    },
)

analyzer = ContentAnalyzer(
    base_analyzer_id="prebuilt-document",
    description="Claims intake form extractor",
    field_schema=field_schema,
    config=ContentAnalyzerConfig(
        enable_ocr=True,
        enable_layout=True,
        return_details=True,
        estimate_field_source_and_confidence=True,
    ),
    models={
        "completion": "gpt-5.2",
        "embedding": "text-embedding-3-large",
    },
)

poller = cu_client.begin_create_analyzer(
    analyzer_id="claims-intake-form-extractor",
    resource=analyzer,
)
poller.result()
```


### Code block 12


```python
from azure.ai.contentunderstanding.models import AnalysisInput

poller = cu_client.begin_analyze(
    analyzer_id="claims-intake-form-extractor",
    inputs=[AnalysisInput(url=scanned_form_url)],
)
analysis = poller.result()
content = analysis.contents[0]
extracted_fields = {
    name: {"value": field.value, "confidence": field.confidence}
    for name, field in content.fields.items()
}
```


## Responsible AI for multimodal content


### Code block 13


```python
def enforce_visual_policy(image_url: str) -> dict:
    policy_check = client.chat.completions.create(
        model="chat-primary",
        messages=[{"role": "user", "content": [
```


### Code block 14


```text
    return policy_check.choices[0].message.content
```


## Hands-on lab: from prompt to accessible, adjuster-ready asset


### Code block 15


```python
def generate_and_describe_damage_reference(damage_description: str) -> dict:
    generated = client.images.generate(
        model="image-gen-primary",
        prompt=f"Professional documentation photo of {damage_description}, neutral lighting, no text.",
        size="1024x1024",
        quality="medium",
    )
    b64 = generated.data[0].b64_json
    image_bytes = base64.b64decode(b64)
    image_path = "damage_reference.png"
    with open(image_path, "wb") as f:
        f.write(image_bytes)

    image_data_url = f"data:image/png;base64,{b64}"
    caption = client.chat.completions.create(
        model="chat-primary",
        messages=[{"role": "user", "content": [
            {"type": "text", "text": "Describe only observable damage for an adjuster's file note; do not infer cause or duration."},
            {"type": "image_url", "image_url": {"url": image_data_url}},
        ]}],
    ).choices[0].message.content

    alt_text = client.chat.completions.create(
        model="chat-primary",
        messages=[{"role": "user", "content": [
            {"type": "text", "text": "Write concise alt text under 125 characters for this image."},
            {"type": "image_url", "image_url": {"url": image_data_url}},
        ]}],
    ).choices[0].message.content

    return {"image_path": image_path, "adjuster_caption": caption, "alt_text": alt_text}
```
