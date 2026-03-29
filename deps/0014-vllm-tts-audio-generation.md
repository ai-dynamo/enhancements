# TTS Audio Generation Endpoint via vLLM Omni

**Status**: Draft

**Authors**: [@hatemfaheem](https://github.com/hatemfaheem)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [Ayush Agarwal](https://github.com/ayushag-nv) - Based on vllm omni activity.

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**:

[Tracking Issue #7664](https://github.com/ai-dynamo/dynamo/issues/7664)
[Draft PR #7661](https://github.com/ai-dynamo/dynamo/pull/7661)

## Summary

Add a Text-to-Speech (TTS) audio generation endpoint (`POST /v1/audio/speech`) to Dynamo, powered by vLLM Omni. The endpoint accepts text input and returns a complete WAV or PCM audio file, following the same architectural patterns already established by the image and video generation modalities. This proposal covers the first working version (V1) that delivers complete audio responses, while explicitly laying the groundwork for future streaming and additional codecs.

## Motivation

Dynamo supports multimodal generation through its vLLM Omni integration, with image generation (`/v1/images/generations`) and video generation (`/v1/videos`) already shipping. TTS is a natural next modality: vLLM Omni already supports TTS models (e.g., Qwen3-TTS), and the Dynamo codebase contains partial scaffolding for audio (protocol stubs, endpoint type enum, router placeholders) but no working end-to-end implementation.

Users deploying TTS models through Dynamo today have no supported path. Adding TTS completes the multimodal generation story and opens the door for voice-enabled applications built on Dynamo's inference platform.

My team has a working version with `Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice` ([link](https://docs.vllm.ai/projects/vllm-omni/en/stable/user_guide/examples/online_serving/qwen3_tts)) and we would love to contribute this work.

### Goals

- Deliver a working, OpenAI-compatible TTS endpoint that follows Dynamo's established multimodal architecture
- Maintain strict consistency with the existing image and video modality patterns (protocol types, transport, handler structure, aggregation)
- Support the most common TTS use case: text in, audio file out
- Design the implementation so that streaming, additional codecs, and new TTS models can be added incrementally without architectural changes

### Non-Goals

- **Streaming audio to clients:** deferred to a future proposal
- **Voice cloning:** deferred to Phase 2
- **Additional audio codecs:** beyond WAV and PCM (MP3, FLAC, Opus, AAC, OGG) deferred to later phase
- **PyO3 bindings:** would be addressed across all modalities together, not audio-specific

## Requirements

### REQ 1 OpenAI API Compatibility

The endpoint **MUST** conform to OpenAI's `POST /v1/audio/speech` API shape:

https://developers.openai.com/api/reference/resources/audio/subresources/speech/methods/create

```json
{
  "model": "model-name",
  "input": "Text to synthesize",
  "voice": "alloy",
  "response_format": "wav",
  "instructions": "voice control",
  "speed": 1.0
}
```

The response **MUST** be raw binary audio with the appropriate `Content-Type` header (e.g., `audio/wav`).

### REQ 2 Architectural Consistency

The implementation **MUST** follow the same patterns as image and video generation:
- Dual protocol types (Rust struct + Python Pydantic model)
- Base64-encoded audio transport across the Python-Rust boundary
- Stream aggregation via a dedicated aggregator
- Non-streaming response to the client (single complete file)
- Format encoding performed in Python
- NVIDIA extensions via an `nvext` field on the request

### REQ 3 Supported Response Formats

V1 **MUST** support:
- `wav` (default): PCM wrapped in a RIFF/WAVE container
- `pcm`: raw PCM bytes with no container

Unsupported format values **MUST** be rejected with an error, not silently downgraded.

### REQ 4 Model Discovery and Registration

Audio models **MUST** be discoverable and registerable through the existing `ModelManager` / `WorkerSet` infrastructure, following the same namespace pattern used by images and videos.

### REQ 5 Metrics and Observability

The audio endpoint **MUST** report the same categories of metrics as other generation endpoints.

### REQ 6 Model-Agnostic Design

The Python handler's TTS logic (prompt construction, prompt length estimation, audio extraction) **MUST** be structured so that supporting a new TTS model does not require modifying shared handler code. Model-specific behavior (e.g., prompt format, estimation method) **SHOULD** be encapsulated behind an abstraction that can be extended per model.

# Proposal

## Overview

The TTS endpoint follows the established Dynamo multimodal generation architecture. The request enters through a Rust HTTP handler, crosses into Python via the streaming engine boundary, is processed by the vLLM Omni handler which drives the TTS model, and the resulting audio flows back through the same boundary as a base64-encoded payload before being decoded and returned as raw binary to the client.

## Architecture

```
                        RUST                              PYTHON
                  ──────────────                    ────────────────

  Client ──POST──> Axum Router
                   /v1/audio/speech
                        |
                   HTTP Handler
                        |
                   NvCreateAudioSpeechRequest
                        |
                   ModelManager -> WorkerSet
                   -> audios_engine
                        |
                   engine.generate()  ────────────>  OmniHandler.generate()
                                                           |
                                                      Parse request type
                                                      -> AudioGeneration
                                                           |
                                                      Build TTS model inputs
                                                      (model-specific prompt
                                                       construction)
                                                           |
                                                      vLLM engine generates
                                                      audio (cumulative PCM)
                                                           |
                                                      Extract & format audio
                                                      -> WAV or PCM + base64
                                                           |
                   <──── NvAudiosResponse ────────  yield NvAudiosResponse
                   (stream, base64 audio)
                        |
                   Aggregator
                   -> fold stream (take latest)
                        |
                   base64 decode -> raw bytes
                   Set Content-Type header
                        |
  Client <──200──  raw binary audio (WAV/PCM)
```

This architecture is intentionally identical to the image and video flows. The only tts-specific differences occur inside the Python handler (prompt construction, audio extraction, format encoding).

## Protocol Types

**Request** (`NvCreateAudioSpeechRequest`):

```json
{
  "model": "Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice",
  "input": "Hello, welcome to Dynamo.",
  "voice": "alloy",
  "response_format": "wav",
  "speed": 1.0,
  "nvext": {
    "language": "en",
    "max_new_tokens": 2048,
    "seed": 42
  }
}
```

**Response** (`NvAudiosResponse`) -- internal shape crossing the Python-Rust boundary:

```json
{
  "audio_b64": "<base64-encoded audio bytes>",
  "content_type": "audio/wav",
  "model": "Qwen/Qwen3-TTS-12Hz-1.7B-CustomVoice",
  "created": 1711500000
}
```

The Rust HTTP handler decodes `audio_b64` and returns raw binary audio to the client with the appropriate `Content-Type` header. The client never sees this JSON shape.

Matching Python Pydantic models mirror the Rust structs field-for-field, consistent with how image and video protocols are structured.

## Key Design Decisions

### Version of vllm-omni

The draft version has been tested with `v0.17.0rc1`. Although, we noticed that later version(s) of vllm-omni had multiple improvements to TTFB and TTFA. We ran a couple quick experiments on vllm-omni separetly (outside Dynamo i.e. vllm serve) and tested with a simple web app that streams the audio response. We saw:

* `v0.17.0rc1`: We noticed TTFA of ~1.3-1.5 seconds.
* `latest ~0.18.0`: We noticed TTFA of ~270-320 ms.

This could be a later topic for phase 2/3 where we introduce streaming, as it would unblock ultra low latency TTS use cases.

#### Known Limitation: Quadratic Transfer Cost

vLLM currently sends cumulative audio instead of deltas, so we end up re-sending previously generated data on every step. That means the total data crossing the Python–Rust boundary grows roughly quadratically with the length of the audio. If the final output size is N over K steps, we’re effectively transferring about N * K / 2 bytes instead of just N.

For short TTS clips (a few seconds), this doesn’t really matter. But as audio gets longer, the overhead becomes significant, which is why delta-based output starts to make more sense. We could add a diffing layer in Dynamo, but it’s only really worth it if we also support client-side streaming, which we’re not planning for V1.

**Why N * K / 2 ?**

```
Total size of audio file is N bytes over K steps where

     N = a1 + a2 + a3 + ... + aK

As everytime we send the whole accumulated audio.

Total bytes transferred would be

     Total Sent = a1 + (a1​ + a2​) + (a1 ​+ a2 ​+ a3​) + ... + (a1​ + ... + aK​)

That simplifies to

     Total Sent = a1 K + a2 (K-1) + ... + aK (1)

Assuming all chunks (ai) are same size of (N / K)

     Total Sent = (N/K) . (1 + 2 + ... + K)

The above formula would simplify to

     Total Sent = N * (K + 1) / 2
     Total Sent =  N * K / 2            (For large K, K+1 ≈ K)
```

### HTTP Response: Raw Binary, Not JSON

Unlike images and videos which return JSON (with URL or b64_json), the audio endpoint returns raw binary audio with a `Content-Type` header. This matches the OpenAI `/v1/audio/speech` convention and is the natural format for audio data that clients want to save or play directly.

### Model-Specific Prompt Construction

TTS models do not consume text tokens directly the way LLMs do. Instead, each TTS model has its own prompt format that the Python handler must construct. For example, Qwen3-TTS expects a prompt dictionary with placeholder token IDs and a separate parameters dictionary. Other models (e.g., Voxtral TTS) may expect entirely different structures.

This model-specific logic prompt construction, prompt length estimation, and audio output extraction needs to be encapsulated so that adding a new TTS model means adding a new adapter, not modifying shared handler code. V1 ships with support for Qwen3-TTS as the first adapter, but the design should not hardcode assumptions about any single model's interface.

### Format Encoding in Python

WAV header construction and format encoding happen in Python, consistent with how images and videos handle their format encoding.

# Implementation Phases

## Phase 1: Core TTS Endpoint

**Supported:**

- `POST /v1/audio/speech` endpoint
- WAV and PCM response formats
- Voice selection via `voice` field
- Speed control via `speed` field
- Language selection via `nvext.language`
- Seed control via `nvext.seed`
- Token limit via `nvext.max_new_tokens`
- Full metrics and observability
- Model discovery and registration
- Qwen3-TTS as the first supported model

**Not Supported:**

- Streaming audio responses
- Voice cloning (ref_audio/ref_text)
- Additional codecs (MP3, FLAC, Opus, AAC, OGG)
- Instructions/style control

## Phase 2: Voice Cloning, Additional Codecs, and Multi-Model

Relatively small followup changes

- Voice cloning via `nvext.ref_audio` and `nvext.ref_text`
- Additional response formats (MP3, FLAC, etc.)
- Instructions/style control via `nvext.instructions`
- Additional TTS model adapters (e.g., Voxtral TTS)

## Phase 3: Streaming Audio (Future DEP)

A separate DEP may be required for audio streaming

# Alternate Solutions

## Alt 1: WAV Formatting in Rust

Move WAV header construction from Python to Rust. Python would return raw PCM bytes (base64-encoded), and the Rust HTTP handler would wrap them in a WAV container before responding.

**Pros:**
- Cleaner separation: Python handles inference, Rust handles HTTP encoding
- Rust has robust audio crates (e.g., `hound`)

**Cons:**
- Breaks consistency with image and video, which both do format encoding in Python
- Splits audio logic across two languages

**Reason Rejected:**
- Architectural consistency with images and videos outweighs the marginal benefits

## Alt 2: Shared Memory / Zero-Copy Transport

Replace the base64-encoded JSON transport between Python and Rust with shared memory buffers or PyO3 zero-copy bindings.

**Pros:**
N/A

**Cons:**
N/A

**Reason Rejected:**
- The existing codebase already has a TODO to replace Pydantic models with PyO3 bindings. Changing all modalities togther in the future would keep the code consistent.
- Audio V1 should not diverge from the established transport pattern

## Alt 3: Return JSON Response Instead of Raw Binary

Return the audio as a JSON object with a base64-encoded `audio` field (similar to how images can be returned as `b64_json`), instead of raw binary with a Content-Type header.

**Pros:**
- Consistent response shape with images (`b64_json` format)
- Could include metadata alongside the audio

**Cons:**
- Incompatible with OpenAI's `/v1/audio/speech` API, which returns raw binary
- Adds unnecessary encoding overhead for the client
- Most audio use cases want a playable file, not a JSON wrapper

**Reason Rejected:**
- OpenAI API compatibility is a primary goal
- OpenAI's endpoint returns raw binary audio, and clients expect this behavior

