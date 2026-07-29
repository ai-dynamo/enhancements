# Make the Dynamo Rust EPP Extensible (PrepareData + Load Shedding + Worker State)

**Status**: Draft

**Authors**: [atchernych](https://github.com/atchernych)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [TBD — assign a code owner / maintainer]

**Required Reviewers**: [TBD — inference-gateway / EPP owners]

**Review Date**: [TBD]

**Pull Request**: [ai-dynamo/enhancements#96](https://github.com/ai-dynamo/enhancements/pull/96)

**Implementation PR / Tracking Issue**: [TBD — link to ai-dynamo/dynamo tracking issue].
Related in-flight work:
[dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865) — built-in load
shedding (DEP-1073), and
[dynamo#11868](https://github.com/ai-dynamo/dynamo/pull/11868) — cached_tokens in the
EPP response (DYNO-93).

# Summary

The Dynamo Rust EPP (`dynamo-ext-proc`) today is effectively two things: the
Envoy `ext_proc` server and a single, hard-wired router that tokenizes the
prompt and picks a worker. There are no extension points. The Go EPP — which
follows the GAIE / llm-d pipeline and supports plugins — is being deprecated, so
the Rust EPP becomes the single EPP and must absorb the extensibility that
previously only existed in Go.

This proposal adds a minimal extension model to the Rust EPP: **two plugin points and
one shared input**. The plugins are **PrepareData** (per-request data preparation before
scheduling; tokenization is the first use) and **load shedding / admission** (reject
a request under overload before scheduling). The shared input is a read-only
**worker-state view** — the KV and prefill load signal that overload decisions
actually depend on — exposed once at the extension boundary instead of being
rebuilt privately by every implementation. Everything else — parsing, the KV-aware
scheduler/picker, and bookkeeping — stays built in. We deliberately do not reproduce
the full llm-d pipeline.

The worker-state view is a critical addition. A shedding policy that cannot
see KV-cache and prefill saturation is not a shedding policy, and today no extension
point can see it: `EndpointPicker::pick` receives request metadata and an endpoint
list carrying pod identity only, so any out-of-tree policy must stand up a parallel
state feed. Exposing state as an input is deliberately *not* the same as making
scoring pluggable, which remains a non-goal.

# Motivation

The Go EPP is being deprecated, and its plugin-based extensibility goes away with
it. The Rust EPP cannot currently be extended: all logic lives in one router
implementation, so users who want custom tokenization or custom load-shedding
policy must fork it. Three concrete needs drive this proposal:

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
  extensible. The frontend's shed is a routing failure surfaced as an error, discovered after the request has already been parsed and tokenized. But customers pressing on explicit rejection versus implicit shed. The frontend returns 529 (configurable via DYN_HTTP_OVERLOAD_STATUS_CODE) with no retry hint. The EPP will return 429 with Retry-After. Both choices are intentional: 429 is what a gateway and its failover logic already understand, and the retry hint is new capability rather than parity.
  The *built-in* half of this is in flight in
  [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865), which
  reuses the frontend's `KvWorkerMonitor` and adds a `PickError::Saturated
  { retry_after_secs }` variant surfaced as HTTP 429 with `Retry-After`. What remains is the wiring for the plugin. Caveat: One frontend capability also does not carry
  over: `POST`/`GET /busy_threshold` retunes thresholds per model on a live fleet,
  whereas the EPP reads env vars once at startup. Closing that is out of scope here but needs to be done.
- **Policy state must be shared.** Every prioritization policy consumes the same family of signals: active decode blocks and KV used blocks against `kv_total_blocks`, and active prefill tokens against
  `max_num_batched_tokens`. The EPP already tracks these in `WorkerLoadState`. They
  are not reachable from any extension point, so each out-of-tree policy would have
  to re-derive them from its own subscription. We want to avoid duplicate plumbing, and a second
  source of truth that can silently disagree with the one the built-in shedder uses.
  This leads to (REQ 1).

## Goals

* Provide exactly two extension points in the Rust EPP: PrepareData and load
  shedding / admission.
* Expose one read-only worker-state view as a first-class input, consumed by the
  built-in shedder and available to any plugin, so policy state has a single source
  of truth.
* Move today's inline tokenization behind the default PrepareData plugin with no
  behavior change.
* Ship a default load-shedding plugin that reuses the frontend's saturation /
  busy-threshold *detection* — the same `KvWorkerMonitor`, thresholds, and
  overloaded-worker exclusion — while deliberately diverging on the decision surface:
  an explicit pre-tokenization gate rather than a routing failure surfaced as an
  error, and HTTP 429 with `Retry-After` rather than 529. Full parity is not the
  goal; a shared detector with a gateway-appropriate contract is.
* Allow users to add plugins via compile-time registration in a custom Rust EPP
  binary/image.

### Non Goals

* Pluggable scorers, pickers, or profile handlers. This interface depends on the
  Dynamo Router and the change has to come with the change in its interfaces.
  Exposing worker state as a read-only *input* (above) does not breach this: a
  policy may read the same signals the scheduler reads, and may reject or defer a
  request, but it does not participate in scoring or selection.
* Steering the router's internal scoring from an EPP extension — for example
  zeroing a worker's prefix-overlap credit, retuning the KV indexer's TTL decay, or
  forcing an index resync for one worker. Integrators have asked for these, and they
  are legitimate needs, but they are operations on the router's own state, not on the
  EPP's request path,
  and giving the EPP a back door into them would create exactly the coupling the
  first non-goal avoids. They belong to the router and its policy-class admission
  API.
* DataProducer dependency graphs, fairness / priority-band queues, request
  eviction, or the `flowControl` feature gate from llm-d.
* Dynamic plugin loading (`.so` / WASM).
* Any dependency on the deprecated Go EPP or its config schema.

## Requirements

These come from a production integrator running the Rust EPP behind agentgateway,
who intends to supply their own load-shedding and prioritization policy. Their
framing is worth recording because it narrows the design: for them load shedding is
primarily a *routing signal* — a fast explicit rejection that failover can act on,
replacing implicit shed-by-timeout that wastes prefill compute — and prioritization
is about retry-versus-first-attempt and request size, not tenant tiers.

### REQ 1 Worker state as a first-class input

Worker state feeds (as the integrator framed it: KV blocks, prefill tokens, queue
depth) must be an input to the extension boundary rather than something each policy
derives itself. Note that of these, KV blocks and prefill tokens are tracked today;
queue depth is not, and this DEP proposes exposing what exists rather than adding a
new signal. **Not yet met.**
`pick()` receives request metadata and an endpoint list of pod identity only; in
practice the server passes an empty endpoint slice because pickers resolve endpoints
internally. All load signal is private to the concrete router. This is the gap this
revision closes.

### REQ 2 Shed semantics distinguishable from failure

Rejection under overload must be distinguishable from an error and routable by the
gateway, with a retry hint. **Met by
[dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865)** via `PickError::Saturated
{ retry_after_secs }` → HTTP 429 with `Retry-After`. Because it is an ordinary error
variant, an out-of-tree policy can return it too.


### REQ 3 Class-aware shed thresholds

Today the threshold is one number for everybody. For example, shed when a worker is over 85% KV-block occupancy. The #11865 rejects only when every eligible worker is over it. So the pool has exactly two states: open to all, or closed to all. The moment saturation is crossed, a 200-token chat request and a 100k-token request get treated identically: both get a 429. The customers want flexibility here as not all requests are equal under load. The EPP already knows prompt size because it tokenized, and a retry marker is one header away so we can extend this.
The router side already has most of the machinery: the KV router already assigns a request to a class either by explicit name or by an uncached ISL bucket — that is, bucketed on the tokens that actually need prefilling, which is a better cost proxy than raw prompt length. So request-size-aware classes exist; what's missing is expressing shed thresholds per class rather than process-wide.

The classifier itself is reusable — `PolicyProfile::resolve_class_index` is public, not
scheduler-internal. The complication is its argument: it takes *uncached* tokens, and
the EPP cannot know those at shed time. Overlap comes from the KV index query inside
`pick()`, whereas the shed gate runs before the body is even decoded. So this
requirement resolves into one of three options, which differ in cost and in whether the
router changes at all:

1. **Classify on raw token count in the EPP.** No router change. The EPP keeps its own
   per-class threshold table and buckets on prompt length. Cheapest, but the EPP's
   class for a request then disagrees with the router's, so one request can be a
   "small" shed class and a "large" queue class at the same time — the two-sources-of-
   truth problem this DEP objects to elsewhere, in a new place.
2. **Move the shed gate after tokenization and the overlap query.** No router change,
   and class identity stays consistent, but it gives up the cheap pre-parse refusal
   that distinguishes the EPP's shed from the frontend's error-driven one.
3. **Push the decision into the router**, where uncached tokens and class assignment
   already sit together. This is the only option requiring a router contract change:
   `AdmissionDecision` is `Bypass | Ready | Defer` today, with no rejection variant, so
   the policy-class admission API can hold a request indefinitely but cannot shed one.

A related question is where per-class thresholds are configured. They arguably belong
beside the class definitions in the router's policy YAML — which already carries a
per-class `admission:` envelope — rather than in EPP environment variables, since
splitting class definitions from class thresholds invites drift.

This DEP does not pick one yet; the choice should be made before implementation starts.

# Proposal

## Overview

Keep the existing request flow and insert two plugin points around the existing
scheduler, both reading one shared worker-state view:

```
                                        worker-state view (read-only, shared)
                                             |            |
ext_proc request                             v            v
  -> parse body / build request         (built in)
  -> PrepareData plugin                  (pluggable)  <-- tokenization lives here
  -> Load-shedding / admission plugin    (pluggable)  <-- reject under overload
  -> Scheduler: pick worker              (built in, existing KV router)
  -> attach routing headers / tokens     (built in)
  -> return decision to Envoy
  -> response callbacks                  (built in)  <-- prefill complete, usage
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
| Worker-state view | No — read-only input | Not a stage. One shared source of truth for the signals policies need (REQ 1) |
| Scheduler (worker pick) | No (for now) | The existing KV-aware router stays the default; can be revisited later |
| Bookkeeping / headers | No | Internal correctness |

The Rust EPP is currently `ext_proc` + scheduler; this proposal adds exactly two
pluggable stages in front of the scheduler, one read-only input feeding them, and
leaves the scheduler built in. We are not making parsing, scoring, or picking
pluggable at this time.

## How the plugins work (conceptually)

* A plugin is a small piece of user code selected by name in configuration.
* **PrepareData** plugins run first and can attach prepared data (starting with
  token IDs / multimodal metadata) to the request. Tokenization ships as the
  default PrepareData plugin. If token data is already present (e.g. a pre-tokenized
  request), the plugin is skipped. PrepareData is fail-open: if a plugin errors, the
  request still proceeds and scheduling falls back to prompt-based behavior.
* **Load-shedding** plugins run next and default to allow: each plugin may reject
  (mapped to HTTP 429 / 503, with 529 available for Dynamo clients); if none
  reject, the request proceeds. Multiple plugins can be chained and any rejection
  stops the request.
* The scheduler then runs unchanged, consuming the token data that PrepareData
  produced.
* We should also reconsider how to do priority scheduling to decide if we want to align with GAIE. See related [proposal](https://github.com/ai-dynamo/enhancements/pull/90)

## Worker state as a first-class input

A load-shedding plugin is only as good as what it can see. Today the EPP computes the
overload signal in `KvWorkerMonitor` and keeps it private to the concrete router, so
a plugin would either be handed a pre-baked boolean or have to build its own feed. The
first is not a policy seam; the second duplicates plumbing and creates a second
source of truth that can disagree with the built-in shedder.

Instead, the same state the built-in shedder consumes is exposed once, read-only, at
the extension boundary:

* **Per-worker load**, in the terms the thresholds are already expressed in: active
  decode blocks and KV used blocks against `kv_total_blocks`, and active prefill
  tokens against `max_num_batched_tokens`, per dp_rank. Queue depth is *not* part of
  this set today; adding it would be a deliberate extension of the view, not an
  assumed member of it.
* **The derived overloaded-worker set**, so a policy can ask the cheap question
  ("is anything free?") without recomputing the expensive one — and, more
  importantly, get the same latched answer the built-in shedder acts on rather than
  an unlatched approximation of it.
* **A distinction between structurally eligible and currently available workers**,
  matching the split the router's admission contract already makes, so a policy can
  tell "no worker can ever serve this" from "every worker is busy right now."

Three properties keep this from turning into a scoring API. The view is **read-only**:
a policy observes, it does not mutate router state. It is a **snapshot** with a
consistent view across a single decision, not a live handle that invites a policy to
poll in a loop. And it is **typed and narrow** — named accessors for signals we
already maintain, not a metadata bag — so adding a fact is a deliberate act and the
boundary stays reviewable.

This is what makes the decorator authoring model honest. Wrapping the stock router
in an outer policy already works mechanically — the trait is a normal Rust trait and
the server is generic over it — but a wrapper that cannot see load can only reject
blindly or reorder what comes back. With the state view it can make the same class
of decision the built-in shedder makes, which is the whole point of the seam.

## Relationship to the router's policy-class admission API

Dynamo already has a second extension boundary that is easy to miss and must not be
duplicated: `PolicyClassAdmissionPolicy`, in
`lib/kv-router/src/scheduling/queue_admission/`. It is contract-complete — per-class
lifecycle events, an opaque per-class configuration envelope each policy
deserializes itself, and crucially a `Defer` / `MakeReady` pair, so it can *hold* a
request rather than only rejecting it. It has no production implementations today
and is not wired into the EPP.

The two boundaries answer different questions and should stay that way:

| | EPP plugins (this DEP) | Router policy-class admission |
|---|---|---|
| Scope | One request at the gateway edge | A class of requests inside the scheduler |
| Vocabulary | Admit or reject, now | Admit, defer, release, place |
| Signal | Worker state at pick time | Queue state and class capacity over time |
| Answer to overload | Shed with 429 + `Retry-After` | Queue and release when capacity returns |

Shedding at the edge and queueing in the scheduler are complements: the edge is where
you cheaply refuse work you should never start, and the scheduler is where you hold
work that is worth waiting for. Building a third mechanism to span them would be a
mistake.

This division also gives REQ 3 its natural home. Policy classes are already assigned
either by explicit class name or by a `(policy family, uncached-ISL bucket)` pair —
that is, **request-size-aware classes already exist** in the router, which is most of
what class-aware shedding asks for. The remaining work is connective rather than new
machinery: the EPP knows prompt size and can see a retry marker, so it can label a
request with a policy class, and shed thresholds can then be expressed per class
instead of process-wide. Whether that label rides the existing `policy_class` hint
(the frontend takes a `policy-class` metadata key; the selection service takes an
`x-dynamo-meta-policy-class` header) or a dedicated EPP-side mapping is an
implementation question this DEP defers.

## Configuration

The Rust EPP gets a small, self-contained config (a mounted file / env var read
at startup) with just two things: which PrepareData plugin to use, and an ordered
list of load-shedding plugins to run. No CRD and no dependency on the Go EPP's
config schema. Built-in plugins (a default tokenizer and a default
saturation / busy-threshold shedder) are always available; users select their own
by name. Sensible defaults mean an unconfigured Rust EPP behaves as it does
today.

## User authoring model (compile-time)

Users implement a plugin in Rust, register it by name, and build a custom Rust EPP
binary that links the framework plus their plugin, then publish that image and
point the GAIE `InferencePool` at it. This is compile-time only — no dynamic
loading in this first cut.

## Delivery outline

1. Land the built-in shedder
   ([dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865)): `KvWorkerMonitor` reuse,
   `PickError::Saturated { retry_after_secs }`, HTTP 429 with `Retry-After`.
   Satisfies REQ 2 and gives the later plugin a reference implementation.
2. Expose the worker-state view as a read-only input and refactor the built-in
   shedder to consume it, proving the boundary carries a real policy before any
   third party depends on it. Satisfies REQ 1.
3. Introduce the two plugin points and move today's inline tokenization behind the
   default PrepareData plugin (no behavior change).
4. Add the config plumbing to select / chain plugins, with the built-in shedder as
   the default load-shedding plugin.
5. Express shed thresholds per policy class rather than process-wide, reusing the
   router's existing class assignment. Satisfies REQ 3.
6. Document the authoring model and ship one example custom plugin alongside the
   existing GAIE docs.

Step 2 before step 3 is deliberate. Shipping plugins first would mean publishing a
seam whose only real policy input is still private, which is how a boundary ends up
frozen around the wrong shape.

# Related Proposals

* [DEP: Multi-DC KV-Aware Request Routing](https://github.com/ai-dynamo/dynamo/issues/11225) —
  its Relay publishes the same family of worker/serving signals upward for the
  fleet-level loop. One worker-state surface should feed both the Relay and the
  EPP's policies rather than two parallel paths.
* [Priority scheduling proposal](https://github.com/ai-dynamo/enhancements/pull/90) —
  decides whether EPP-side prioritization aligns with GAIE; REQ 3 depends on the
  outcome.
* [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865) — "Enable load
  shedding in the EPP" (DEP-1073). Adds the built-in shedder this DEP's default plugin
  is built from: `KvWorkerMonitor` reuse, `PickError::Saturated { retry_after_secs }`,
  and HTTP 429 with `Retry-After`. Satisfies REQ 2, and its private
  `WorkerLoadState` is exactly what REQ 1 proposes to expose.
* [dynamo#11868](https://github.com/ai-dynamo/dynamo/pull/11868) — "Enable
  cached_tokens in EPP response" (DYNO-93). Parses `usage` from the response body and
  adds `on_request_complete_with_usage`. Not a plugin concern — it extends a built-in
  `EndpointPicker` callback, and neither plugin point in this proposal runs on the
  response path — but it is adjacent in-flight work in the same crate, and it supplies
  the observed cache hit that integrators want for comparing predicted overlap against
  actual, which this DEP treats as router work rather than an extension point.

The two are complements, not overlaps: #11865 reads worker capacity on the request
path to decide whether to admit, and #11868 reads request outcome on the response
path after the decision is already made. They do touch the same files, so whichever
lands second will need a small merge resolution in `picker.rs` and `epp.rs`.

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

* Over-engineered for current needs. The two plugins plus the shared state view cover
  the real use cases (custom tokenizer, custom overload policy). Additional
  extension points can be proposed later if a concrete need appears.

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

## Alt 3 Ship the plugins without exposing worker state

Leave the state feed private and let each out-of-tree policy build its own — the
`EndpointPicker` trait is already wrappable, so an integrator can decorate the stock
router and subscribe to worker events independently.

**Pros:**

* Smallest possible boundary; no new types to maintain or version.
* Nothing to get wrong in the state view's shape before we have several policies to
  generalize from.

**Cons:**

* Two sources of truth for overload. A policy's private feed and the built-in
  shedder's `KvWorkerMonitor` can disagree, and the resulting behavior — shed by one,
  admitted by the other — is very hard to debug from outside.
* Duplicate plumbing in every deployment that wants a custom policy, which is the
  specific cost REQ 1 was raised to avoid.
* The plugin seam would be mostly decorative: a shedding plugin that cannot see
  saturation can only apply request-shaped heuristics, so the interesting policies
  would still live in forks.

**Reason Rejected:**

* This is effectively the status quo with extra ceremony. If the seam ships without
  the input the policy needs, integrators keep forking and we still own the
  compatibility burden of a published trait.
