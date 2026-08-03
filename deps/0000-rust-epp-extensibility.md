# Make the Dynamo Rust EPP Extensible through Plugins

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
Envoy `ext_proc` server and a hard-wired router that tokenizes the
prompt and picks a worker. There are no extension points. The Go EPP — which
follows the GAIE / llm-d pipeline and supports plugins — is being deprecated, so
the Rust EPP becomes the single EPP and must absorb the extensibility that
previously only existed in Go.

This proposal adds a minimal extension model to the Rust EPP: **two plugin types and
one shared input**. The plugins are **DataProducer** (per-request data preparation before
scheduling; tokenization is the first use) and **Admitter (load shedding)** (reject
a request under overload before scheduling). 

All plugins will need to have read-only worker-state view (the KV and prefill load signal)
The worker-state view is a critical addition. A shedding policy that cannot
see KV-cache and prefill saturation is not a shedding policy, and today no extension
point can see it: `EndpointPicker::pick` receives request metadata and an endpoint
list carrying pod identity only, so any out-of-tree policy must stand up a parallel
state feed. Exposing state as an input is deliberately not the same as making
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
  Users want to supply their own policy. The shedding must happen at the EPP level. 
- **Policy state** must be shared, not rebuilt. Any policy reasoning about saturation needs the signals the built-in thresholds are already expressed in: active_decode_blocks and kv_used_blocks against kv_total_blocks, and active_prefill_tokens against max_num_batched_tokens. These span two feeds — numerators from the ActiveLoad stream, denominators from the runtime-config watch — and WorkerLoadState already joins them. It lives in dynamo-llm, which the EPP links today, so the EPP should construct a KvWorkerMonitor and reuse it rather than grow a parallel implementation. Tracking is only half the gap: the state is a private field and EndpointPicker::pick has no parameter. We need to expose it to enable plugins:

```bash
async fn pick(
    &self,
    req: &RequestInfo,
    endpoints: &[Endpoint], 
    load: &WorkerLoadView, // here
) -> Result<PickResult, PickError>;

pub struct WorkerLoadView{}
  load: FxHashMap<WorkerWithDpRank, WorkerLoad>
  overloaded: FxHashSet<WorkerId>,
  /// Thresholds in force when the snapshot was taken.
  thresholds: LoadThresholdConfig,
  observed_at: Instant,
}

pub struct WorkerLoadState {
    pub active_decode_blocks: HashMap<u32, u64>,
    pub kv_used_blocks: HashMap<u32, u64>,
    pub kv_total_blocks: HashMap<u32, u64>,
    pub active_prefill_tokens: HashMap<u32, u64>,
    /// max_num_batched_tokens from runtime config (same for all dp_ranks)
    pub max_num_batched_tokens: HashMap<u32, u64>,
    decode_overload_latches: HashMap<u32, DecodeOverloadLatchState>,
}
```
  

## Goals


* Provide extension points in the Rust EPP: DataProducer and Admitter.
* Expose one read-only worker-state view as a first-class input available to any plugin, so policy state has a single source of truth.
* Move today's inline tokenization behind the default PrepareData plugin with no
  behavior change.
* Ship a default load-shedding plugin that reuses the frontend's saturation /
  busy-threshold detection (the same `KvWorkerMonitor`, thresholds, and
  overloaded-worker exclusion) while deliberately diverging on the decision surface:
  an explicit admission gate rather than a routing failure surfaced as an
  error, and HTTP 429 with `Retry-After` rather than 529. Full parity is not the
  goal; a shared detector with a gateway-appropriate contract is.
* Allow users to add plugins via compile-time registration in a custom Rust EPP
  binary/image.
* We deliberately do not reproduce the full llm-d pipeline.


### Non Goals

* Pluggable scorers, pickers, or profile handlers. This interface depends on the
  Dynamo Router and the change has to come with the change in its interfaces.
  Exposing worker state as a read-only *input* (above) does not breach this: a
  policy may read the same signals the scheduler reads, and may reject or defer a
  request, but it does not participate in scoring or selection. The related proposal is 
  reflected in these [slides](https://docs.google.com/presentation/d/1_k-ytG9QxgSUAOnLG7CZG90QM6Zjx3PvpzaI24N4_So/edit?slide=id.g3f6000d6429_0_0#slide=id.g3f6000d6429_0_0)
* Steering the router's internal scoring from an EPP extension — for example
  zeroing a worker's prefix-overlap credit, retuning the KV indexer's TTL decay, or
  forcing an index resync for one worker. Clients have asked for these, and they
  are legitimate needs, but they are operations on the router's own state, not on the
  EPP's request path, and giving the EPP a back door into them would create exactly the coupling the
  first non-goal avoids. They belong to the router and its policy-class admission
  API.
* Dynamic plugin loading (`.so` / WASM).
* Any dependency on the deprecated Go EPP or its config schema.

## Requirements

Clients want to supply their own load-shedding and prioritization policy. For them load shedding is
primarily a *routing signal* — a fast explicit rejection that failover can act on,
replacing implicit shed-by-timeout that wastes prefill compute . Also for them prioritization
is about retry-versus-first-attempt and request size, not tenant tiers.

### REQ 1 Worker state as a first-class input

Worker state feeds (KV blocks, prefill tokens, queue depth) must be an input to plugins rather than something each policy derives itself. Note that of these, KV blocks and prefill tokens are tracked today; queue depth is not, and this DEP proposes exposing it. 


### REQ 2 New Shed semantics

Today the FontEnd implements saturation detection and responds with http 529 error. The shedding decision outcome should not be an error or requeue inside the router, it should reside in the EPP as a custom-either retry or reject and sent to the gateway. We will use the FrontEnd logic with a change in this [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865)** via `PickError::Saturated
{ retry_after_secs }` → HTTP 429 with `Retry-After`. 



### REQ 3 Class-aware shed thresholds

Today the threshold is one number for everybody. For example, shed when a worker is over 85% KV-block occupancy. The #11865 rejects only when every eligible worker is over it. So the pool has exactly two states: open to all, or closed to all. The moment saturation is crossed, a 200-token chat request and a 100k-token request get treated identically: both get a 429. The customers want flexibility here as not all requests are equal under load. 

The customer wants to admit small first-attempts while shedding the oversized retries. For example an oversized retry might shed at 70% while a small first attempt keeps being admitted until 95%. So the class is (request size, retry-vs-first-attempt). Dynamo has no concept of the retry marker for a class. In this proposal the EPP would set policy_class = "retry" or "first_attempt".
We will take the uncached_tokens argument as the proxy for the request size. This is better, since a 100k-token request with a 99k prefix hit is cheap and shouldn't be shed as if it were expensive. The classifier multiplies them, so the pair (size, retry) is the request_class.

Today the shed decision is funtion_of(worker_state). The client wants function_of(worker_state, request_class).


The router side already has some of the machinery. The KV router already assigns a request to a class either by explicit name or by an uncached ISL bucket, bucketing on the tokens that actually need prefilling, which is a better cost proxy than raw prompt length.
But it drives queueing, and the shed path never calls it. They're also in different crates: classification in dynamo-kv-router, shed threshold in dynamo-llm's monitor. We need to use the classifier from the kv router. 
The classifier itself is implemented in `PolicyProfile::resolve_class_index`. The complication is its argument: it takes *uncached* tokens, which cannot be derived from the request alone. Overlap is a property of the prompt's block hashes walked against the KV index, not a property of worker state, so no amount of background monitoring produces it — the index has to be queried for this specific prompt. Today that query happens inside the routing call, whereas Gate A runs before the body is even decoded. 

```bash
pub fn resolve_class_index(&self, requested: Option<&str>, uncached_tokens: usize) -> usize {
    match &self.classifier {
        PolicyClassifier::SyntheticSingle { class_index } => *class_index,
        PolicyClassifier::FamilyBucket(classifier) => {
            // TODO: Add bounded observability for unknown requested policy values.
            classifier.class_index(requested, uncached_tokens)
        }
    }
}
```
We can run this alg before we select the worker and feed the tokens from the prompt into uncached_tokens BUT this would result in making the shedding worse. Raw ISL would systematically over-estimate cost on that traffic, so a mostly-cached large request would land in the "oversized" bucket and get shed — the precise issue we want to prevent.

The way out is that the router already exposes a routing query that books nothing.
`KvRouter::find_best_match_details_without_admission` runs the normal route in
`ScheduleMode::QueryOnly` and returns the facts a cost model needs — `cached_tokens` among
them — without reserving anything:

```rust
pub enum FindBestMatchAdvisoryOutcome {
    Routed {
        worker: WorkerWithDpRank,
        overlap_blocks: u32,
        effective_overlap_blocks: f64,
        cached_tokens: usize,
        potential_decode_blocks: u64,
        selected_worker_load: scheduling::AdvisoryWorkerLoad,
        routing_hashes: Option<RoutingDecisionHashes>,
    },
    QueueRejected { rejection: scheduling::QueueRejection },
}
```

Because `cached_tokens` comes back directly, deriving the class index needs no block-size
arithmetic:

```rust
let uncached_tokens = tokens.len().saturating_sub(cached_tokens);
let class_index = profile.resolve_class_index(policy_class.as_deref(), uncached_tokens);
```

Conditional disaggregation already uses this path in exactly this shape: it calls the
advisory query, reads `cached_tokens` and `selected_worker_load`, then calls
`selected_worker_load.prefill_load_exceeds(threshold)` to decide whether to bypass prefill
(`lib/llm/src/kv_router/prefill_router/conditional_bypass.rs`). A class-aware shed is the
same decision shape, so this is an existing mechanism rather than a new one.

**Performing the advisory query is the plugin's responsibility, not the pipeline's.** The
Admitter therefore stays similar to where llm-d puts it — after data preparation, before the
scheduler — and a policy that needs overlap-derived facts issues the advisory query
itself. That makes the cost opt-in. A deployment running no Admitter, or one whose policy
keys only on worker state and request headers, routes exactly once and pays nothing extra.
Only a policy that actually wants uncached ISL pays for a second query, and it pays
knowingly. Moving the gate after the scheduler instead would impose the reordering on
every deployment, whether or not it used a class-aware policy.

Alt 4 proposes the opposite arrangement — placing the class-aware shed inside the router,
where classification already sees accurate uncached ISL — which would remove the need for
the advisory query and for a gateway-side Admitter altogether. It is open rather than
rejected, and the choice between it and the arrangement below should be made before
implementation starts.

The following flow is proposed. `Router::pick()` is the `EndpointPicker` trait method in
`epp.rs`; everything below it runs inside that call.

```text
Router::pick()                             epp.rs
 ├─ GATE A request-blind shedding          epp.rs      (1)
 ├─ tokenize                               epp.rs
 ├─ GATE B request-aware shedding          epp.rs      (2)  ← new, pluggable
 │    └─ advisory query — only if the policy needs uncached ISL
 │         └─ KvRouter::find_best_match_details_without_admission()
 │                                         kv-router    existing, books nothing
 ├─ Router::route_prefill()                epp.rs       thin wrapper
 │    └─ PrefillRouter::…                  kv-router    unchanged
 ├─ Router::route_decode()                 epp.rs       thin wrapper
 │    └─ KvRouter::find_best_match()       kv-router    unchanged
 ├─ resolve the worker endpoint            epp.rs
 └─ Router::add_request()                  epp.rs       bookkeeping
      └─ KvRouter::add_request()           kv-router    unchanged
```

1. **GATE A** — basic, request-blind shedding, already implemented in
   [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865). Not a plugin. It runs
   before tokenization, so it costs nothing to refuse a fully saturated pool. Every worker is continuously marked overloaded-or-not based on how full its KV cache and prefill queue are versus a fixed threshold, and Gate A rejects the incoming request with 429 only when every worker eligible to serve it is currently marked overloaded. Dynamo's frontend already sheds load (HTTP 529); the GAIE path needs the same, made extensible. The frontend's shed is a routing failure surfaced as an error, discovered after the request has already been parsed and tokenized. But customers pressing on explicit rejection versus implicit shed. The frontend returns 529 (configurable via DYN_HTTP_OVERLOAD_STATUS_CODE) with no retry hint. The EPP will return 429 with Retry-After. Both choices are intentional: 429 is what a gateway and its failover logic already understand, and the retry hint is new capability rather than parity. The beginning of its implementation is in [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865), which reuses the frontend's `KvWorkerMonitor` and adds a `PickError::Saturated { retry_after_secs }` variant surfaced as HTTP 429 with `Retry-After`. What remains is the wiring for the plugin. Caveat: One frontend capability also does not carry over: `POST`/`GET /busy_threshold` returns thresholds per model on a live fleet, whereas the EPP reads env vars once at startup. Closing that is out of scope here.
2. **GATE B** — the new class-aware shedding, exposed as a plugin, occupying the Admitter
   position: after tokenization, before the scheduler. It rejects before anything is
   booked, so a rejection has nothing to roll back — the advisory query reserves no
   capacity even when a policy chooses to issue one. A policy that needs uncached
   ISL obtains it from the advisory query; a policy keying only on worker state and
   headers skips that call entirely.

Everything marked `unchanged` is `dynamo-kv-router` code this proposal does not touch:
scoring, selection, and the router's own bookkeeping all stay as they are.


The per-class thresholds should be configured in the router's policy YAML — which already carries a
per-class `admission:` envelope rather than in EPP environment variables, since
splitting class definitions from class thresholds invites drift. `RouterPolicyConfig::from_yaml`, `resolve_profile`, `PolicyProfile`, and both class-index methods are all public, so the EPP can use them. But `KvRouterConfig::loaded_policy_config()` is private, so the EPP either gets that made public or loads the YAML itself.

This DEP does not pick one yet; the choice should be made before implementation starts.

# Proposal

## Overview

Keep the existing request flow and insert two plugin points in front of the existing
scheduler, both reading one shared worker-state view:

```text
ext_proc request
  → parse body / build request            built in
  → GATE A: global saturation shed        built in    request-blind, dynamo#11865
  → DataProducer plugin                   pluggable   tokenization lives here    [view]
  → GATE B: Admitter (load shedding)      pluggable   reject under overload      [view]
  → Scheduler: pick worker                built in    existing KV router, unchanged
  → book the request + attach headers     built in
  → return decision to Envoy
  → response callbacks                    built in    prefill complete, usage

[view] = reads the shared read-only worker-state view
```

This matches the useful part of the llm-d / GAIE flow — a data-preparation stage, then a
stage that may reject, then scheduling — without the surrounding machinery. llm-d's own
ordering is:

```text
Parse body → Build LLMRequest → Flow control   ← builtin, not pluggable
    → DataProducer plugins   ← tokenization runs here
    → Admitter plugins       ← can reject
    → Scheduler (Filter → Score → Pick)
    → PreRequest plugins → route to model server
```

### Alignment with llm-d's two admission layers

llm-d admits in two distinct places:

* **Flow control** runs first and is a *builtin*. Capacity rejection
  (`maxBytes` / `maxRequests`), TTL eviction, and priority-band traversal are
  infrastructure that protects the gateway process itself, so they are not
  swappable. Priority-band selection is, in their words, "hardcoded and not
  pluggable," and the queue contract states that "capacity management occurs
  outside the queue implementation." Only the *signal* (`SaturationDetector`), the
  *curve* (`UsageLimitPolicy`), and the *order* (`FairnessPolicy`,
  `OrderingPolicy`) are plugins.
* **`Admitter` plugins** run after `DataProducer` and before scheduling, and these
  *can* reject outright. The `latency-slo-admitter`plugin is an example here.

The load-shedding plugin proposed here is that `Admitter`: same pipeline position, same
ordering relative to data preparation, same ability to reject. That correspondence is the
reason the seam sits where it does.

One difference in the *policies* is worth noting, because it explains why this DEP needs a
mechanism upstream does not. llm-d's `latency-slo-admitter` keys on facts that arrive with
the request — SLO class and priority — plus a fleet-level saturation signal, so every
input is available at ingress. Dynamo's class is derived partly from request *cost*, and
the honest cost measure is uncached ISL, which requires querying the KV index for that
specific prompt. Keeping the Admitter at the same pipeline position while still supporting
a cost-derived class is what the advisory query above buys: the plugin reaches for overlap
when its policy needs it, rather than the pipeline reordering itself for every deployment.


The upstream split between the saturation *signal* and the admit/reject *decision*
is a refinement this DEP does not currently make — it proposes one plugin that
does both. This is TBD.

## What becomes pluggable, and what does not

| Stage | Pluggable now? | Rationale |
|-------|----------------|-----------|
| `ext_proc` protocol / server | No | Transport; not a policy decision |
| Parse body / build request | No | Stable, shared parsing |
| Global saturation shed (Gate A) | No | Process self-protection; fixed threshold, already built in |
| DataProducer (tokenization) | Yes | Users need different tokenizers; enables prefix-cache routing and tokens-in forwarding |
| Admitter (load shedding, Gate B) | Yes | Users need their own overload policy |
| Worker-state view | No — read-only input | Not a stage. One shared source of truth for the signals policies need (REQ 1) |
| Scheduler (worker pick) | No (for now) | The existing KV-aware router stays the default; can be revisited later |
| Bookkeeping / headers | No | Internal correctness |

The Rust EPP is currently `ext_proc` + scheduler; this proposal adds exactly two
pluggable stages in front of the scheduler, one read-only input feeding them, and leaves
the scheduler built in. We are not making parsing, scoring, or picking pluggable at this
time.

## How the plugins work (conceptually)

* A plugin is a small piece of user code selected by name in configuration.
* **DataProducer** plugins run first and can attach prepared data (starting with
  token IDs / multimodal metadata) to the request. Tokenization ships as the
  default PrepareData plugin. If token data is already present (e.g. a pre-tokenized
  request), the plugin is skipped. PrepareData is fail-open: if a plugin errors, the
  request still proceeds and scheduling falls back to prompt-based behavior.
* **Admitter (Load-shedding)** plugins run next and default to allow: each plugin may reject
  (mapped to HTTP 429 / 503, with 529 available for Dynamo clients); if none
  reject, the request proceeds. Multiple plugins can be chained and any rejection
  stops the request. A plugin whose policy needs request cost issues the advisory routing
  query itself; that cost falls on the policies that want it rather than on every request.
* The scheduler then runs unchanged, consuming the token data that DataProducer
  produced.
* We should also reconsider how to do priority scheduling to decide if we want to align with GAIE. See related [proposal](https://github.com/ai-dynamo/enhancements/pull/90)

## Worker state as a first-class input

A load-shedding plugin is only as good as what it can see. Today the EPP computes the
overload signal in `KvWorkerMonitor` and keeps it private to the concrete router. 
`KvWorkerMonitor` already holds `worker_load_states: Arc<DashMap<u64, WorkerLoadState>>`, 
kept current by its own
background task against the Dynamo runtime and already recomputing the derived
overloaded set on every update. There is nothing to fetch: the gap is that this `Arc`
is a private field and the trait boundary has no parameter to pass it through. The
cheap fix is for the monitor to publish an immutable snapshot into a `watch` channel —
a pattern it already uses internally — so the update path pays the cost and each
decision takes one refcount bump rather than a map traversal.

For the Admitter the following data is needed. 

**1. Raw per-worker load.** The same numbers the built-in thresholds compare against,
per worker and per dp_rank: `active_decode_blocks` and `kv_used_blocks` against
`kv_total_blocks`, and `active_prefill_tokens` against `max_num_batched_tokens`. A
plugin that wants a different rule (i.e. shed at 70% instead of 85%, or weigh prefill
pressure differently) reads these  numbers so that it can decide for itself. 

**2. We need to add the Queue depth to the worker state view.
Workers already publish per-worker, per-dp_rank queued request counts and token sums in the forward-pass metrics stream, but that stream does not feed the router. A solution is needed here.

**3. The precomputed overloaded-worker set.** Which workers are currently too busy. A
plugin uses this to decide whether to admit the request. If some workers are free, it
admits and lets normal routing pick one. If they are all busy, the plugin decides what
to do with *this* request — for example reject low-priority traffic with 429 and
`Retry-After` but let high-priority traffic through. That per-request choice is the
point, since the built-in shedder is all-or-nothing.

**4. Structurally eligible versus currently available workers.** Two sets, matching
the split `WorkerEligibilitySnapshot` already makes in the router's admission
contract:

* No structurally eligible worker means nothing in the fleet can ever serve this
  request. Retrying will not help.
* Structurally eligible but none available means every capable worker is busy right
  now. Retrying will help.

This distinction decides what the gateway gets told, which makes it load bearing for
REQ 2: only the second case should become 429 with `Retry-After`. The first is a
permanent failure, and attaching a retry hint to it actively misleads the gateway's
failover logic. A policy that cannot see both sets cannot tell the two apart.


## Relationship to the router's policy-class admission API

Dynamo already has a second extension boundary that is easy to miss and must not be
duplicated: `PolicyClassAdmissionPolicy`, in
`lib/kv-router/src/scheduling/queue_admission/`. The router can *hold* a
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


## Configuration

The Rust EPP gets a small, self-contained config (a mounted file / env var read
at startup) with just two things: which PrepareData plugin to use, and an ordered
list of load-shedding plugins to run. No CRD and no dependency. 
Built-in plugins (a default tokenizer and a default
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
* The gap is also narrower than it looks. `DataProducer` and `Admitter` are the only
  two plugin points llm-d places between request parsing and scheduling, so this
  proposal already matches upstream over that span. What Alt 1 adds beyond it is the
  flow-control machinery and pluggable scoring — both already Non Goals, and the
  former not pluggable upstream either.

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

## Alt 4 Implement class-aware shedding in the KV router; make the EPP a thin wrapper

Put the shed decision where classification already happens — inside `dynamo-kv-router`'s
scheduling path — instead of adding an Admitter plugin at the gateway. The EPP then
contributes only the parts that are genuinely HTTP-shaped: an overload signal, a mapping
from request headers to `policy_class`, and the translation of a rejection into 429 with
`Retry-After`. 

Most of the machinery for this already exists:

| Piece | State |
|-------|-------|
| Class = family × uncached-ISL bucket | Exists (`FamilyBucketClassifier`) |
| Per-class configuration envelope | Exists (`PolicyClassConfig`) |
| Per-class rejection with a typed reason | Exists (`QueueRejection` → `KvSchedulerError::QueueRejected`) |
| Reject when all eligible workers are overloaded | Exists (`KvSchedulerError::AllEligibleWorkersOverloaded`) |
| Host-supplied overload signal | Exists (`OverloadedWorkerProvider`) |
| Overload signal keyed *by class* | Missing |
| EPP supplying any overload signal | Missing |

The router will get a shedding policy plugin:

```bash
if let Some(policy) = &self.shed_policy {
    match policy.evaluate(&ShedContext { class, uncached_tokens, eligibility, load }) {
        ShedDecision::Reject { retry_after } => {
            request.respond(Err(KvSchedulerError::Shed { policy_class: class.name.clone(), retry_after }));
            return (false, false);
        }
        ShedDecision::Continue => {}
    }
}
```

The router therefore already rejects, and already rejects per class — the gap is narrower
than "it queues instead of rejecting." `OverloadedWorkerProvider` is
`Arc<dyn Fn() -> Option<HashSet<WorkerId>>>`; a closure taking no arguments can answer
"who is overloaded" but never "who is overloaded *for this class*." Passing the class and
its uncached token count is the core change.

**Pros:**

* Classification runs where `uncached_tokens` is already correct, after overlap is known.
  This removes the need for the advisory query, the class-index derivation in the EPP, and
  any question about where the gate sits relative to the scheduler — REQ 3's whole
  complication disappears.
* The frontend inherits the same capability from the same code, so 529-vs-429 becomes a
  presentation choice for EPP vs the FrontEnd rather than two implementations of one policy.
* Per-class thresholds live beside the per-class definitions in the router policy YAML,
  which is where this DEP already argues they belong.
* Queue-versus-reject behavior switch needed. We need two levels of configuration.
 1. Per class, in the policy YAML. A field on PolicyClassConfig next to the thresholds it already holds — on_saturation: Queue | Reject, defaulting to Queue so no Frontend deployment changes.The client wants the oversized-retry class rejects while the small-first-attempt class keeps queueing.
 2. We need a way to say reject-instead-of-queue in a form of a policy config. The EPP cannot defer requests because it runs inside an ext_proc callout with a timeout budget, where parking a request burns that budget and then likely times out anyway. The Frontend leaves it off because it owns the stream and can legitimately hold. Today you cannot ask for a class-aware shed threshold without also asking for deferral. 

**Cons:**

* Shedding stops being a gateway-level plugin. A user-supplied policy becomes a router
  admission policy registered at a composition root, which is a different authoring
  model from the one the rest of this DEP describes.
* The shed necessarily happens after tokenization and the index query, so a rejected
  request has already paid for both. Gate A still bounds that cost for the fully
  saturated case.

**Open — not yet rejected.** 

There is also a coordination risk. `lib/kv-router/src/scheduling/queue_admission/` is
described as the established boundary for policy-class admission algorithms, but the
module currently contains only `RequestProgress` and `WorkerPlacement` — the contract is
designed and unbuilt, and its decision set is `Bypass` / `Ready` / `Defer` with no
`Reject`. Alt 4 amounts to adding `Reject` to that contract, so it likely belongs folded
into whichever proposal owns it rather than pursued from the EPP side. Identify that
owner before choosing between Alt 4 and the main proposal.
