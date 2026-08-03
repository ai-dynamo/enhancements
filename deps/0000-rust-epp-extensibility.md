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
a request under overload). The Admitter runs after the scheduler has picked a worker and
before the request is booked; "Why the Admitter runs after the scheduler" below explains
why it sits there rather than in front of the scheduler as llm-d does.

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
The classifier itself is implemented in `PolicyProfile::resolve_class_index`. The complication is its argument: it takes *uncached* tokens, and the EPP cannot know those at shed time. Overlap comes from the KV index query inside `pick()`, whereas the shed gate runs before the body is even decoded. 

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

This complication forces use to move the class-aware shedding logic after the workers are picked. This incurs additional overhead for a request we can potentially drop but this preserves our routing semantics. 
The route_decode() function returns the returns overlap_blocks we need for the `resolve_class_index`. The signature is `Result<(WorkerWithDpRank, overlap_blocks)>`. We can then run
```bash
let (decode_worker, overlap_blocks) = self.route_decode(/* ... */).await?;

let cached_tokens =
    overlap_blocks as usize * self.decode_router.block_size() as usize;
let uncached_tokens = tokens.len().saturating_sub(cached_tokens);
```

The following flow is proposed. `Router::pick()` is the `EndpointPicker` trait method in
`epp.rs`; everything below it runs inside that call.

```text
Router::pick()                             epp.rs
 ├─ GATE A request-blind shedding          epp.rs      (1)
 ├─ tokenize                               epp.rs
 ├─ Router::route_prefill()                epp.rs       thin wrapper
 │    └─ PrefillRouter::…                  kv-router    unchanged
 ├─ Router::route_decode()                 epp.rs       thin wrapper
 │    └─ KvRouter::find_best_match()       kv-router    unchanged
 │         → (worker, overlap_blocks)
 ├─ GATE B request-aware shedding          epp.rs      (2)  ← new
 ├─ resolve the worker endpoint            epp.rs
 └─ Router::add_request()                  epp.rs       bookkeeping
      └─ KvRouter::add_request()           kv-router    unchanged
```

1. **GATE A** — basic, request-blind shedding, already implemented in
   [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865). Not a plugin. It runs
   before tokenization, so it costs nothing to refuse a fully saturated pool. Every worker is continuously marked overloaded-or-not based on how full its KV cache and prefill queue are versus a fixed threshold, and Gate A rejects the incoming request with 429 only when every worker eligible to serve it is currently marked overloaded. Dynamo's frontend already sheds load (HTTP 529); the GAIE path needs the same, made extensible. The frontend's shed is a routing failure surfaced as an error, discovered after the request has already been parsed and tokenized. But customers pressing on explicit rejection versus implicit shed. The frontend returns 529 (configurable via DYN_HTTP_OVERLOAD_STATUS_CODE) with no retry hint. The EPP will return 429 with Retry-After. Both choices are intentional: 429 is what a gateway and its failover logic already understand, and the retry hint is new capability rather than parity. The beginning of its implementation is in [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865), which reuses the frontend's `KvWorkerMonitor` and adds a `PickError::Saturated { retry_after_secs }` variant surfaced as HTTP 429 with `Retry-After`. What remains is the wiring for the plugin. Caveat: One frontend capability also does not carry over: `POST`/`GET /busy_threshold` returns thresholds per model on a live fleet, whereas the EPP reads env vars once at startup. Closing that is out of scope here.
2. **GATE B** — the new class-aware shedding, exposed as a plugin. It sits after
   selection so it can derive uncached ISL from `overlap_blocks`, and before
   `add_request`, so nothing is booked yet and a rejection needs no rollback.

Everything marked `unchanged` is `dynamo-kv-router` code this proposal does not touch:
scoring, selection, and the router's own bookkeeping all stay as they are.


The per-class thresholds should be configured in the router's policy YAML — which already carries a
per-class `admission:` envelope rather than in EPP environment variables, since
splitting class definitions from class thresholds invites drift. `RouterPolicyConfig::from_yaml`, `resolve_profile`, `PolicyProfile`, and both class-index methods are all public, so the EPP can use them. But `KvRouterConfig::loaded_policy_config()` is private, so the EPP either gets that made public or loads the YAML itself.

This DEP does not pick one yet; the choice should be made before implementation starts.

# Proposal

## Overview

Keep the existing request flow and insert two plugin points around the existing
scheduler — data preparation in front of it, admission behind it — both reading one
shared worker-state view:

```text
ext_proc request
  → parse body / build request            built in
  → GATE A: global saturation shed        built in    request-blind, dynamo#11865
  → DataProducer plugin                   pluggable   tokenization lives here    [view]
  → Scheduler: pick worker                built in    existing KV router, unchanged
  → GATE B: Admitter (load shedding)      pluggable   reject under overload      [view]
  → book the request + attach headers     built in
  → return decision to Envoy
  → response callbacks                    built in    prefill complete, usage

[view] = reads the shared read-only worker-state view
```

This takes the useful part of the llm-d / GAIE flow — a data-preparation stage and a
separate stage that may reject — without the surrounding machinery. For reference,
llm-d's own ordering is:

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

The load-shedding plugin proposed here is that `Admitter`: same ability to reject
outright, same read-only view of saturation, same ordering relative to data
preparation. It diverges upstream in one respect — it runs *after* the scheduler rather
than in front of it.

### Why the Admitter runs after the scheduler

Dynamo's cost proxy for a request is uncached ISL: the prompt tokens that actually need
prefilling. That number is derived from `overlap_blocks`, which does not exist until
`KvRouter::find_best_match()` has consulted the prefix index. Admitting in front of the
scheduler would see raw prompt length only, and raw length mis-prices exactly the
requests a class-aware policy cares about — a 100k-token prompt that is 95% cached is
cheap, while an 8k-token prompt that is entirely uncached is not. Getting overlap earlier
would mean either a second prefix-index query or reordering the router, and leaving
`dynamo-kv-router` untouched is a constraint of this proposal. The gate therefore sits
after selection and before `Router::add_request()`, where nothing has been booked yet, so
a rejection needs no rollback.

Deciding late costs something: a rejected request has already paid tokenization and the
index query. Gate A bounds that cost. When every eligible worker is saturated the
built-in shed refuses before any of that work happens, so the late gate only pays on
requests that had a plausible home. llm-d can admit earlier because its
`latency-slo-admitter` does not consult prefix overlap.


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
| Scheduler (worker pick) | No (for now) | The existing KV-aware router stays the default; can be revisited later |
| Admitter (load shedding, Gate B) | Yes | Users need their own overload policy |
| Worker-state view | No — read-only input | Not a stage. One shared source of truth for the signals policies need (REQ 1) |
| Bookkeeping / headers | No | Internal correctness |

The Rust EPP is currently `ext_proc` + scheduler; this proposal adds exactly two
pluggable stages around the scheduler — data preparation in front of it, admission
behind it — plus one read-only input feeding both, and leaves the scheduler itself built
in. We are not making parsing, scoring, or picking pluggable at this time.

## How the plugins work (conceptually)

* A plugin is a small piece of user code selected by name in configuration.
* **DataProducer** plugins run first and can attach prepared data (starting with
  token IDs / multimodal metadata) to the request. Tokenization ships as the
  default PrepareData plugin. If token data is already present (e.g. a pre-tokenized
  request), the plugin is skipped. PrepareData is fail-open: if a plugin errors, the
  request still proceeds and scheduling falls back to prompt-based behavior.
* The scheduler then runs unchanged, consuming the token data that DataProducer produced
  and yielding a selected worker together with its `overlap_blocks`.
* **Admitter (Load-shedding)** plugins run last and default to allow: each plugin may reject
  (mapped to HTTP 429 / 503, with 529 available for Dynamo clients); if none
  reject, the request proceeds. Multiple plugins can be chained and any rejection
  stops the request. Because the gate precedes `Router::add_request()`, a rejection
  leaves no booking to undo.
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

### What keeps this from becoming a scoring API which we do not want to change

Exposing state is not the same as making selection pluggable, which stays a Non Goal.
Three properties hold that line:

* **Read-only.** A plugin observes. It cannot mark a worker overloaded, retune a
  threshold, or touch the KV index, so it has no way to steer selection through the
  state view.
* **A snapshot, not a live handle.** One frozen picture per decision, so two reads
  within a single decision agree and there is nothing to poll in a loop. It is also
  cheap to pass: a refcount bump rather than a map traversal.
* **Typed and narrow.** Named accessors for signals we already maintain, not a
  `HashMap<String, Value>`. Adding a fact becomes a reviewable code change instead of
  a plugin quietly depending on a key nobody knew existed.

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
* The gap is also narrower than it looks. `DataProducer` and `Admitter` are the only two
  plugin points llm-d exposes anywhere around scheduling, and this proposal adopts both —
  differing only in where the `Admitter` sits relative to the scheduler, for the reason
  given above. What Alt 1 adds beyond that is the flow-control machinery and pluggable
  scoring — both already Non Goals, and the former not pluggable upstream either.

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
