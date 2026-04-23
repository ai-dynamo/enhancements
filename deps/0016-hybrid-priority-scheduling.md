# Hybrid Priority Scheduling in Dynamo

**Status**: Draft

**Authors**: atchernych

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: https://github.com/ai-dynamo/dynamo/issues/8580

# Summary

Use the upstream [InferenceObjective](https://github.com/kubernetes-sigs/gateway-api-inference-extension) CRD
(`inference.networking.x-k8s.io/v1alpha2`) to drive per-request priority
in Dynamo's KV-aware router **without** requiring the full GAIE EPP stack.

Why CRD:
- Using CRD allows for runtime priority changes without redeploying anything. An ops engineer can change a priority by editing one YAML object at any time. No container restart, no config reload, no redeploy. With a CLI flag or env var, you'd need to restart the frontend pod to change priorities.
- Compatibility with the Gateway API ecosystem, we want to be kubernetes friendly as much as we can.
- For local dev and quick experiments, the body-level agent_hints.priority is simpler. The CRD is for production multi-tenant deployments where you want priority to be managed as infrastructure, not embedded in application code.

Clients pass `x-gateway-inference-objective: <name>` in the HTTP header.
The frontend resolves the objective name to a priority integer from a
Kubernetes-watched in-memory cache and injects it into the existing
`priority_jump` pipeline.  The scheduler queue honours priority for both
ordering and load-shedding.

# Motivation

- Dynamo already has a priority-ordered scheduler queue (`priority_jump`)
  but no mechanism to resolve priority from Kubernetes-native metadata.
- PR [#8144](https://github.com/ai-dynamo/dynamo/pull/8144) adds
  queue-depth backpressure, but it is priority-blind — a high-priority
  realtime request is shed the same as a batch job.
- [Issue #8189](https://github.com/ai-dynamo/dynamo/issues/8189)
  (Request Rejection Refactoring) proposes moving rejection to a
  `RejectionLayer` before tokenization — a hard gate based on
  utilization and queue-depth signals. This proposal adds priority awareness to both tiers.
- The GAIE `InferenceObjective` CRD is the emerging Kubernetes-native
  way to express per-use-case priority and SLOs.  Adopting it lets
  Dynamo participate in the Gateway API ecosystem without coupling to
  the EPP.
- In addition to the `InferenceObjective` CRD we would need to install the `InferencePool` CRD where the DGD would reside.

## Goals

* Introduce per-request priority resolution from Kubernetes-native `InferenceObjective` CRDs.
* Implement two-tier priority-aware rejection (hard gate before tokenization, soft shedding in scheduler queue).
* Maintain backward compatibility with body-level `agent_hints.priority`.
* Distinguish 503 (system overload) from 429 (priority shedding) for operational clarity.
* Add Prometheus rejection metrics with `tier` and `reason` labels.

### Non Goals

* Replacing or reimplementing the full GAIE EPP stack.
* Modifying the existing scheduler queue ordering algorithm beyond priority-aware admission.

## Requirements

### REQ 1 CRD-Based Priority Resolution

The system **MUST** resolve per-request priority from `InferenceObjective` CRD resources watched via a kube-rs reflector. Priority **MUST** be cached in an in-memory `HashMap` and **MUST NOT** require a K8s API call on the request hot path.

### REQ 2 HTTP Header Contract

The system **MUST** accept an `x-gateway-inference-objective` HTTP header whose value is the name of an `InferenceObjective` resource. The header **MUST** be resolved before tokenization.

### REQ 3 Body-Level Override Precedence

If the request body already contains `nvext.agent_hints.priority`, the body value **MUST** take precedence over the CRD-resolved value.

### REQ 4 Two-Tier Rejection

The system **MUST** implement two distinct rejection tiers:
- **Tier 1 (Hard Gate):** Before tokenization, returning HTTP 503 under gross overload for low-priority requests.
- **Tier 2 (Soft Shedding):** Inside the scheduler queue, returning HTTP 429 when the queue is full for low-priority requests.

### REQ 5 High-Priority Exemption

Requests with priority > 0 **MUST** be exempt from both Tier 1 rejection and Tier 2 queue shedding.

### REQ 6 Rejection Metrics

The system **MUST** expose a `dynamo_requests_rejected_total` Prometheus counter with at least `tier` and `reason` labels.

### REQ 7 Graceful Degradation

If the `InferenceObjective` CRD is not installed (API 404), the system **MUST** log a warning and fall back to body-level or default priority without crashing.

# Proposal

## Relationship to Issue #8189 (Request Rejection Refactoring)

This proposal assumes PR #8144 (queue-depth backpressure) is
implemented and builds on top of
[issue #8189](https://github.com/ai-dynamo/dynamo/issues/8189)
(Request Rejection Refactoring).  The two proposals interact at every
level — rejection placement, signals, status codes, and guarantees.

### The pipeline after #8189

Issue #8189 moves rejection from the `PushRouter` to a new
`RejectionLayer` **before tokenization**:

```
HTTP → [RejectionLayer] → Tokenize → Migrate → Backend → PrefillRouter → PushRouter → Workers
        (hard gate)                                                       (routing + queue)
```

The `RejectionLayer` is a binary accept/reject gate.  Its signals are
utilization (`instance_ids_free()`) and queue depth (from #8144's
`router_max_queue_depth_per_worker`).  Once a request is admitted past
the gate, #8189 guarantees it will not be rejected later at the
frontend.

### Two-tier priority-aware rejection

This DEP proposes a 2-tiered system. We want to introduce the rejection in the existing scheduler queue in lib/kv-router/src/scheduling/queue.rs — the BinaryHeap that PR #8144 adds back pressure to.

```
                    Tier 1 (hard gate)                               Tier 2 (soft shedding)
                    Before tokenization                              Inside scheduler queue
                           │                                                  │
HTTP ──▶ [RejectionLayer] ──▶ Tokenize ──▶ ... ──▶ PushRouter ──▶ [Scheduler Queue] ──▶ Workers
              │                                                       │
              │  gross overload?                                      │  queue full?
              │  yes + low pri  → 503                                 │  yes + low pri  → 429
              │  yes + high pri → admit                               │  yes + high pri → admit
              │  no             → admit                                │  no             → schedule
```

| Tier | Where | When it fires | Priority-aware? | HTTP | Purpose |
|------|-------|---------------|-----------------|------|---------|
| **1 — Hard gate** | `RejectionLayer` (before tokenization) | Gross overload: no free workers **and/or** aggregate queue depth exceeded | Yes — high-priority requests bypass the gate | **503** | Prevent wasting tokenization CPU under system-wide overload |
| **2 — Soft shedding** | `SchedulerQueue::enqueue` (after tokenization) | Queue full per #8144's `router_max_queue_depth_per_worker` | Yes — high-priority requests are never shed | **429** | Fine-grained priority differentiation; protects high-value traffic |

### How priority reaches the RejectionLayer

The `x-gateway-inference-objective` header is available at the HTTP
layer, **before tokenization**, so the `RejectionLayer` can resolve
priority from the same in-memory `InferenceObjectivePriorityMap` —
no tokenization needed.  This is a cheap `HashMap` read, not a K8s
API call.

### Modified #8189 guarantee

Issue #8189 originally guarantees "once admitted, never rejected at
the frontend."  With priority-aware soft shedding this relaxes to:

> Requests with `high priority` are never shed once admitted.
> Requests with `lower priority` may be shed from the queue if it
> exceeds the depth cap — but only to make room for higher-priority
> work.

The shedding decision should be progressive. It should compare the request's priority against the current queue depth. The heavier the load, the higher the priority bar to get admitted.

This is a deliberate policy choice (protecting high-value traffic),
not a race condition.  The #8189 race — where a queued request is
rejected because the busy signal re-asserts — is still eliminated
because Tier 1 rejection and Tier 2 shedding are distinct
mechanisms with distinct semantics.

### Answers to #8189 open questions

| #8189 question | Answer from this proposal |
|----------------|--------------------------|
| **#5 — Convergence with PR #8144** | #8144's `router_max_queue_depth_per_worker` is consumed at both tiers.  Tier 1 uses aggregate queue depth as a coarse rejection signal.  Tier 2 uses the same cap for fine-grained priority shedding.  No overlapping knobs — each tier has a distinct role. |
| **#6 — Backpressure semantics** | 503 for system overload (Tier 1, hard gate).  429 for priority shedding (Tier 2, soft shedding).  429 signals "back off and retry" without triggering circuit breakers. |
| **#7 — Minimize knobs** | Priority adds one knob (InferenceObjective CRD) but *reduces* the need for manual threshold tuning.  Instead of tuning `active_decode_blocks_threshold` to protect realtime traffic, declare priority tiers and let the system shed the right requests automatically. |

## Kubernetes Resources

### InferencePool (required by `poolRef`)

```yaml
apiVersion: inference.networking.k8s.io/v1
kind: InferencePool
metadata:
  name: dynamo-llama3
  namespace: ai-prod
spec:
  targetPorts:
    - number: 8000
  selector:
    matchLabels:
      nvidia.com/dynamo-component-type: worker
```

### InferenceObjective (one per priority tier)

```yaml
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferenceObjective
metadata:
  name: realtime-chat
  namespace: ai-prod
spec:
  priority: 100          # highest — always served first
  poolRef:
    name: dynamo-llama3
---
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferenceObjective
metadata:
  name: standard-api
  namespace: ai-prod
spec:
  priority: 0            # default tier
  poolRef:
    name: dynamo-llama3
---
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferenceObjective
metadata:
  name: batch-jobs
  namespace: ai-prod
spec:
  priority: -10           # lowest — shed first
  poolRef:
    name: dynamo-llama3
```

Both objectives reference the **same** Dynamo deployment via the
shared `InferencePool`.

## Architecture

```
                         ┌──────────────────────────────────┐
                         │  Kubernetes API                  │
                         │  InferenceObjective resources    │
                         └────────────┬─────────────────────┘
                                      │ watch (kube-rs reflector)
                                      ▼
                         ┌──────────────────────────────────────────────────────────────────────┐
                         │  Dynamo Frontend (Rust)                                             │
                         │                                                                     │
┌─────────────┐   HTTP   │  ┌─────────────────────────────────┐    ┌─────────────────────────┐ │
│   Client    │─────────▶│  │ Tier 1: RejectionLayer          │    │ Tier 2: Scheduler Queue │ │
│  + header:  │          │  │ (before tokenization)           │    │ (after tokenization)    │ │
│  x-gateway- │          │  │                                 │    │                         │ │
│  inference- │          │  │ 1. Resolve priority from header │    │ • BinaryHeap ordered by │ │
│  objective  │          │  │ 2. If overloaded + low pri: 503 │───▶│   priority_jump         │ │
└─────────────┘          │  │ 3. If overloaded + high pri:    │    │ • If full + low pri:    │ │
                         │  │    admit anyway                 │    │   shed (429)            │ │
                         │  │ 4. If not overloaded: admit     │    │ • If full + high pri:   │ │
                         │  └─────────────────────────────────┘    │   admit anyway          │ │
                         │                                         └────────────┬────────────┘ │
                         └──────────────────────────────────────────────────────┼───────────────┘
                                                                               ▼
                                                                         ┌──────────┐
                                                                         │ Workers  │
                                                                         └──────────┘
```

## Why CRD over CLI flags or request-body priority

Priority is a property of the **workload**, not the code.  ML engineers
already think in terms of workloads — training jobs, batch inference,
realtime serving.  The CRD maps directly to that mental model.

**Without the CRD:** every client has to know its own priority and pass
it correctly in every request.  Priority logic ends up scattered across
inference clients — the chat app hardcodes `"priority": 100`, the batch
pipeline hardcodes `"priority": -10`, and when someone forgets or gets
it wrong, a batch job starves realtime traffic.

**With the CRD:** you define the priority once, give it a human-readable
name, and every request just says "I'm a `realtime-chat` request" via
one header.  When product decides realtime-chat should be higher
priority, nobody touches any inference client — one `kubectl patch` (or
a GitOps PR) updates the YAML object.

| Without CRD | With CRD |
|------------|----------|
| Every client hardcodes a priority number | Clients use a human-readable name (or nothing at all) |
| Changing priority = code change + redeploy | Changing priority = one `kubectl patch` |
| "What priority should I use?" conversation per team | Platform team manages the policy centrally |
| Batch jobs get randomly killed under load | Low-priority workloads are shed first, automatically |
| Priority config scattered across N codebases | Priority config is one YAML file in your GitOps repo |

For **local dev and quick experiments**, the body-level
`agent_hints.priority` still works and requires zero infrastructure.
The CRD is for **production multi-tenant deployments** where priority
should be managed as infrastructure, not embedded in application code.
Both paths coexist — the body always overrides the CRD, so power users
retain full control.

# Implementation Details

## 1. InferenceObjective CRD type (Rust, `lib/runtime`)

A kube-rs `CustomResource` derive struct.  Only the fields Dynamo
consumes are modelled; unknown fields are silently ignored for
forward compatibility.

```
InferenceObjectiveSpec
├── pool_ref: PoolRef
│     └── name: String
└── priority: Option<i32>        // unset treated as 0
```

Public exports:

| Symbol                               | Kind       | Description                                             |
|--------------------------------------|------------|---------------------------------------------------------|
| `InferenceObjective`                 | struct     | kube-rs generated CR type                               |
| `InferenceObjectiveSpec`             | struct     | `spec` body                                             |
| `PoolRef`                            | struct     | Reference to an InferencePool by name                   |
| `InferenceObjectivePriorityMap`      | type alias | `Arc<RwLock<HashMap<String, i32>>>`                     |
| `new_priority_map()`                 | fn         | Creates an empty shared map                             |
| `spawn_inference_objective_watcher()`| async fn   | Starts the kube-rs reflector background task            |
| `resolve_priority()`                 | async fn   | Looks up a name in the map; returns `Option<i32>`       |

Location: `lib/runtime/src/discovery/kube/inference_objective.rs`

## 2. Watcher contract

```
spawn_inference_objective_watcher(
    kube_client:   KubeClient,
    namespace:     &str,
    pool_name:     Option<String>,    // filter by poolRef.name
    priority_map:  InferenceObjectivePriorityMap,
    cancel_token:  CancellationToken,
)
```

Behavioural contract:

- The watcher is spawned inside the FrontEnd process.
- Uses `KubeClient` similar to discovery watcher and watches all `InferenceObjective` resources in `namespace`.
- If `pool_name` is `Some`, only objectives whose `poolRef.name`
  matches are loaded into the map.
- On each applied object, inserts/updates `map[metadata.name] = spec.priority`.
- If the CRD is not installed (API 404), logs a warning and exits
  gracefully.  Priority resolution returns `None` for all lookups,
  and the system falls back to body-level or default priority.
- Cancelled via `cancel_token`.

## 3. HTTP header contract

| Header                             | Value          | Example                                      |
|------------------------------------|----------------|----------------------------------------------|
| `x-gateway-inference-objective`    | objective name | `x-gateway-inference-objective: realtime-chat`|

Processing (in `lib/llm/src/protocols/openai/nvext.rs`):

```
apply_header_inference_objective(
    nvext:         Option<NvExt>,
    headers:       &HeaderMap,
    priority_map:  Option<&InferenceObjectivePriorityMap>,
) -> Option<NvExt>
```

Precedence rules:

1. If `priority_map` is `None` (feature disabled), pass through unchanged.
2. If the header is absent, pass through unchanged.
3. If the objective name is not in the map, log at `debug` level and
   pass through unchanged.
4. If the request body already has `nvext.agent_hints.priority` set,
   **body wins** (explicit per-request override).
5. Otherwise, set `nvext.agent_hints.priority` to the resolved value.

This feeds into the existing preprocessor path without any changes to
downstream code:

```
header → cache lookup → priority
  → nvext.agent_hints.priority
    → preprocessor (lib/llm/src/preprocessor.rs)
      → RoutingHints.priority_jump
        → SchedulingRequest.priority_jump
          → queue key (FCFS: priority_jump - arrival_offset)
```

## 4. Frontend State extension

`service_v2::State` gains:

| Method                                      | Description                                      |
|---------------------------------------------|--------------------------------------------------|
| `with_inference_objective_priorities(map)`   | Builder-style setter; returns `Self`             |
| `inference_objective_priorities()`           | Returns `Option<&InferenceObjectivePriorityMap>` |

The map is created and the watcher spawned at frontend startup.  When
the feature is not configured, the field remains `None` and no K8s
watches are created.

## 5. Tier 1 — Priority-aware RejectionLayer

Extends the `RejectionLayer` proposed in
[issue #8189](https://github.com/ai-dynamo/dynamo/issues/8189).

The `RejectionLayer` sits before tokenization and receives the raw
HTTP request including headers.  It resolves priority from the
`x-gateway-inference-objective` header using the same
`InferenceObjectivePriorityMap` shared with the HTTP handlers.
This is a cheap in-memory lookup — no tokenization, no K8s API call.

### Interface

```
RejectionLayer {
    client:       Client,                               // existing — tracks worker busy state
    priority_map: Option<InferenceObjectivePriorityMap>, // new — shared with HTTP handlers
}

impl Operator for RejectionLayer {
    fn forward(&self, request) -> Result<request> {
        if !system_is_overloaded():
            return Ok(request)

        priority = resolve_priority_from_header(request.headers)

        if priority > N:
            return Ok(request)       // high-priority bypasses the hard gate

        return Err(ResourceExhausted)  // 503
    }
}
```

The `system_is_overloaded()` check combines the signals from #8189:
- Utilization: `client.instance_ids_free().is_empty()` (existing)
- Queue depth: aggregate queue depth exceeds cap (from #8144)

### Latching

Consistent with #8189 and PR #7912, the rejection state latches:
once the system enters rejecting mode it stays rejecting until load
drops below a cool-off threshold.  Priority exemption still applies
during the latched period — high-priority requests pass through even
while the latch is held.

## 6. Tier 2 — Priority-aware queue shedding

Extends the queue-depth backpressure from PR #8144 inside the
`SchedulerQueue`.

### Config (from #8144, already implemented)

| Field                              | Type            | Default | Description                                                                                   |
|------------------------------------|-----------------|---------|-----------------------------------------------------------------------------------------------|
| `router_max_queue_depth_per_worker`| `Option<usize>` | `None`  | Max queued requests per worker slot.  Effective limit = value * current worker count.          |

### Error variant (from #8144, already implemented)

```
KvSchedulerError::Backpressure {
    reason:          RouterBackpressureReason,
    queue_depth:     usize,
    max_queue_depth: Option<usize>,
}
```

### Admission logic (`SchedulerQueue::enqueue`)

When all workers are busy and the request would be queued:

1. Compute `effective_max = max_queue_depth_per_worker * num_workers`.
2. Load current `queue_depth`.
3. If `queue_depth >= effective_max`:
   - If `request.priority_jump > 0.0` — **admit anyway** (high-priority
     requests are never shed by queue-depth backpressure).
   - Otherwise — **reject** with `KvSchedulerError::Backpressure`.

### Why Tier 2 exists alongside Tier 1

Tier 1 (the `RejectionLayer`) catches gross overload — all workers
saturated, system-wide.  It fires before tokenization to avoid
wasting CPU.

Tier 2 (queue shedding) handles a subtler case: the system is not
in gross overload (Tier 1 admitted the request), but the scheduler
queue has grown past its cap.  This happens when load is moderate
but concentrated — enough requests to fill the queue but not enough
to trip the utilization gate.  Tier 2 sheds the lowest-priority
queued requests to protect high-value traffic.

The two tiers never conflict:
- A request rejected at Tier 1 never reaches Tier 2.
- A request admitted at Tier 1 may be shed at Tier 2, but **only if
  its priority is <= 0**.  High-priority requests are exempt at both
  tiers.

## 7. HTTP status codes

| Tier | Condition                                | Error type          | HTTP status |
|------|------------------------------------------|---------------------|-------------|
| 1    | System overload, low priority            | `ResourceExhausted` | **503**     |
| 2    | Queue full, low priority                 | `Throttled` (new)   | **429**     |

**503 (Tier 1):** system-wide overload.  Fires before tokenization.
Signals "the service is genuinely overwhelmed."  Existing error type
`ResourceExhausted`, unchanged from #8189.

**429 (Tier 2):** priority-based queue shedding.  Fires after
tokenization.  Signals "back off and retry" to clients without
triggering circuit breakers or pulling the frontend out of load
balancer rotation.  A new `DynamoErrorType::Throttled` variant is
added to `lib/runtime/src/error.rs`.

The distinction matters operationally:
- Load balancers and proxies often react to 503 by marking the backend
  unhealthy and removing it from rotation.  That is appropriate for
  Tier 1 (genuine overload) but not for Tier 2 (policy-driven shedding
  where the frontend is healthy).
- Client SDKs (including OpenAI-compatible libraries) typically have
  built-in retry logic keyed on 429 with exponential backoff.  This is
  the correct client behaviour for priority-shed requests.

## Rejection Metrics

When the system rejects a request, operators need to know **why** and
**who got hurt**.  Without dedicated metrics, two very different
scenarios look the same in a dashboard: "requests are failing."

**Scenario A — capacity exhaustion (Tier 1):** all workers are maxed
out, the `RejectionLayer` is rejecting everything regardless of
priority.  This is a scaling problem.  You need more GPUs.

**Scenario B — priority shedding (Tier 2):** the queue is full of
realtime requests, so only some requests are being shed.  The
system is working as designed — priority shedding is protecting
high-value traffic.  This is **not** a scaling problem.

### Proposed counter

A Prometheus counter incremented on every rejection, with `tier` and
`reason` labels:

```
dynamo_requests_rejected_total{tier="hard_gate", reason="capacity_exhausted"}        12
dynamo_requests_rejected_total{tier="queue_shed", reason="priority_shed"}            847
```

If you only see `priority_shed`, the system is healthy and doing its
job.  If `capacity_exhausted` is climbing, you need to scale.

With optional additional labels for deeper analysis:

```
dynamo_requests_rejected_total{tier="queue_shed", reason="priority_shed", objective_name="batch-jobs", priority="-10"}     800
dynamo_requests_rejected_total{tier="queue_shed", reason="priority_shed", objective_name="standard-api", priority="0"}      47
dynamo_requests_rejected_total{tier="hard_gate", reason="capacity_exhausted", objective_name="", priority=""}              12
```

This tells you that batch jobs are absorbing almost all the shedding
(as intended), standard-api is occasionally clipped, and realtime-chat
never appears (high priority is never shed).

### Cardinality note

The `tier` + `reason` labels have bounded cardinality (2-3 values
each) — always safe.  Adding `objective_name` is more useful but
introduces a time series per objective.  This is fine for a handful
of objectives (3-10) but could stress Prometheus if someone creates
hundreds.  Recommendation: ship with `tier` + `reason` only and add
`objective_name` behind a feature flag or config option.

# Alternate Solutions

## Alt 1 CLI Flags / Environment Variables

**Pros:**

* Simple to implement — no K8s dependency.
* Works in non-K8s environments out of the box.

**Cons:**

* Requires container restart to change priorities.
* Priority logic scattered across deployment manifests.
* No runtime dynamism — cannot adapt to changing workload patterns.

**Reason Rejected:**

* Priority is a runtime property of workloads, not a deployment-time constant. The CRD model allows priority changes without any restarts or redeployments.

## Alt 2 Request-Body-Only Priority

**Pros:**

* Already partially implemented via `agent_hints.priority`.
* Zero infrastructure requirement.

**Cons:**

* Every client must know and correctly pass its own priority number.
* Priority policy scattered across N inference client codebases.
* No central management or audit trail.

**Reason Rejected:**

* Retained as a fallback/override mechanism but insufficient for production multi-tenant deployments. The CRD complements rather than replaces this path.

## Alt 3 Full GAIE EPP Integration

**Pros:**

* Full Gateway API ecosystem compliance.
* Richer SLO-based routing beyond just priority integers.

**Cons:**

* Requires deploying and operating the full EPP sidecar.
* Adds latency and operational complexity.
* EPP may duplicate functionality already in Dynamo's KV-aware router.

**Reason Rejected:**

* Overkill for the immediate priority scheduling need. The proposal adopts only the CRD type for forward compatibility without the runtime dependency.

# Background

## References

* [Gateway API Inference Extension (GAIE)](https://github.com/kubernetes-sigs/gateway-api-inference-extension) — upstream CRD definitions
* [PR #8144](https://github.com/ai-dynamo/dynamo/pull/8144) — Queue-depth backpressure
* [Issue #8189](https://github.com/ai-dynamo/dynamo/issues/8189) — Request Rejection Refactoring
* [PR #7912](https://github.com/ai-dynamo/dynamo/pull/7912) — Rejection latching behaviour

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **InferenceObjective** | A Kubernetes CRD (`inference.networking.x-k8s.io/v1alpha2`) that associates a named workload with a priority integer and a target InferencePool. |
| **InferencePool** | A Kubernetes CRD that represents a set of inference workers sharing the same model, referenced by InferenceObjective via `poolRef`. |
| **Priority Jump** | Dynamo's existing per-request priority field (`priority_jump`) that influences scheduler queue ordering. |
| **RejectionLayer** | A proposed request gate (from #8189) that sits before tokenization and rejects requests under system overload. |
| **Soft Shedding** | Priority-aware request rejection inside the scheduler queue (Tier 2) — sheds low-priority requests to protect high-value traffic. |

## Acronyms & Abbreviations

**CRD:** Custom Resource Definition

**DEP:** Dynamo Enhancement Proposal

**EPP:** Endpoint Picker Plugin (part of the GAIE stack)

**GAIE:** Gateway API Inference Extension

**SLO:** Service Level Objective

# Open Questions

1. **429 vs 503 for all backpressure** — Should *all* queue-depth
   shedding use 429 (not just priority-driven), or only when an
   InferenceObjective is involved?

2. **Dynamic priority updates** — Should the watcher handle deletes
   (remove from map) or only inserts/updates?  Deletes would cause
   in-flight requests with that objective name to fall back to default
   priority on next request.

3. **Non-K8s deployments** — For bare-metal / Docker Compose setups,
   should there be a static config file alternative to the K8s watcher
   (e.g. a JSON/YAML map of objective name to priority loaded at startup)?

4. **Tier 1 priority threshold** — The current design exempts requests
   with `priority > 0` at both tiers.  Should this threshold be
   configurable, or is `0` the right universal default?

5. **Multiple frontends** — #8189 open question #1 asks how rejection
   stays consistent across multiple frontends.  With priority awareness,
   each frontend independently resolves priority from its own watcher
   (all watching the same InferenceObjective resources).  The priority
   map is eventually consistent — is that sufficient, or do frontends
   need a shared view?
