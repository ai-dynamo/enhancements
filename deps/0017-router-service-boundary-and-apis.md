# The Router as a Standalone Service: Ownership Boundary and API Surface

**Status**: Draft

**Authors**: [atchernych](https://github.com/atchernych)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [TBD — assign a code owner / maintainer]

**Required Reviewers**: [TBD — router and inference-gateway owners]

**Review Date**: [TBD]

**Pull Request**: [TBD]

**Implementation PR / Tracking Issue**: [TBD — link to ai-dynamo/dynamo tracking issue].
Related in-flight work:
[dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865) — load shedding in the
EPP (DEP-1073), and
[dynamo#12663](https://github.com/ai-dynamo/dynamo/pull/12663) — per-class request
shedding from the EPP (DYN-1073).

# Summary

Dynamo now has three hosts that need routing decisions: 
- the Frontend
- the Rust EPP (`dynamo-ext-proc`) with the Dynamo Runtime
- the standalone selection service (EPP without the runtime is the first example). 

We want to make sure that router capabilities do not get reimplemented per host or silently dropped.

This proposal does two things. First, it states an **ownership rule** among these hosts. 
Second, it proposes the some APIs the router should expose or change so that every host consumes the same one.

# Motivation

The Frontend today is roughly three things:

1. HTTP server and protocol handling
2. Pre/post-processing — tokenization, chat-template application, tool-call and
   reasoning parsing, detokenization 
3. Admission control, routing, worker request handling (EPP also does this)

Items 1 and 2 are cleanly separable from the router and are not in scope here.
Item 3 is not one thing: it is a dozen inter-related concerns whose ownership has
never been written down. 


## Goals

* State an ownership rule for Frontend responsibility 3 that a reviewer can
  apply to a new feature without re-litigating the boundary.
* Give the router one decision entry point that all hosts use, so a routing
  behavior added once is available to every host.
* Make refusals, bookings, and lifecycle transitions explicit and typed, so
  hosts translate rather than reinvent them.
* Supply the read-only worker-state view that
  [`0000-rust-epp-extensibility`](0000-rust-epp-extensibility.md) requires for
  its Admitter plugins.
* Preserve the router's ability to run in-process (Frontend, EPP Dynamo mode) or
  over the wire (selection service) behind the same interface.

### Non Goals

* Making scoring or endpoint selection pluggable. Plugin seams inside the EPP
  are [`0000-rust-epp-extensibility`](0000-rust-epp-extensibility.md); this
  proposal defines the interface those plugins consume.
* Where per-request priority originates. That is
  [`0016-hybrid-priority-scheduling`](0016-hybrid-priority-scheduling.md); here,
  priority is an input field the host fills.
* Changing the ext-proc contract, making the EPP terminating, or altering the
  Frontend's HTTP surface.
* Moving pre/post-processing (Frontend responsibilities 1 and 2).

## Requirements

EPP Integration with the Standalone Router and customer requests revealed the following issues.

### REQ 1 Meed for a Single Decision Entry Point

**The router has four entry points.** `KvRouter` exposes
`find_best_match`, `find_best_match_details`,
`find_best_match_details_with_policy_class`, (17! positional parameters)
`find_best_match_details_without_admission`.  
We have the positional-argument explosion issue. 

Existing `find_best_match*` variants **MUST** become thin
compatibility wrappers and **SHOULD** be removed once all hosts migrate.
The FrontEnd proposes a positional list, named instead of ordered.

```rust
struct BestMatchArgs<'a> {
    context_id: &'a str,
    routing_parts: RoutingRequestParts<'a>,
    router_config_override: Option<&'a RouterConfigOverride>,
    update_states: bool,
    return_routing_hashes: bool,
    lora_name: Option<String>,
    cache_namespace: Option<String>,
    priority_jump: f64,
    strict_priority: u32,
    policy_class: Option<String>,
    session_id: Option<String>,
    expected_output_tokens: Option<u32>,
    pinned_worker: Option<WorkerWithDpRank>,
    allowed_worker_ids: Option<HashSet<WorkerId>>,
    routing_constraints: RoutingConstraints,
}
```
We should promote the `BestMatchArgs` to public. 


### REQ 2 Atomic Booking

Selecting a worker and booking it MUST be one call. The booking id MUST NOT come from a client-supplied value such as x-request-id; either the router or the caller may mint it, as long as the client cannot set it.

Two things go wrong otherwise. A client-supplied id is not unique — retries reuse it on purpose — so one request finishing can free another's booking, and the router thinks a worker is free when it is not. And if selecting and booking are separate calls, two requests can pick the same worker in the gap between them, because neither one has been counted yet.

Both EPP modes show the point. Standalone mode already does this correctly: it mints its own id and passes it into a single select-and-book call. Dynamo mode cannot, because KvRouter only offers query-then-add_request, so it books after the fact using x-request-id and carries a TODO saying so. This is because the two modes sit on two different router APIs, and only one of them lets the correct pattern be written at all. Standalone is the proof that the shape works; this requirement makes it available everywhere.

### REQ 3 A Body-Free Admission Probe

The router **MUST** expose an admission probe that requires no request body and
no tokens, so a host can refuse a saturated pool before tokenizing and worker selection.

A client asked for a functionality in EPP that would run prior to routing to determine if all workers are busy so that they do not waste prefill compute. It was implemented in [#11865](https://github.com/ai-dynamo/dynamo/pull/11865/) We should implement a probe() API in the Router.

Rust EPP code:
```rust
fn shed_if_saturated(&self) -> Option<PickError> {
    if !self.worker_monitor.is_configured() { return None; }            // move to the router
    let counts = self.decode_router.client().routing_instance_counts(); // move to the router
    if counts.discovered > 0 && counts.free == 0 {                      // router — the predicate
        return Some(PickError::AllWorkersOverloaded {                   // host — protocol mapping
            retry_after_secs: self.shed_retry_after_secs,               // host — config
        });
    }
    None
}
```

We would want something like `router.probe()`:
```rust
fn shed_if_saturated(&self, class: &CallerClass) -> Option<PickError> {
    self.router.probe().refused().map(|r| PickError::from_refusal(r, self.shed_retry_after_secs))
}
```

The router would have:
```rust
fn probe(&self, family: Option<&str>) -> Option<Refusal>;
```

The FrontEnd can also use this `probe()` call prior to booking and benefit from the shedding gate. 


### REQ 3 B ###
The client using EPP wants to retry in case of request refusal.
I implemented the SHED_RETRY_AFTER_SECS. This should really be a dynamic field in the router's `QueueRejection` since it knows queue depth. 

```rust
pub struct QueueRejection {
    pub policy_class: String,
    pub limit_kind: QueueLimitKind,
    pub current: u64,
    pub limit: u64,
    pub retry_after: Option<Duration>,  // new
}
```

### REQ 3 C ###
When the router turns a request away, it should say so as a plain answer, not as an error. Today it returns an error, so every caller has to guess what that error means — and they guess differently: selection service returns 503, FrontEnd 529, EPP returns 429. The 503 is wrong, because it tells the gateway the server is broken when the router only meant "we're full, come back soon." So the router should return a refusal that says what it is and how long to wait, and each caller just translates that into its own protocol. Same meaning everywhere, and no one has to guess.

The functions`KvRouter::find_best_match` and `KvPushRouter::select_best_match` return 
`FindBestMatchOutcome::QueueRejected { rejection } => Err(rejection.into())` 
The router should return something like

```rust
pub(super) enum SelectionOutcome {
    Selected(WorkerSelection),
    Refused(QueueRejection),
}
```

### REQ 3 D ###
The decision what class the request belongs should have one implementation in the router. 
"Which class is this request" is now answered in three places: `policy_class_from_headers` in epp.rs, a second `policy_class_from_headers` in `lib/kv-router/src/services/selection/server.rs`, and the Frontend's metadata path at push_router.rs:159 `(request.metadata().get("policy-class"))`
The host should pass a hint, and the router should resolve the class.
```rust
pub struct RequestClassHint {
    pub family: Option<String>,     // explicit, from x-dynamo-meta-policy-class
    pub retry_attempt: Option<u32>, // observed; router decides what counts as a retry
}

impl RequestClassHint {
    /// Owns the wire names: x-dynamo-meta-policy-class, x-dynamo-retry-attempt,
    /// x-envoy-attempt-count. Hosts pass headers, router decides.
    pub fn from_headers(headers: &[(String, String)]) -> Self;
}
```

We should take it as the routing input, replacing today's `policy_class: Option<String>` parameter:

```rust
// before
policy_class: Option<String>,
// after
class_hint: RequestClassHint,
```


### REQ 4 Router-Owned Decision Metrics

We need to make sure that the hosts (epp and the standalone router clients) can get these metrics.

There are two registration mechanisms.

- Mechanism A — host-supplied prometheus::Registry. The host creates a registry, calls a register function, and the router writes into the shared metric objects on each event. Three groups use it, and any host can call all three today:

register_worker_load_metrics(&registry)
register_router_queue_metrics(&registry) — includes the per-class backpressure counter
RoutingOverheadMetrics::register(&registry, instance_id)
The Frontend calls all three during HTTP service setup (service_v2.rs), serving them on port 8000. The EPP already has a registry and a /metrics endpoint on port 9090 in both modes, so it can call these now.

One caveat: instance_id becomes the router_id const label, and the Frontend sources it from config.drt_discovery.instance_id(). Standalone mode has no discovery and needs its own stable identifier — pod name hash or similar. It's a label, not a dependency, but someone has to choose.

- Mechanism B — DRT Component. RouterRequestMetrics::from_component(&Component) digs out component.drt().discovery().instance_id() and registers through the DRT MetricsHierarchy. This is the only real gap.

EPP in Dynamo mode: fixable with a small change. The DRT is available — from_discovery builds it as a local, uses it, and lets it drop at return. Retaining the component handle would work today.
EPP in standalone mode: no DRT ever. Needs a registry-based constructor.
Standalone router (python -m dynamo.router): has a DRT, so B works, but it gets none of the Mechanism A metrics because those are registered by the Frontend's HTTP setup.


# Proposal

## The ownership rule

> If the Frontend and the EPP would both have to answer a question, and
> answering it differently is a bug, it belongs in the router. If the answer
> legitimately differs by protocol or deployment, it belongs in the host.

Corollary: **the router returns typed decisions, never protocol artifacts.**

Applying that test to responsibility 3 yields the three lists below.

## What the router owns

**Fleet state**

* Worker registry and per-worker state: readiness, active decode blocks, active
  prefill tokens, queue depth, DP ranks
* Model registry — which models this pool serves, as the authoritative input for
  host-side validation
* KV / prefix index and its event ingestion; overlap computation; **uncached-ISL
  derivation** (prompt tokens still needing prefill after overlap), which has no
  header representation and cannot be computed outside the router
* Cross-replica load-state synchronization

**Decisions**

* Saturation: whether a worker is busy, and whether a class is saturated across
  all eligible workers — one model, not two
* Class resolution for router-derived dimensions such as the ISL bucket; the
  caller-derived family arrives from the host
* Eligibility filtering over router-known attributes, the scoring / cost model,
  and the selection algorithm
* Queueing and ordering (FCFS, WSPT, priority tiers)
* The admission decision itself — admit, defer, or refuse — as a typed outcome,
  never a status code
* Disaggregation pairing (prefill plus decode) and conditional bypass
* Ranked alternates for failover, and reselection excluding a given worker

**Bookkeeping and visibility**

* Booking lifecycle: reserve, prefill-complete, free — with router-minted IDs
  and release on drop
* Decision metrics: overlap estimate, chosen worker, queue depth, booking
  gauges, and shed counters by policy class

## What the host owns

Host means the Frontend, the EPP, or any service which owns the client relationship. 

**Protocol and lifecycle**

* The protocol surface, and **where in the request lifecycle the router is
  consulted**. Gating before the body is decoded, as
  [dynamo#11865](https://github.com/ai-dynamo/dynamo/pull/11865) does, is a host
  choice and the right one.
* Wire-to-facts extraction: policy family from `x-dynamo-meta-policy-class`,
  `x-dynamo-retry-attempt`, or `x-envoy-attempt-count`; priority hints; tenant
  cache salt; session ID; subset hint
* Status mapping and retry advertisement, plus message sanitization
* Request validation: body-size limits, malformed JSON, and mapping a model
  mismatch to a status — using the router's model registry as the source of
  truth

**Transport and lifetime**

* Worker identity to transport address (pod reflector versus discovery instance)
* Dispatch and response-stream ownership, or delegation to Envoy
* Disconnect detection, and reporting the abort so the router can release the
  booking
* Failover mechanics: forwarding the router's ranked alternates, whether by
  Envoy retry or Frontend re-dispatch
* Configuration and admin surface — environment variables versus a
  `/busy_threshold`-style endpoint
* Lifecycle metrics from its own vantage: TTFT, ITL, inflight, per-status counts

## What the contract carries

* A request-facts input: tokens, caller-derived class family, priority, tenant
  salt, session, subset constraint, expected output length
* Two-phase admission: a body-free probe, then a full admit
* A booking handle with release on drop, and a lifecycle event set that
  distinguishes completed, aborted, and retryable-abort — because "the client
  disconnected, but Envoy may retry" is a question no host can answer alone
* A rule that a host never forwards to an alternate the router has not accounted
  for

## Two topology consequences, not gaps

**Mid-stream migration** replays already-delivered tokens onto a replacement
worker, which requires owning the response stream. That is structurally
impossible for an ext-proc EPP and belongs to whoever holds the worker
connection. *Pre-first-token failover* is the part the EPP can and should do, via
the alternates in REQ 8.

**Worker-facing cancellation** likewise belongs to the holder of the worker
connection — Envoy's connection reset in the EPP path. What is not optional for
any host is reporting the abort so the booking is released.

Note also that migration deliberately excludes `ResourceExhausted` and
`Cancelled` from its retryable set: migration is a fault path and stays
orthogonal to shedding rather than being another flavor of it.


# Related Proposals

* [`0000-rust-epp-extensibility`](0000-rust-epp-extensibility.md) — plugin seams
  inside the EPP. Its Admitter plugins need the read-only worker-state view this
  proposal defines as `FleetView`; the two are complementary and should land in
  that order.
* [`0016-hybrid-priority-scheduling`](0016-hybrid-priority-scheduling.md) —
  where per-request priority originates. Under this boundary, resolving an
  `InferenceObjective` name to a priority is host-side fact extraction feeding
  `RequestFacts::priority`.
* [`0005-llm-request-migration`](0005-llm-request-migration.md) — the mid-stream
  migration mechanism that this proposal classifies as host-owned and
  stream-bound.

# Alternate Solutions

## Alt 1 Leave the Boundary Implicit

**Pros:**

* No migration cost; in-flight shedding work lands sooner.

**Cons:**

* Each new host re-derives the boundary; the 429/529/503 divergence and the two
  saturation models grow rather than shrink.
* Capabilities keep silently disappearing per host, as `fallbacks` and decision
  metrics already have.

**Reason Rejected:**

* The cost is already being paid in duplicated threshold systems and per-host
  behavior differences; deferring only raises it.

## Alt 2 Make the EPP Terminating

**Pros:**

* One host, so no boundary is needed.

**Cons:**

* Abandons the ext-proc contract that keeps the EPP portable across conformant
  gateways; GAIE is explicit that deviating from the ext-proc protocol is
  ill-advised.

**Reason Rejected:**

* Out of scope and contrary to the gateway integration strategy.

## Alt 3 Adopt the GAIE Plugin Taxonomy Wholesale

**Pros:**

* Alignment with llm-d and the upstream ecosystem.

**Cons:**

* Answers a different question — how the EPP is extended internally, not what
  the router owns. Plugins still need the interface proposed here.

**Reason Rejected:**

* Complementary rather than alternative; see
  [`0000-rust-epp-extensibility`](0000-rust-epp-extensibility.md).

# Background

## References

* [dynamo#11865 — Enable load shedding in the EPP](https://github.com/ai-dynamo/dynamo/pull/11865)
* [dynamo#12663 — Per-class request shedding from the EPP](https://github.com/ai-dynamo/dynamo/pull/12663)
* [Gateway API Inference Extension — Endpoint Picker Protocol](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/004-endpoint-picker-protocol)

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **Booking** | A router-held reservation of load against a chosen worker, created at admit and released at abort or completion. |
| **Caller class** | The half of a request's policy class supplied by the client or gateway (for example retry versus first attempt), as opposed to the router-derived ISL bucket. |
| **Host** | Whichever component owns the client relationship in a topology: the Frontend, the EPP, or the selection service. |
| **Probe** | A body-free, side-effect-free query asking whether a class is refusable given current fleet state. |
| **Uncached ISL** | Prompt tokens still requiring prefill after prefix-cache overlap is accounted for. |

## Acronyms & Abbreviations

**EPP:** Endpoint Picker

**GAIE:** Gateway API Inference Extension

**ISL:** Input Sequence Length

**TTFT / ITL:** Time To First Token / Inter-Token Latency
