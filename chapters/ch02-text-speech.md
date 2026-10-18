# Chapter 2 — Text and Speech

These examples follow the book in chapter order. Read the surrounding section for prerequisites, configuration, and security guidance. Some blocks show example output or partial commands rather than standalone scripts.


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
    "Customer reports a kitchen pipe burst on 2026-08-14, water damage to "
    "flooring and cabinetry, estimated at $18,400. Second claim from this "
    "policyholder in six months."
)
 
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

### Code block 2

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

### Code block 3

```json
{
  "claim_type": "property",
  "incident_date": "2026-08-14",
  "requires_human_review": true,
  "review_reason": "High estimated damage amount combined with a second claim from the same policyholder within six months."
}
```


## Detecting sentiment, tone, and sensitive content

### Code block 4

```python
from azure.ai.contentsafety import ContentSafetyClient
from azure.ai.contentsafety.models import AnalyzeTextOptions
 
safety_client = ContentSafetyClient(
    endpoint="https://aif-ai103-labs.cognitiveservices.azure.com/",
    credential=credential,
)
 
analysis = safety_client.analyze_text(
    AnalyzeTextOptions(text=intake_note)
)
for category in analysis.categories_analysis:
    print(f"{category.category}: severity {category.severity}")
```

### Code block 5

```text
Hate: severity 0
SelfHarm: severity 0
Sexual: severity 0
Violence: severity 0
```


## Nuanced sentiment and tone beyond a three-way label

### Code block 6

```python
tone_response = client.chat.completions.create(
    model="chat-primary",
    messages=[
        {"role": "system", "content": (
            "Classify the policyholder's tone as one of: neutral, frustrated, "
            "or escalation_risk. escalation_risk means language suggesting legal "
            "action, regulatory complaint, or public/social media escalation. "
            "Respond with only the label."
        )},
        {"role": "user", "content": (
            "This is the third time I've called about this claim. If it isn't "
            "resolved by Friday I'm contacting the insurance commissioner."
        )},
    ],
)
print(tone_response.choices[0].message.content)
```

### Code block 7

```text
escalation_risk
```


## Customizing outputs for domain-specific tasks

### Code block 8

```python
compliance_summary = client.chat.completions.create(
    model="chat-primary",
    messages=[
        {"role": "system", "content": (
            "Summarize this claims call transcript for a compliance audit log. "
            "The summary must: (1) state whether the caller was informed of their "
            "right to appeal, (2) note if any promise of a specific payout amount "
            "was made — agents are not authorized to guarantee amounts, and (3) "
            "flag if the call exceeded 15 minutes without documented resolution. "
            "Do not include the caller's full name; refer to them as 'the caller'."
        )},
        {"role": "user", "content": transcript},
    ],
)
```


## Translation: Foundry Tool connection vs. LLM-powered flow

### Code block 9

```python
translated = client.chat.completions.create(
    model="chat-primary",
    messages=[
        {"role": "system", "content": "Translate the user's message to English. Respond with only the translation, no commentary."},
        {"role": "user", "content": "Un tuyau a éclaté dans la cuisine le 14 août, causant des dégâts d'eau importants."},
    ],
)
print(translated.choices[0].message.content)
```

### Code block 10

```text
A pipe burst in the kitchen on August 14, causing significant water damage.
```


## Speech as an agent modality, not a standalone transcription service

### Code block 11

```bash
az cognitiveservices account create \
  --name spch-ai103-labs \
  --resource-group rg-ai103-labs \
  --kind SpeechServices \
  --sku S0 \
  --location eastus2
 
az role assignment create \
  --assignee "$IDENTITY_PRINCIPAL_ID" \
  --role "Cognitive Services Speech User" \
  --scope "/subscriptions/<sub-id>/resourceGroups/rg-ai103-labs/providers/Microsoft.CognitiveServices/accounts/spch-ai103-labs"
```

### Code block 12

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
speak(
    "I've logged your claim as property damage from a water incident. "
    "Because of the estimated amount, an adjuster will review it within one business day."
)
```

### Code block 13

```text
Transcript: "There was a pipe burst in my kitchen on August fourteenth, water everywhere,
probably eighteen or nineteen thousand dollars in damage."
[audio output synthesized and played/saved]
```


## Translating spoken audio

### Code block 14

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

### Code block 15

```python
LIVE_ESCALATION_SEVERITY = 4  # Example policy threshold for this fictional claims workflow.
def process_claim_intake(audio_file_path: str) -> dict:
    transcript = transcribe(audio_file_path)  # speech-to-text, as above
 
    safety_result = safety_client.analyze_text(AnalyzeTextOptions(text=transcript))
    if any(cat.severity >= LIVE_ESCALATION_SEVERITY for cat in safety_result.categories_analysis):
        return {"status": "escalated", "reason": "sensitive content detected", "transcript": transcript}
 
    extracted = extract_claim_fields(transcript)  # JSON-schema extraction, as above
 
    confirmation_text = (
        f"I've logged your claim as {extracted['claim_type']} damage. "
        + ("An adjuster will review it shortly." if extracted["requires_human_review"]
           else "It has been queued for standard processing.")
    )
    speak(confirmation_text)  # text-to-speech, as above
 
    return {"status": "processed", "extraction": extracted, "transcript": transcript}
```

