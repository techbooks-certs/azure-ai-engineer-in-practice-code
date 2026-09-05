# Chapter 2 — Text Analysis and Speech for Agentic Systems

> These code blocks are reproduced from the book in the order they appear. Read the surrounding book section for setup, dependencies, interpretation, and caveats.


## Extracting structured JSON via generative prompting


### Code block 1


```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI
import json

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://ai.azure.com/.default"
)

# Current Azure OpenAI v1 endpoint; no api_version parameter is required.
client = OpenAI(
    base_url="https://aif-ai103-labs.openai.azure.com/openai/v1/",
    api_key=token_provider,
)
intake_note = (
```


### Code block 2


```python
from pydantic import BaseModel
from typing import Literal

class ClaimExtraction(BaseModel):
    claim_type: Literal["property", "auto", "liability", "other"]
    incident_date: str | None = None
    requires_human_review: bool
    review_reason: str

response = client.responses.parse(
    model="chat-primary",
    instructions=(
        "Extract claim fields. Flag for human review if the claim amount is high "
        "or the policyholder has a recent prior claim."
    ),
    input=intake_note,
    text_format=ClaimExtraction,
)
result = response.output_parsed.model_dump()
print(json.dumps(result, indent=2))
```


### Code block 3


```python
def extract_claim_fields(note_text: str) -> dict:
    response = client.responses.parse(
        model="chat-primary",
        instructions=(
            "Extract claim fields. Flag for human review if the claim amount is high "
            "or the policyholder has a recent prior claim."
        ),
        input=note_text,
        text_format=ClaimExtraction,
    )
    return response.output_parsed.model_dump()

result = extract_claim_fields(intake_note)
print(json.dumps(result, indent=2))
```


## Detecting sentiment, tone, and sensitive content


### Code block 4


```python
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import AnalyzeTextOptions
```


### Code block 5


```text
safety_client = ContentSafetyClient(
    endpoint="https://aif-ai103-labs.cognitiveservices.azure.com/",
    credential=credential,
```


### Code block 6


```python
analysis = safety_client.analyze_text(
```


### Code block 7


```text
for category in analysis.categories_analysis:
    print(f"{category.category}: severity {category.severity}")
```


## Nuanced sentiment and tone beyond a three-way label


### Code block 8


```text
tone_response = client.chat.completions.create(
    model="chat-primary",
    messages=[
```


### Code block 9


```text
print(tone_response.choices[0].message.content)
```


## Customizing outputs for domain-specific tasks


### Code block 10


```text
compliance_summary = client.chat.completions.create(
    model="chat-primary",
    messages=[
```


## Translation: Foundry Tool connection vs. LLM-powered flow


### Code block 11


```python
translated = client.chat.completions.create(
    model="chat-primary",
    messages=[
```


### Code block 12


```text
print(translated.choices[0].message.content)
```


## Speech as an agent modality, not a standalone transcription service


### Code block 13


```bash
az cognitiveservices account create \
  --name spch-ai103-labs \
  --resource-group rg-ai103-labs \
  --kind SpeechServices \
  --sku S0 \
```


### Code block 14


```bash
az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Cognitive Services Speech User" \
```


### Code block 15


```python
import azure.cognitiveservices.speech as speechsdk

SPEECH_ENDPOINT = "https://spch-ai103-labs.cognitiveservices.azure.com/"

def _speech_config() -> speechsdk.SpeechConfig:
    # Current Speech SDK supports TokenCredential directly for recognizer/synthesizer.
    return speechsdk.SpeechConfig(
        token_credential=credential,
        endpoint=SPEECH_ENDPOINT,
    )

def transcribe(audio_file_path: str) -> str:
    recognizer = speechsdk.SpeechRecognizer(
        speech_config=_speech_config(),
        audio_config=speechsdk.audio.AudioConfig(filename=audio_file_path),
    )
    result = recognizer.recognize_once()
    if result.reason != speechsdk.ResultReason.RecognizedSpeech:
        raise RuntimeError(f"Speech recognition failed: {result.reason}")
    return result.text

def speak(text: str) -> None:
    synthesizer = speechsdk.SpeechSynthesizer(speech_config=_speech_config())
    result = synthesizer.speak_text_async(text).get()
    if result.reason != speechsdk.ResultReason.SynthesizingAudioCompleted:
        raise RuntimeError(f"Speech synthesis failed: {result.reason}")
# Exercise both functions against the opening scenario's intake call
transcript = transcribe("intake_call.wav")
```


## Translating spoken audio


### Code block 16


```python
def transcribe_and_translate(audio_file_path: str, target_language: str = "en") -> str:
    """Speech-to-text with translation for a non-English caller."""
    translation_config = speechsdk.translation.SpeechTranslationConfig(
        token_credential=credential,
        endpoint=SPEECH_ENDPOINT,
    )
    translation_config.speech_recognition_language = "es-ES"
    translation_config.add_target_language(target_language)

    audio_config = speechsdk.audio.AudioConfig(filename=audio_file_path)
    recognizer = speechsdk.translation.TranslationRecognizer(
        translation_config=translation_config,
        audio_config=audio_config,
    )
    result = recognizer.recognize_once()
    if result.reason != speechsdk.ResultReason.TranslatedSpeech:
        raise RuntimeError(f"Speech translation failed: {result.reason}")
    return result.translations[target_language]
```


## Hands-on lab: the full text-and-speech agent tool


### Code block 17


```text
LIVE_ESCALATION_SEVERITY = 4  # Example policy threshold for this fictional claims workflow.
def process_claim_intake(audio_file_path: str) -> dict:
    transcript = transcribe(audio_file_path)  # speech-to-text, as above
```


### Code block 18


```text
    safety_result = safety_client.analyze_text(AnalyzeTextOptions(text=transcript))
    if any(cat.severity >= LIVE_ESCALATION_SEVERITY for cat in safety_result.categories_analysis):
        return {"status": "escalated", "reason": "sensitive content detected", "transcript": transcript}
```


### Code block 19


```text
    extracted = extract_claim_fields(transcript)  # JSON-schema extraction, as above
```


### Code block 20


```text
    confirmation_text = (
```


### Code block 21


```text
    return {"status": "processed", "extraction": extracted, "transcript": transcript}
```
