# Make the Dynamo Rust EPP Extensible (PrepareData + Load Shedding)

**Status**: Draft

**Authors**: [atchernych](https://github.com/atchernych)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [TBD — assign a code owner / maintainer]

**Required Reviewers**: [TBD — inference-gateway / EPP owners]

**Review Date**: [TBD]

**Pull Request**: [TBD — link to this PR in ai-dynamo/enhancements]

**Implementation PR / Tracking Issue**: [TBD — link to ai-dynamo/dynamo tracking issue]

# Summary

The Dynamo Rust EPP (`dynamo-ext-proc`) today is effectively two things: the
Envoy `ext_proc` server and a single, hard-wired router that tokenizes the
prompt and picks a worker. There are no extension points. The Go EPP — which
follows the GAIE / llm-d pipeline and supports plugins — is being deprecated, so
the Rust EPP becomes the single EPP and must absorb the extensibility that
previously only existed in Go.

This proposal adds a minimal, two-hook extension model to the Rust EPP so users
can supply their own code at exactly two points: **PrepareData** (per-request
data preparation before scheduling; tokenization is the first use) and **load
shedding / admission** (reject a request under overload before scheduling).
Everything else — parsing, the KV-aware scheduler/picker, and bookkeeping —
stays built in. We deliberately do not reproduce the full llm-d pipeline.

# Motivation

The Go EPP is being deprecated, and its plugin-based extensibility goes away with
it. The Rust EPP cannot currently be extended: all logic lives in one router
implementation, so users who want custom tokenization or custom load-shedding
policy must fork it. Two concrete needs drive this proposal:

- **Tokenization must be swappable.** Different deployments tokenize differently
  (in-process, vLLM `/render`, estimate/byte-packing). llm-d moved tokenized
  prompt data into a dedicated, plugin-populated field on the request for exactly
  this reason; the Rust EPP should have an equivalent seam instead of hard-coding
  one tokenizer. Structured token data also enables accurate prefix-cache routing
  and future tokens-in forwarding to model servers.
- **Load shedding must be pluggable and LLM-aware.** Overload protection for LLM
  serving must look at KV-cache and queue saturation, not generic request rate.
  It belongs inside the EPP and users want to supply their own policy. Dynamo's
  frontend already sheds load (HTTP 529); the GAIE path needs the same, made
  extensible.

## Goals

* Provide exactly two extension points in the Rust EPP: PrepareData and load
  shedding / admission.
* Move today's inline tokenization behind the default PrepareData hook with no
  behavior change.
* Ship a default load-shedding hook with parity to the frontend's
  saturation / busy-threshold behavior.
* Keep the architecture simple; an unconfigured Rust EPP behaves as it does
  today.
* Allow users to add hooks via compile-time registration in a custom Rust EPP
  binary/image.

### Non Goals

* Pluggable scorers, pickers, or profile handlers. This interface depends on the Dynamo Router and the change has to come with the change in its interfaces. 
* DataProducer dependency graphs, fairness / priority-band queues, request
  eviction, or the `flowControl` feature gate from llm-d.
* Dynamic plugin loading (`.so` / WASM).
* Any dependency on the deprecated Go EPP or its config schema.

# Proposal

## Overview

Keep the existing request flow and insert two hook points around the existing
scheduler:

```
ext_proc request
  -> parse body / build request        (built in)
  -> PrepareData hook                   (pluggable)  <-- tokenization lives here
  -> Load-shedding / admission hook     (pluggable)  <-- reject under overload
  -> Scheduler: pick worker             (built in, existing KV router)
  -> attach routing headers / tokens    (built in)
  -> return decision to Envoy
```

This mirrors the useful part of the llm-d / GAIE flow (PrepareData, then
admission, then scheduling) without the surrounding machinery.

```
Parse body → Build LLMRequest → Admission (flow control)
    → PrepareData plugins  ← tokenization runs here
    → Admission plugins
    → Scheduler (Filter → Score → Pick)
    → PreRequest plugins → route to model server
```

## What becomes pluggable, and what does not

| Stage | Pluggable now? | Rationale |
|-------|----------------|-----------|
| `ext_proc` protocol / server | No | Transport; not a policy decision |
| Parse body / build request | No | Stable, shared parsing |
| PrepareData (tokenization) | Yes | Users need different tokenizers; enables prefix-cache routing and tokens-in forwarding |
| Load shedding (admission) | Yes | Users need their own overload policy |
| Scheduler (worker pick) | No (for now) | The existing KV-aware router stays the default; can be revisited later |
| Bookkeeping / headers | No | Internal correctness |

The Rust EPP is currently `ext_proc` + scheduler; this proposal adds exactly two
pluggable stages in front of the scheduler and leaves the scheduler built in. We
are not making parsing, scoring, or picking pluggable at this time.

## How the hooks work (conceptually)

* A hook is a small piece of user code selected by name in configuration.
* **PrepareData** hooks run first and can attach prepared data (starting with
  token IDs / multimodal metadata) to the request. Tokenization ships as the
  default PrepareData hook. If token data is already present (e.g. a pre-tokenized
  request), the hook is skipped. PrepareData is fail-open: if a hook errors, the
  request still proceeds and scheduling falls back to prompt-based behavior.
* **Load-shedding** hooks run next and default to allow: each hook may reject
  (mapped to HTTP 429 / 503, with 529 available for Dynamo clients); if none
  reject, the request proceeds. Multiple hooks can be chained and any rejection
  stops the request.
* The scheduler then runs unchanged, consuming the token data that PrepareData
  produced.
* We should also reconsider how to do priority scheduling to decide if we want to align with GAIE. See related [proposal](https://github.com/ai-dynamo/enhancements/pull/90/changes)

## Configuration

The Rust EPP gets a small, self-contained config (a mounted file / env var read
at startup) with just two things: which PrepareData hook to use, and an ordered
list of load-shedding hooks to run. No CRD and no dependency on the Go EPP's
config schema. Built-in hooks (a default tokenizer and a default
saturation / busy-threshold shedder) are always available; users select their own
by name. Sensible defaults mean an unconfigured Rust EPP behaves as it does
today.

## User authoring model (compile-time)

Users implement a hook in Rust, register it by name, and build a custom Rust EPP
binary that links the framework plus their hook, then publish that image and
point the GAIE `InferencePool` at it. This is compile-time only — no dynamic
loading in this first cut.

## Delivery outline

1. Introduce the two hook points in the Rust EPP and move today's inline
   tokenization behind the default PrepareData hook (no behavior change).
2. Add a default load-shedding hook (saturation / busy-threshold parity with the
   frontend) and the config plumbing to select / chain hooks.
3. Document the authoring model and ship one example custom hook alongside the
   existing GAIE docs.

# Alternate Solutions

## Alt 1 Reproduce the full llm-d / GAIE plugin pipeline in Rust

**Pros:**

* Maximum flexibility (pluggable scorers, pickers, profile handlers, fairness,
  flow control).
* Closer 1:1 mapping to the Go EPP and upstream GAIE.

**Cons:**

* Large surface area and ongoing maintenance for extension points nobody is
  asking for yet.
* Slower to deliver the two capabilities actually needed.

**Reason Rejected:**

* Over-engineered for current needs. The two hooks cover the real use cases
  (custom tokenizer, custom overload policy). Additional extension points can be
  proposed later if a concrete need appears.

## Alt 2 Keep load shedding / tokenization out of the EPP (gateway-level only)

**Pros:**

* No EPP changes.

**Cons:**

* Generic gateway RPS limits are a poor proxy for KV-cache / queue saturation.
* Tokenization for prefix-cache routing has to happen where routing decisions are
  made.

**Reason Rejected:**

* Both capabilities are inherently LLM-aware and belong in the EPP, consistent
  with llm-d and Dynamo's own frontend admission behavior.
