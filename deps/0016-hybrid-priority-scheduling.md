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

Clients pass `x-gateway-inference-objective: <name>` in the HTTP header.
The frontend resolves the objective name to a priority integer from a
Kubernetes-watched in-memory cache and injects it into the existing
`priority_jump` pipeline.  The scheduler queue honours priority for both
ordering and load-shedding.

# Why CRD

Today Dynamo treats `nvext.agent_hints.priority` as already-trusted
input — the assumption is that *some other service in front of Dynamo*
(an authenticated gateway, an SDK proxy, a service mesh policy) is
responsible for populating, sanitizing, or stripping that field
before the request lands. Dynamo itself does not authenticate the
field, does not bound it (the upper end is unclamped), and does not
distinguish a tenant-asserted value from a platform-assigned one. In
other words, the priority you see in `nvext` is whatever the most
recent hop chose to put there, and Dynamo simply trusts it. That is
fine for single-tenant dev and for deployments where an external
gateway already does this work — but it leaves Dynamo with no
first-class story for production multi-tenant priority. The CRD is
that first-class story.

Priority is a property of the **workload**, not the code. ML engineers
already think in terms of workloads — training jobs, batch inference,
realtime serving. The CRD maps directly to that mental model. It also
gives operators four concrete properties that a CLI flag, environment
variable, or request-body field cannot:

1. **Runtime priority changes without redeploy.** An ops engineer can
   change a priority by editing one YAML object at any time — no
   container restart, no config reload, no redeploy. With a CLI flag
   or env var, every priority change requires restarting the frontend
   pod. With a request-body field, the change has to land in every
   client codebase. With the CRD, it's one `kubectl patch` (or a
   GitOps PR) and the watcher picks it up sub-second.
2. **Gateway API ecosystem compatibility.** The
   `InferenceObjective` CRD is the emerging Kubernetes-native way to
   express per-use-case priority and SLOs. Adopting it lets Dynamo
   participate in the Gateway API Inference Extension ecosystem
   without coupling to the full EPP stack — the same header contract
   works whether you're behind a GAIE gateway or just a plain
   Dynamo frontend. We want Dynamo to be Kubernetes-friendly as
   much as we can.
3. **Separation of dev ergonomics from production governance.** For
   local dev and quick experiments, the body-level
   `nvext.agent_hints.priority` is simpler and requires zero
   infrastructure. The CRD is for production multi-tenant deployments
   where priority should be managed as infrastructure, not embedded
   in application code. Both paths coexist; the operator chooses the
   posture per environment via `inference_objective_policy` (see §9).
4. **A future single source of truth for class metadata.** This is
   actually the strongest argument for the future
   `InferenceObjective` extensions noted in §Forward Compatibility:
   today it takes three separate config surfaces (DGDR `sla`,
   GlobalRouter JSON grids, per-pool Planner JSON) to describe one
   logical "this class wants TTFT 200ms" intent. A single
   `InferenceObjective` with `targetTTFT` would let one CRD instance
   be the source of truth that both the GlobalRouter (routing) and
   the per-pool Planners (scaling) consume. We get `priority` for
   free today, and the same integration point absorbs SLOs / quotas
   / LoRA authorization later — without rewriting any of the three
   config surfaces it replaces.

**Without the CRD:** every client has to know its own priority and pass
it correctly in every request. Priority logic ends up scattered across
inference clients — the chat app hardcodes `"priority": 100`, the batch
pipeline hardcodes `"priority": -10`, and when someone forgets or gets
it wrong, a batch job starves realtime traffic.

**With the CRD:** you define the priority once, give it a human-readable
name, and every request just says "I'm a `realtime-chat` request" via
one header. When product decides realtime-chat should be higher
priority, nobody touches any inference client — one `kubectl patch` (or
a GitOps PR) updates the YAML object.

## Trust: priority is platform policy, not client assertion

The deeper reason for the CRD is **authorization**, not just ergonomics.

In the current code path, a client can put any integer they like into
`nvext.agent_hints.priority` — including `i32::MAX`. The frontend
clamps the bottom (`priority.max(0)` in `lib/llm/src/preprocessor.rs:687`)
but not the top, so a single request with `priority: 2_000_000_000`
sits at the head of the FCFS heap permanently and starves every other
arrival. There is no per-tenant cap, no auth on the field, no rate
limit. It is identical in trust model to letting clients set their own
`Authorization: admin` header. In production deployments today the
de facto answer is "Dynamo trusts its inputs; put it behind an
authenticated edge" — but that's an implicit assumption nowhere
enforced in the code.

The CRD path inverts this:

- The client **names a policy** (`x-gateway-inference-objective: gold`).
- The frontend **resolves the name** against an in-memory map populated
  from RBAC-protected `InferenceObjective` resources in the cluster.
- The number is set by the *platform team* in a YAML object subject to
  Kubernetes RBAC, audit logs, and admission webhooks — not by the
  request payload.

A client can ask for `gold`, but only the principals with `create`
permission on `inferenceobjectives.inference.networking.x-k8s.io` in
that namespace can define what `gold` means. A client cannot guess a
name they don't have, cannot mint a new objective, and cannot exceed
the priority value the platform team sanctioned for it. That's the
difference between **"client claims a priority"** and **"platform
authorizes a class"**.

The CRD exists so that the priority number isn't asserted by the
request. The request only *names* a policy, and the policy is an
authenticated, RBAC-scoped, lifecycle-managed Kubernetes object.

This shifts the threat model from "we trust everyone who can reach
the HTTP endpoint" to "we trust everyone who has RBAC on
`InferenceObjective` resources in the namespace" — which is the
normal Kubernetes authorization story and integrates with whatever
identity, audit, and policy tooling the cluster already runs. See §9
(Trust Model) for the concrete knobs that enforce this in code.

| Without CRD | With CRD |
|------------|----------|
| Every client hardcodes a priority number | Clients use a human-readable name (or nothing at all) |
| Changing priority = code change + redeploy | Changing priority = one `kubectl patch` |
| Changing priority means restarting the frontend (CLI flag / env var) | Changing priority is a runtime YAML edit, picked up sub-second |
| "What priority should I use?" conversation per team | Platform team manages the policy centrally |
| Batch jobs get randomly killed under load | Low-priority workloads are shed first, automatically |
| Priority config scattered across N codebases | Priority config is one YAML file in your GitOps repo |
| Any client can claim any priority — including `i32::MAX` | Priority value is set by the *platform team* in a YAML object subject to Kubernetes RBAC, audit logs, and admission webhooks |
| No notion of "who is allowed to be priority 100" | RBAC on `inferenceobjectives.inference.networking.x-k8s.io` is the authorization surface |
| Outside the Gateway API ecosystem | Native interop with GAIE / Inference Gateway tooling |

For **local dev and quick experiments**, the body-level
`agent_hints.priority` still works and requires zero infrastructure.
The CRD is for **production multi-tenant deployments** where priority
should be managed as infrastructure, not embedded in application code.
Both paths coexist, but in production multi-tenant mode the operator
configures the body-level path off (see `inference_objective_policy`
in §9) so it cannot be used to bypass the CRD-resolved priority.

# Motivation

- Dynamo already has a priority-ordered scheduler queue (`priority_jump`)
  but no mechanism to resolve priority from Kubernetes-native metadata.
- PR [#8144](https://github.com/ai-dynamo/dynamo/pull/8144) adds
  scheduler-queue backpressure tiered by cache-miss tokens (it sheds
  expensive, low cache-hit requests first). It is priority-blind — a
  high-priority realtime request with low cache hit is shed the same
  as a low-priority batch job with low cache hit.
- [Issue #8189](https://github.com/ai-dynamo/dynamo/issues/8189)
  (Request Rejection Refactoring) proposes moving rejection to a
  `RejectionLayer` before tokenization — a hard gate based on
  utilization and queue-depth signals. This proposal adds priority awareness to both tiers.
- The GAIE `InferenceObjective` CRD is the emerging Kubernetes-native
  way to express per-use-case priority and SLOs. Adopting it lets
  Dynamo participate in the Gateway API ecosystem without coupling to
  the EPP runtime.
- The upstream `InferenceObjective` schema makes
  `spec.poolRef` **required** (the API server rejects an objective
  without one), and the referenced `InferencePool` resource must exist
  in the same namespace. The `InferencePool` CRD definition therefore
  has to be installed alongside the `InferenceObjective` CRD — but
  that does **not** mean Dynamo requires the EPP runtime. Dynamo runs
  with and without EPP; this DEP makes the `InferenceObjective` path
  work in both modes by having the Dynamo operator emit a stub
  `InferencePool` keyed to the DGD whenever the user opts into the
  priority feature (see REQ 9 and §10). In the priority-only mode the
  pool is plumbing — selector points at DGD worker pods, no
  `EndpointPickerRef` controller is required to back it. Users only
  ever write `InferenceObjective` YAML; the pool is operator-managed.

## Goals

* Introduce per-request priority resolution from Kubernetes-native `InferenceObjective` CRDs.
* Make the `InferenceObjective` path work whether Dynamo is deployed **with** EPP (full GAIE flow) or **without** EPP (priority-only flow). In both modes the user only writes `InferenceObjective` resources; the supporting `InferencePool` is generated by the Dynamo operator.
* Implement two-tier priority-aware rejection (hard gate before tokenization, soft shedding in scheduler queue).
* Maintain backward compatibility with body-level `agent_hints.priority`.
* Distinguish **503** (system overload, Tier 1) from **429** (queue-depth shedding, Tier 2) for all queue-depth backpressure — independent of whether the priority feature is enabled.
* Make the priority admission threshold configurable rather than hard-coding "priority > 0".
* Add Prometheus rejection metrics with `tier` and `reason` labels.

### Non Goals

* Replacing or reimplementing the full GAIE EPP stack.
* Modifying the existing scheduler queue ordering algorithm beyond priority-aware admission.

## Requirements

### REQ 1 CRD-Based Priority Resolution

The system **MUST** resolve per-request priority from `InferenceObjective` CRD resources watched via a kube-rs reflector. Priority **MUST** be cached in an in-memory `HashMap` and **MUST NOT** require a K8s API call on the request hot path.

The reflector **MUST** process deletes (removing the entry from the map). Deleted objective names that arrive on subsequent requests **MUST** fall back to default priority and **MUST NOT** retain their prior cached value.

The system **MUST** export an observability surface for the priority map — at minimum the current map size and the timestamp of the most recent successful apply event — so operators can detect stale or empty caches.

### REQ 2 HTTP Header Contract

The system **MUST** accept an `x-gateway-inference-objective` HTTP header whose value is the name of an `InferenceObjective` resource. The header **MUST** be resolved before tokenization.

### REQ 3 Configurable Body / Header Priority Policy

The system **MUST** support an `inference_objective_policy` config with three modes governing how a body-level `nvext.agent_hints.priority` interacts with the header-resolved priority:

- `BodyWins` — the body value (if present, including `Some(0)`) takes precedence over the CRD-resolved value. Suitable for single-tenant dev environments. **Default.**
- `HeaderWins` — the header-resolved value takes precedence over the body value when both are present. Suitable for production multi-tenant deployments where operators want clients to be able to set a body fallback but cannot use it to elevate priority.
- `HeaderOnly` — the body `nvext.agent_hints.priority` field is *stripped* on inbound requests; only the header-resolved priority is honored. Suitable for hardened multi-tenant deployments where the body path must not be a backdoor.

In all three modes, an absent header still falls back to the body value (or default `0`) — operators reach `HeaderOnly` semantics by stripping the body, not by ignoring the header.

Production multi-tenant deployments **MUST** configure either `HeaderWins` or `HeaderOnly`. With `BodyWins` the priority field is unauthenticated and can be set by any client to any value (see §9 Trust Model).

### REQ 4 Two-Tier Rejection

The system **MUST** implement two distinct rejection tiers:
- **Tier 1 (Hard Gate):** Before tokenization, returning **HTTP 503** under utilization-based overload. Low-priority requests are rejected; high-priority requests bypass the gate.
- **Tier 2 (Soft Shedding):** Inside the scheduler queue, returning **HTTP 429** when a queue-depth cap is exceeded. Low-priority requests are rejected; high-priority requests are admitted to the queue regardless of depth.

The 429 vs 503 split is a property of the *tier*, not of priority. Even with the priority feature disabled, all Tier 2 queue-depth shedding (the existing PR #8144 backpressure path) **MUST** map to 429 — Tier 1 utilization rejection retains its existing 503 mapping. This makes admission semantics independent of whether `InferenceObjective` is in use.

### REQ 5 High-Priority Exemption

The system **MUST** support a configurable `priority_admission_floor: i32` (default `1`). Requests whose effective priority is `>= priority_admission_floor` **MUST** be exempt from both Tier 1 rejection and Tier 2 queue shedding.

Note that this admission decision is *binary* (admit / shed) and intentionally separate from the *continuous* queue-ordering effect of `priority_jump` inside the scheduler heap. Two requests at `priority = 1` and `priority = 100` are both exempt from shedding, but the latter still sits ahead of the former in the queue.

### REQ 6 Rejection Metrics

The system **MUST** expose a `dynamo_frontend_requests_rejected_total` Prometheus counter with at least `tier` and `reason` labels. The prefix matches the existing router-side counters (e.g. `dynamo_frontend_router_queue_backpressure_total` from PR #8144) so all rejection telemetry shares a registry and namespace.

### REQ 7 Graceful Degradation

If the `InferenceObjective` CRD is not installed (API 404), the system **MUST** log a warning and fall back to body-level or default priority without crashing.

### REQ 8 Priority Value Ceiling

The system **MUST** support a configurable `priority_max: i32` ceiling (default reasonable value, e.g. `1000`). After the body / header / CRD resolution chain produces an `i32` priority, the value **MUST** be clamped to `[0, priority_max]` before being lifted into `RoutingHints { priority_jump, priority }`.

This is defense in depth: even with `inference_objective_policy = BodyWins` (the default for dev environments) and a buggy or hostile client setting `priority: 2_000_000_000`, the clamped value `priority_max` is the worst the queue ever sees. It bounds the damage from a misconfigured trust boundary without changing routine semantics — operators in multi-tenant deployments still pair this with `HeaderWins` / `HeaderOnly` for correctness.

### REQ 9 Operator-Generated InferencePool

The Dynamo operator **MUST** generate a stub `InferencePool` resource keyed to the `DynamoGraphDeployment` whenever the user opts into the priority feature, so that user-authored `InferenceObjective` resources have a valid `spec.poolRef` to reference.

A new optional field on the DGD spec controls this behavior:

```yaml
spec:
  priorityConfig:
    generatePool: true       # default false; opt-in
    poolName: my-pool        # optional override; defaults to "<dgd-name>-pool"
```

The operator behavior **MUST** be:

- When `priorityConfig.generatePool` is `true`, emit an `InferencePool` named `<poolName>` in the DGD's namespace, with selector labels matching the DGD's worker pods (the same selector the existing EPP-mode pool uses, see `deploy/operator/internal/dynamo/epp/inference_pool.go`).
- When EPP is also enabled (`spec.eppConfig` is set), the operator **MUST** generate exactly **one** pool — the existing EPP-mode pool — and reuse it for both the EPP wiring and as the `poolRef` target for `InferenceObjective`s. No duplicate pool should be created.
- When EPP is **not** enabled and `priorityConfig.generatePool` is `true`, the operator **MUST** still generate the pool, but with a self-referential or no-op `EndpointPickerRef` (e.g. pointing at the frontend service) since no EPP runtime exists to back it. The `EndpointPickerRef` field is required by the upstream `InferencePool` schema; populating it with the frontend service satisfies validation without introducing an EPP dependency.
- When `priorityConfig.generatePool` is `false` (default), the operator **MUST NOT** generate a pool. Users in this mode are responsible for creating the `InferencePool` themselves (or skipping the named-objective feature entirely and using only the body-level priority path).

In all cases the user only writes `InferenceObjective` YAML; the `InferencePool` is operator-managed plumbing tied to the DGD lifecycle (created on DGD apply, deleted on DGD delete).

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

The `RejectionLayer` is a binary accept/reject gate. Its signal is
utilization (`Client::instance_ids_free()` empty). Once a request is
admitted past the gate, #8189 guarantees it will not be rejected
later at the frontend. PR #8144's queue-depth backpressure
(`router_queue_by_incoming_missing_isl`) is a separate, in-queue
signal handled by Tier 2 — see the modified guarantee below.

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
| **1 — Hard gate** | `RejectionLayer` (before tokenization) | Utilization-based overload — no free workers (`Client::instance_ids_free().is_empty()`) | Yes — requests at or above the admission floor bypass the gate | **503** | Prevent wasting tokenization CPU under system-wide overload |
| **2 — Soft shedding** | `SchedulerQueueActor::handle_enqueue` (after tokenization) | All workers prefill-busy AND the per-tier ISL-token cap from PR #8144's `router_queue_by_incoming_missing_isl` is reached | Yes — requests at or above the admission floor are never shed | **429** | Fine-grained priority differentiation under moderate-but-concentrated load; protects high-value traffic |

### How priority reaches the RejectionLayer

The `x-gateway-inference-objective` header is available at the HTTP
layer, **before tokenization**, so the `RejectionLayer` can resolve
priority from the same in-memory `InferenceObjectivePriorityMap` —
no tokenization needed.  This is a cheap `HashMap` read, not a K8s
API call.

### Modified #8189 guarantee

Issue #8189 originally guarantees "once admitted, never rejected at
the frontend."  With priority-aware soft shedding this relaxes to:

> Requests at or above the configured `priority_admission_floor` are
> never shed once admitted past Tier 1. Requests below the floor may
> be shed at Tier 2 if the per-tier cache-miss-keyed ISL-token cap
> from PR #8144 is reached — they are never shed *retroactively*; the
> decision happens only at queue-entry time.

This is a deliberate policy choice (protecting high-value traffic),
not a race condition. The #8189 race — where a queued request is
rejected because the busy signal re-asserts — is still eliminated
because Tier 1 rejection and Tier 2 shedding are distinct
mechanisms with distinct semantics, and Tier 2 only fires at
admission, never on already-parked entries.

We deliberately do **not** introduce a continuous "progressive"
shedding curve (e.g. priority threshold scaling with queue depth).
The binary admission rule keeps the contract observable: an operator
can predict from `priority`, `priority_admission_floor`, and the
backpressure metric exactly which requests will be shed under load,
without modeling a feedback loop.

### Answers to #8189 open questions

| #8189 question | Answer from this proposal |
|----------------|--------------------------|
| **#5 — Convergence with PR #8144** | Each tier consumes a distinct signal. Tier 1 consumes worker free-pool state from `Client::instance_ids_free()` (utilization). Tier 2 consumes PR #8144's `router_queue_by_incoming_missing_isl` per-tier ISL-token cap (queue depth, already cache-miss-aware). Tier 2 wraps the existing #8144 backpressure path with a priority admission check — it does not introduce a second cap knob. |
| **#6 — Backpressure semantics** | **503** for utilization-based system overload (Tier 1, hard gate). **429** for queue-depth shedding (Tier 2, soft shedding) — applied uniformly whether the priority feature is enabled or not. 429 signals "back off and retry" without triggering circuit breakers; 503 signals "the service is genuinely overwhelmed" and is appropriate for load-balancer-driven backend ejection. |
| **#7 — Minimize knobs** | Priority adds two new knobs (`InferenceObjective` CRD content, `priority_admission_floor`) but *reduces* the need for manual threshold tuning. Instead of tuning `router_queue_by_incoming_missing_isl` tiers to protect realtime traffic, declare priority levels and let the system shed the right requests automatically. The cache-miss tiering still applies (it sheds expensive requests within a priority class first), so the two mechanisms compose rather than overlap. |

## Kubernetes Resources

### InferencePool (operator-generated, do not author by hand)

`InferenceObjective.spec.poolRef` is a required field, so an
`InferencePool` resource must exist in the namespace. Users do **not**
write this YAML themselves — the Dynamo operator generates it from
the DGD when `spec.priorityConfig.generatePool: true` is set
(see REQ 9 and §10). The result looks like this:

```yaml
# AUTO-GENERATED by the Dynamo operator from the DGD; do not edit.
apiVersion: inference.networking.k8s.io/v1
kind: InferencePool
metadata:
  name: dynamo-llama3-pool                     # <dgd-name>-pool by default
  namespace: ai-prod
  labels:
    nvidia.com/dynamo-graph-deployment-name: dynamo-llama3
  ownerReferences:
    - apiVersion: nvidia.com/v1beta1
      kind: DynamoGraphDeployment
      name: dynamo-llama3
      controller: true
spec:
  targetPorts:
    - number: 8000
  selector:
    matchLabels:
      nvidia.com/dynamo-component-class: worker
      nvidia.com/dynamo-namespace: dynamo-llama3
  endpointPickerRef:                           # required by schema
    kind: Service
    name: dynamo-llama3-frontend               # self-referential placeholder
    port:                                      # in priority-only mode (no EPP)
      number: 8000
```

In **EPP-enabled** mode the same pool is reused; its
`endpointPickerRef` points at the EPP service instead of the frontend
(this is what the operator does today for EPP, see
`deploy/operator/internal/dynamo/epp/inference_pool.go`).

In **priority-only** mode (no EPP), the pool is plumbing — nothing
reconciles `endpointPickerRef`, the field exists only because the
upstream schema requires it. Dynamo's frontend reads
`InferenceObjective.spec.priority` and ignores the pool entirely.

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
    name: dynamo-llama3-pool   # the operator-generated stub
---
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferenceObjective
metadata:
  name: standard-api
  namespace: ai-prod
spec:
  priority: 0            # default tier
  poolRef:
    name: dynamo-llama3-pool
---
apiVersion: inference.networking.x-k8s.io/v1alpha2
kind: InferenceObjective
metadata:
  name: batch-jobs
  namespace: ai-prod
spec:
  priority: -10           # lowest — shed first
  poolRef:
    name: dynamo-llama3-pool
```

All three objectives reference the **same** Dynamo deployment via the
operator-generated `InferencePool`. The user authors only the three
`InferenceObjective` resources; the pool is plumbing tied to the
DGD's lifecycle.

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
- On each applied (added or updated) object, inserts/updates
  `map[metadata.name] = spec.priority`.
- On each deleted object, removes `metadata.name` from the map.
  Subsequent requests carrying that objective name fall back to
  body-level or default priority.
- Exposes a staleness signal via two metrics:
  `dynamo_frontend_inference_objective_map_size` (gauge) and
  `dynamo_frontend_inference_objective_last_apply_timestamp_seconds`
  (gauge). Operators can alert on a stale `last_apply_timestamp` to
  detect a stuck reflector.
- If the CRD is not installed (API 404), logs a warning and exits
  gracefully. Priority resolution returns `None` for all lookups,
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
4. If the request body already has `nvext.agent_hints.priority` set
   to `Some(_)` (including `Some(0)`), **body wins** — pass through
   unchanged.
5. Otherwise (`agent_hints.priority` is `None` or the whole
   `agent_hints` block is absent), set
   `nvext.agent_hints.priority` to the resolved value.

The natural placement is alongside the existing
`apply_header_routing_overrides` in
`lib/llm/src/protocols/openai/nvext.rs:35` — that function already
mutates `nvext` from inbound headers before preprocessing, so adding
a sibling `apply_header_inference_objective` is a localized change.

Setting `agent_hints.priority` is a single write that feeds **two**
downstream consumers — both via the existing preprocessor in
`lib/llm/src/preprocessor.rs:687`:

```
header → cache lookup → priority (i32)
  → nvext.agent_hints.priority
    → preprocessor lifts into RoutingHints { priority_jump, priority }
        ├── priority_jump (f64, clamped at 0)
        │     → SchedulingRequest.priority_jump
        │       → queue key (FCFS: priority_jump - arrival_offset)
        │       → admission floor check (REQ 5)
        │
        └── priority (i32)
              → forwarded to backend engine generate(priority=...)
                (vLLM, sglang, trtllm)
```

So the same header drives both the **router queue ordering and
admission** and the **engine's in-flight scheduling priority** — no
extra plumbing.

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
    client:                    Client,                                // existing — tracks worker busy state
    priority_map:              Option<InferenceObjectivePriorityMap>, // new — shared with HTTP handlers
    priority_admission_floor:  i32,                                   // new — default 1
}

impl Operator for RejectionLayer {
    fn forward(&self, request) -> Result<request> {
        if !utilization_overloaded(self.client):
            return Ok(request)

        priority = resolve_priority_from_header(request.headers)
            .unwrap_or(0);

        if priority >= self.priority_admission_floor:
            return Ok(request)         // exempt — bypass hard gate

        return Err(ResourceExhausted)  // 503
    }
}
```

### Tier 1 signal: utilization only

The hard gate consumes a single signal:

- `client.instance_ids_free().is_empty() && !client.instance_ids().is_empty()` — same condition as `PushRouter::empty_free_pool_error` (`lib/runtime/src/pipeline/network/egress/push_router.rs:486-498`) and the rejection condition originally proposed in #8189.

Tier 1 deliberately does **not** consult queue-depth signals:

- Tier 1 runs before tokenization, so a request's cache-miss tier is unknown — the per-tier ISL-token caps from #8144 cannot be evaluated.
- In disaggregated mode there are two `SchedulerQueue` instances (prefill and decode, each labelled `worker_type` in metrics). Combining their depths into one Tier 1 signal would either over-reject (any-queue-full) or under-reject (both-queues-full), and that complexity is exactly what Tier 2 already handles correctly inside each queue.

This keeps Tier 1 cheap and stateless and gives each tier one clear job: Tier 1 protects against gross utilization overload, Tier 2 protects against per-queue depth overload.

### Latching

Consistent with #8189 and PR #7912, the rejection state latches:
once the system enters rejecting mode it stays rejecting until load
drops below a cool-off threshold. Latching applies only to the
utilization signal — Tier 2 has no analogous latch since admission
is evaluated per-request against the current cache-miss-tiered cap.
Priority exemption still applies during the latched period —
exempt requests pass through even while the latch is held.

## 6. Tier 2 — Priority-aware queue shedding

Extends the queue-depth backpressure from PR #8144 inside the
`SchedulerQueue`. The exact site is
`SchedulerQueueActor::handle_enqueue` at
`lib/kv-router/src/scheduling/queue.rs:434-449`.

### Config (from #8144, already implemented)

| Field                                  | Type                    | Default          | Description                                                                                              |
|----------------------------------------|-------------------------|------------------|----------------------------------------------------------------------------------------------------------|
| `router_queue_by_incoming_missing_isl` | `RouterQueueDepthTiers` | unbounded (`[]`) | Per-worker pending-ISL-token caps tiered by cache-miss tokens. Effective cap = `max_queue_depth * worker_count` for the matching tier. |

A tier entry is `RouterQueueDepthByMissingIslTier { missing_cache_tokens_floor, max_queue_depth }`. The cap is on **pending ISL tokens** (sum of `isl_tokens` for queued requests), not on request count. The matching tier is the one with the highest `missing_cache_tokens_floor` that the request's cache-miss token count clears.

### Error variant (from #8144, already implemented)

```
KvSchedulerError::Backpressure {
    reason:                RouterBackpressureReason,    // existing variant: MaxQueuedIslTokensExceeded
    queued_isl_tokens:     usize,
    max_queued_isl_tokens: Option<usize>,
}
```

This DEP adds one new variant to `RouterBackpressureReason`:

```
RouterBackpressureReason::PriorityShed
```

### Admission logic — explicit evaluation order

When `handle_enqueue` finds the request would otherwise be queued
(threshold-frac is configured and `all_workers_prefill_busy` returned
true), the admission check runs in this order:

```
1.  effective_priority = request.priority_jump as i32      // already clamped at 0
                                                           // by the preprocessor

2.  if effective_priority >= priority_admission_floor:
        enqueue                                            // exempt: skip cap
        return

3.  if router_queue_by_incoming_missing_isl is unbounded:
        enqueue                                            // capping disabled
        return

4.  cap = tier_cap_for_request(request)                    // existing helper
                                                           // queue.rs:678-691

5.  if pending_isl_tokens >= cap:
        respond Err(KvSchedulerError::Backpressure {
            reason: if effective_priority < priority_admission_floor
                    then PriorityShed
                    else MaxQueuedIslTokensExceeded,
            queued_isl_tokens:     pending_isl_tokens,
            max_queued_isl_tokens: Some(cap),
        })
        return

6.  enqueue
```

The two `RouterBackpressureReason` values let metrics distinguish "we
shed because the priority feature said so" from "we shed because the
cache-miss-tiered cap was reached for *every* priority class." In
practice with the priority feature enabled, exempt requests skip the
cap entirely (step 2), so step 5 returns `PriorityShed` for any
request that reaches it. The `MaxQueuedIslTokensExceeded` reason is
preserved for the priority-feature-disabled path so existing #8144
behavior is unchanged when the feature is off.

### How priority and cache-miss tiering compose

The cache-miss tiering from #8144 already sheds expensive (low cache
hit) requests before cheap ones. Layering priority on top yields:

- **High priority + any cache hit:** never shed (step 2 above).
- **Low priority + low cache hit:** falls into the most expensive
  tier with the tightest cap — first to be shed under load.
- **Low priority + high cache hit:** falls into a cheaper tier with
  a looser cap — survives longer.

So priority and cache-miss tiering compose multiplicatively rather
than overlap. Operators tune cache-miss tiers for *cost-shaping*
backpressure and priority for *value-shaping* backpressure.

### Why Tier 2 exists alongside Tier 1

Tier 1 (the `RejectionLayer`) catches utilization overload — no free
workers, system-wide. It fires before tokenization to avoid wasting
CPU.

Tier 2 (queue shedding) handles a subtler case: the system is not
in utilization overload (Tier 1 admitted the request), but the
per-tier ISL-token cap on the scheduler queue has been reached. This
happens when load is moderate but concentrated — enough queued work
to trip the cap on the request's tier without saturating worker
slots. Tier 2 sheds requests below the admission floor to protect
high-value traffic.

The two tiers never conflict:
- A request rejected at Tier 1 never reaches Tier 2.
- A request admitted at Tier 1 may be shed at Tier 2, but **only if
  its priority is below `priority_admission_floor`**. Exempt requests
  bypass capping at both tiers.

## 7. HTTP status codes

| Tier | Condition                                | Error type          | HTTP status |
|------|------------------------------------------|---------------------|-------------|
| 1    | Utilization overload, below admission floor | `ResourceExhausted` | **503**  |
| 2    | Queue cap exceeded (any reason)          | `Throttled` (new)   | **429**     |

**503 (Tier 1):** utilization-based system overload. Fires before tokenization.
Signals "the service is genuinely overwhelmed." Existing error type
`ResourceExhausted`, unchanged from #8189.

**429 (Tier 2):** queue-depth backpressure. Fires after
tokenization. Signals "back off and retry" to clients without
triggering circuit breakers or pulling the frontend out of load
balancer rotation. Mapped to a **new** `DynamoErrorType::Throttled`
variant.

This 429-for-all-queue-shedding rule applies whether the priority
feature is enabled or not. With the feature off, the existing #8144
`MaxQueuedIslTokensExceeded` backpressure still becomes 429 — see
"Compatibility note" below.

The distinction matters operationally:
- Load balancers and proxies often react to 503 by marking the backend
  unhealthy and removing it from rotation. That is appropriate for
  Tier 1 (genuine overload) but not for Tier 2 (policy-driven shedding
  where the frontend is healthy).
- Client SDKs (including OpenAI-compatible libraries) typically have
  built-in retry logic keyed on 429 with exponential backoff. This is
  the correct client behaviour for priority-shed requests.

### Concrete plumbing tasks

The HTTP 429 path does not exist in tree today (`ErrorType` in
`lib/runtime/src/error.rs:55-63` does not include `Throttled`, and
`from_anyhow` in `lib/llm/src/http/service/openai.rs:230-241` only
maps `ResourceExhausted` → 503). This DEP requires:

1. **Add `ErrorType::Throttled`** to `lib/runtime/src/error.rs` and
   its `Display` impl.
2. **Add a `request_was_throttled(...)` helper** in
   `lib/llm/src/http/service/metrics.rs` mirroring
   `request_was_rejected`. Update the metrics classification table
   (`REJECTION: &[DynamoErrorType] = &[...]`) so both
   `ResourceExhausted` and `Throttled` count as rejections for
   existing telemetry.
3. **Add a 429 branch to `from_anyhow`** in
   `lib/llm/src/http/service/openai.rs` that maps `Throttled` to
   `StatusCode::TOO_MANY_REQUESTS`, ordered before the existing 503
   branch.
4. **Update the router → DynamoError mapping** at
   `lib/llm/src/kv_router/push_router.rs:366-374` to emit `Throttled`
   instead of `ResourceExhausted` when the underlying scheduler
   error is `KvSchedulerError::Backpressure { .. }`.

### Compatibility note

Step 4 changes the HTTP status code for the existing PR #8144
backpressure path from 503 to 429 even when the priority feature is
disabled. This is a deliberate behavior change — the original 503
mapping was a placeholder while #8189 was being designed, and the
DEP positions all queue-depth shedding as 429 to keep admission
semantics independent of whether priority is in use. Mention this in
the PR description and changelog entry; clients that pin retry logic
to specific status codes will need to handle 429 as well as 503.

## 8. Staged delivery

The full proposal can land in two stages. Stage 1 is independent of
DEP #8189 and delivers most of the user-visible value; Stage 2 lands
once #8189's `RejectionLayer` exists.

### Stage 1 — Header → priority_jump + Tier 2 priority exemption + operator pool generation

Self-contained, does not depend on #8189. Splits cleanly across the
Rust frontend codebase and the Go operator codebase; the two halves
can be reviewed and merged in parallel.

**Frontend (Rust):**

- `lib/runtime/src/discovery/kube/inference_objective.rs` — kube-rs
  reflector and `InferenceObjectivePriorityMap` (§1, §2).
- `lib/llm/src/protocols/openai/nvext.rs` —
  `apply_header_inference_objective` sibling to the existing
  `apply_header_routing_overrides` (§3). Honors the
  `inference_objective_policy` config (`BodyWins` / `HeaderWins` /
  `HeaderOnly`); `HeaderOnly` strips
  `nvext.agent_hints.priority` on inbound requests.
- `lib/llm/src/preprocessor.rs:687` — apply the `priority_max` clamp
  on the resolved `i32` before lifting into `RoutingHints`
  (§9 / REQ 8).
- `lib/kv-router/src/scheduling/queue.rs:434-449` — wrap the existing
  `KvSchedulerError::Backpressure` path with the priority admission
  check (§6 step 2). Add `RouterBackpressureReason::PriorityShed`.
- HTTP 429 plumbing per §7: `ErrorType::Throttled`,
  `request_was_throttled`, `from_anyhow` branch, and update
  `kv_router/push_router.rs:366-374` to map
  `KvSchedulerError::Backpressure` → `Throttled`.
- Metrics per §6/REQ 6: `dynamo_frontend_requests_rejected_total`
  with `tier="queue_shed"` reasons.

**Operator (Go) — REQ 9 / §10:**

- `deploy/operator/api/v1beta1/dynamographdeployment_types.go` —
  add `PriorityConfig { GeneratePool, PoolName }` field to the DGD
  spec.
- `deploy/operator/internal/dynamo/epp/inference_pool.go` — add
  `GenerateStubInferencePool` next to the existing
  `GenerateInferencePool`. The new helper produces a schema-valid
  `InferencePool` with a self-referential `endpointPickerRef`
  pointing at the frontend service.
- `deploy/operator/internal/controller/dynamographdeployment_controller.go`
  — branch on `eppEnabled` vs `priorityConfig.generatePool` so EPP
  mode and priority-only mode each generate exactly one pool, with
  EPP-mode reuse where applicable.
- `deploy/operator/internal/webhook/validation/dynamographdeployment.go`
  — reject conflicting `eppConfig` + custom `priorityConfig.poolName`.
- `deploy/helm/` — surface `priorityConfig` in the DGD Helm values
  and document the cluster prerequisite (install the
  `inference.networking.k8s.io` CRD bundle).

After Stage 1: priority differentiation works inside the scheduler
queue, and users in both EPP and non-EPP deployments can author
`InferenceObjective` resources without writing pool YAML. There is
still no priority-aware hard gate — utilization overload still
rejects at `PushRouter::empty_free_pool_error` without consulting
priority. That is the same behavior as today.

### Stage 2 — Priority-aware Tier 1 RejectionLayer

Lands once DEP #8189 ships the `RejectionLayer` operator.

- Extend `RejectionLayer` per §5 to consult the priority map and the
  admission floor before returning `ResourceExhausted`.
- Add `tier="hard_gate"` reason to the rejection counter.

This split lets us discover real-world feedback on the priority
semantics from Stage 1 before committing to the harder Tier 1
design, and avoids a serial dependency on #8189's review timeline.

## 9. Trust Model

Priority is, by definition, a **mechanism for some requests to jump
others**. The moment we wire that mechanism through the queue we
inherit responsibility for who is allowed to use it. This section
spells out the threat, the defenses, and the operational guidance.

### 9.1 Threat: client-asserted priority

Today every component downstream of the frontend trusts
`nvext.agent_hints.priority` as if it were authentic. The lift in
`lib/llm/src/preprocessor.rs:687`

```rust
priority_jump: hints.and_then(|h| {
    h.priority
        .map(|priority| priority.max(0) as f64)
        .or(h.latency_sensitivity)
}),
```

clamps the bottom but not the top. A single curl with

```json
{"nvext": {"agent_hints": {"priority": 2000000000}}}
```

permanently sits at the head of the FCFS heap, and once this DEP lands
also bypasses both Tier 1 (via the admission floor) and Tier 2 (via
the priority-shed exemption). One unauthenticated client can reduce
the system to a single-tenant queue serving themselves.

The mitigation is **not** a code-level allowlist of priority values;
that hardcodes policy. The mitigation is to (a) move authoritative
priority assignment to a server-side, RBAC-protected object, and (b)
defend in depth so that even with a misconfigured trust boundary the
blast radius is bounded.

### 9.2 Defense in depth — three knobs

The implementation introduces three orthogonal config knobs; together
they cover the realistic deployment topologies:

| Knob | Type | Default | Purpose |
|------|------|---------|---------|
| `priority_max` | `i32` | `1000` | Hard ceiling on the resolved priority value (REQ 8). Clamp applied after body / header / CRD resolution, before lift into `RoutingHints`. |
| `inference_objective_policy` | `BodyWins \| HeaderWins \| HeaderOnly` | `BodyWins` | Controls precedence between body-level `nvext.agent_hints.priority` and header-resolved priority (REQ 3). |
| RBAC on `inferenceobjectives.inference.networking.x-k8s.io` | Kubernetes RBAC | n/a (cluster admin sets) | Authorization surface for what a named objective like `gold` actually means. |

**`priority_max`** bounds the worst case unconditionally. Even with
`BodyWins` and a hostile client, the value the queue sees is at most
`priority_max`. Treat it as the same kind of safety net as
`MAX_INT_TOKENS` — a sanity bound, not a security boundary.

**`inference_objective_policy`** controls the *trust boundary*. The
three modes correspond to three real deployment archetypes:

- `BodyWins` — single-tenant dev, the request body is fully trusted.
  Priority comes from whoever wrote the SDK call, headers are a
  fallback. Today's behavior, kept as default for backward
  compatibility.
- `HeaderWins` — production multi-tenant with a trusted gateway.
  The gateway resolves `x-gateway-inference-objective` against the
  CRD-backed map; even if a client also sets `nvext.priority`, the
  header value wins. Body remains usable as a fallback for
  unauthenticated paths.
- `HeaderOnly` — hardened multi-tenant. The frontend strips
  `nvext.agent_hints.priority` on inbound requests; only the header
  path can set priority, and the header is only meaningful if the
  named objective resolves through the RBAC-protected CRD. The body
  path is a closed door, not just an outranked one.

**RBAC on the CRD** is the actual authorization surface. The CRD path
turns priority assignment into a Kubernetes object with all the usual
properties: a named API group
(`inferenceobjectives.inference.networking.x-k8s.io`), namespace
scoping, RBAC verbs (`create` / `update` / `delete`), audit logs, and
admission webhooks if the cluster runs them. A platform team that
wants to control priority simply restricts who can `create` and
`update` `InferenceObjective` resources. Existing cluster identity,
audit, and policy tooling apply automatically — no Dynamo-specific
authorization layer needed.

### 9.3 Recommended deployment posture

| Environment | `inference_objective_policy` | `priority_max` | RBAC scope |
|-------------|------------------------------|----------------|------------|
| Local dev / single-user | `BodyWins` (default) | `1000` (default) | n/a |
| Shared internal, trusted clients | `HeaderWins` | `100` | `InferenceObjective` editable by platform team only |
| Multi-tenant / public-facing | `HeaderOnly` | `100` | `InferenceObjective` editable by SRE only; gateway terminates auth and emits the header |

Operators are expected to set the two config values explicitly in any
non-dev deployment. We deliberately keep `BodyWins` as the default to
preserve today's behavior for users not opted into priority — but the
operator docs must call out that this is the *unauthenticated* mode
and is unsuitable for any multi-tenant deployment.

### 9.4 What this section does *not* solve

A few things remain out of scope for this DEP:

- **Per-tenant priority quota.** Even with `HeaderOnly` and an
  RBAC-protected CRD, every tenant who has access to objective `gold`
  can use it at full priority. Per-tenant rate limiting at priority
  `gold` lives at the gateway / external policy layer. This DEP only
  ensures Dynamo enforces what the gateway authorized.
- **Non-Kubernetes environments.** The CRD path requires a Kubernetes
  cluster. Bare-metal Dynamo deployments fall back to the body / flag
  path with `priority_max` as the only ceiling. We accept this
  trade-off: production multi-tenant Dynamo is overwhelmingly
  Kubernetes-hosted.
- **Cryptographic verification of objective claims.** We trust the
  gateway to terminate auth and emit a sanitized
  `x-gateway-inference-objective` header. If the gateway is
  compromised, all bets are off — but this is the standard
  perimeter-trust model for ingress controllers and is not a problem
  this DEP can or should solve.

## 10. Operator-side InferencePool generation

This section specifies the Go-side operator change required by REQ 9.
It lives in `deploy/operator/` and is independent of the Rust
frontend changes; the two can be reviewed and merged in parallel,
though Stage 1 is only end-to-end usable once both have landed.

### 10.1 DGD CRD field

Add `PriorityConfig` to the `DynamoGraphDeploymentSpec` in
`deploy/operator/api/v1beta1/dynamographdeployment_types.go`:

```go
// PriorityConfig controls the priority feature for this deployment.
// When GeneratePool is true, the operator emits a stub InferencePool
// resource keyed to this DGD so user-authored InferenceObjective
// resources have a valid poolRef target. The pool is operator-managed
// and tied to the DGD's lifecycle.
type PriorityConfig struct {
    // GeneratePool, when true, causes the operator to emit a stub
    // InferencePool. Defaults to false.
    // +optional
    GeneratePool bool `json:"generatePool,omitempty"`

    // PoolName overrides the generated pool's name. Defaults to
    // "<dgd-name>-pool" if unset. Must be a valid DNS-1123 label.
    // +optional
    PoolName string `json:"poolName,omitempty"`
}

type DynamoGraphDeploymentSpec struct {
    // ... existing fields ...

    // PriorityConfig holds priority-related operator behavior.
    // +optional
    PriorityConfig *PriorityConfig `json:"priorityConfig,omitempty"`
}
```

The new field is fully optional and additive — DGDs that don't set
it behave exactly as today.

### 10.2 Reconciliation behavior

In `deploy/operator/internal/controller/dynamographdeployment_controller.go`,
after the existing EPP reconciliation, add a step that decides whether
to emit a standalone stub pool:

```go
// After EPP reconciliation, decide if we additionally need a stub pool.
switch {
case eppEnabled(dgd):
    // EPP path already created the pool with EndpointPickerRef pointing
    // at the EPP service. Reuse it — no extra pool.
case dgd.Spec.PriorityConfig != nil && dgd.Spec.PriorityConfig.GeneratePool:
    pool, err := epp.GenerateStubInferencePool(dgd)
    if err != nil { return ctrl.Result{}, err }
    if err := r.applyResource(ctx, pool); err != nil { return ctrl.Result{}, err }
default:
    // Neither EPP nor priority-pool requested. No-op (today's behavior).
}
```

A new helper `GenerateStubInferencePool` lives next to the existing
`GenerateInferencePool` in
`deploy/operator/internal/dynamo/epp/inference_pool.go`:

```go
// GenerateStubInferencePool creates an InferencePool that satisfies
// the upstream schema without requiring an EPP runtime. Used when
// priorityConfig.generatePool is true and EPP is not enabled.
//
// The EndpointPickerRef field is required by the upstream schema, so
// we point it at the frontend service as a self-referential
// placeholder. Nothing routes through this ref in priority-only mode;
// it exists only to satisfy validation.
func GenerateStubInferencePool(dgd *v1beta1.DynamoGraphDeployment) (*gaiev1.InferencePool, error) {
    poolName := stubPoolName(dgd)
    selectorLabels := map[gaiev1.LabelKey]gaiev1.LabelValue{
        consts.KubeLabelDynamoComponentClass: consts.ComponentClassWorker,
        consts.KubeLabelDynamoNamespace:      gaiev1.LabelValue(dgd.Name),
    }

    pool := &gaiev1.InferencePool{
        ObjectMeta: metav1.ObjectMeta{
            Name:      poolName,
            Namespace: dgd.Namespace,
            Labels: map[string]string{
                consts.KubeLabelDynamoGraphDeploymentName: dgd.Name,
                "app.kubernetes.io/managed-by":            "dynamo-operator",
                "nvidia.com/dynamo-priority-stub-pool":    "true",
            },
        },
        Spec: gaiev1.InferencePoolSpec{
            TargetPorts: []gaiev1.Port{{Number: consts.DynamoServicePort}},
            Selector:    gaiev1.LabelSelector{MatchLabels: selectorLabels},
            EndpointPickerRef: gaiev1.EndpointPickerRef{
                Kind: "Service",
                Name: gaiev1.ObjectName(frontendServiceName(dgd)),
                Port: &gaiev1.Port{Number: consts.DynamoServicePort},
            },
        },
    }
    return pool, nil
}

func stubPoolName(dgd *v1beta1.DynamoGraphDeployment) string {
    if dgd.Spec.PriorityConfig != nil && dgd.Spec.PriorityConfig.PoolName != "" {
        return dgd.Spec.PriorityConfig.PoolName
    }
    return fmt.Sprintf("%s-pool", dgd.Name)
}
```

### 10.3 Lifecycle

The operator **MUST** set an `ownerReference` on the generated pool
pointing at the DGD with `controller: true`. This ensures the pool is
garbage-collected when the DGD is deleted and prevents two DGDs from
fighting over the same pool name.

Pool name conflict handling:

- If `priorityConfig.poolName` is set explicitly and a pool with that
  name already exists *not owned* by this DGD, the operator **MUST**
  fail reconciliation and surface a clear status condition
  (`PriorityPoolConflict`). It must not adopt or overwrite a pool it
  doesn't own.
- If the pool already exists *owned* by this DGD, the operator
  reconciles the spec to the desired state (idempotent update).

### 10.4 Webhook validation

Add a webhook check in
`deploy/operator/internal/webhook/validation/dynamographdeployment.go`:

- Reject DGDs that set both `eppConfig` and
  `priorityConfig.generatePool: true` *with* a custom
  `priorityConfig.poolName` that differs from the EPP-mode pool name.
  In EPP mode the EPP pool is reused; conflicting custom names
  indicate operator misconfiguration.
- Allow `priorityConfig.generatePool: true` with no `eppConfig` set —
  this is the priority-only mode.
- Allow `priorityConfig.generatePool: false` (or absent) — today's
  behavior, no pool generated.

### 10.5 Why not always emit the stub

We could make `priorityConfig.generatePool` default to `true` (or
unconditionally emit a stub for every DGD). We don't, for two reasons:

- **Backward compatibility.** Existing DGDs deployed without the
  `InferencePool` CRD installed in the cluster would start failing to
  reconcile. Opt-in keeps the change additive.
- **Cluster-wide CRD assumption.** Emitting an `InferencePool`
  requires the `inference.networking.k8s.io` CRD to be installed
  cluster-wide. Not every Dynamo cluster has this; opt-in defers the
  cluster-prep requirement to deployments that actually want
  named-objective priority.

### 10.6 Helm chart implications

The `deploy/helm/` chart exposes `priorityConfig` on the DGD values
schema. The chart README documents the prerequisite: install the
`inference.networking.k8s.io` CRD bundle (e.g. via the GAIE Helm
chart) before deploying a DGD with `priorityConfig.generatePool: true`.

## Rejection Metrics

When the system rejects a request, operators need to know **why** and
**who got hurt**.  Without dedicated metrics, two very different
scenarios look the same in a dashboard: "requests are failing."

**Scenario A — utilization overload (Tier 1):** all workers are maxed
out, and the `RejectionLayer` is rejecting every request below the
admission floor (priority-exempt requests still pass). This is a
scaling problem — you need more GPUs.

**Scenario B — priority shedding (Tier 2):** workers have headroom but
the scheduler queue is concentrated on a cache-miss tier, so requests
below the admission floor are shed at queue entry while exempt
requests continue to be admitted. The system is working as designed —
priority shedding is protecting high-value traffic. This is **not** a
scaling problem.

**Scenario C — cap-only shedding (Tier 2, priority feature off or
overrun):** the cache-miss-tiered cap is being hit but no priority
admission floor is configured (or every priority class has filled its
tier). This points at a misconfigured `router_queue_by_incoming_missing_isl`
table, not a scaling problem.

### Proposed counter

A Prometheus counter incremented on every rejection, with `tier` and
`reason` labels:

```
dynamo_frontend_requests_rejected_total{tier="hard_gate",  reason="utilization_overload"}        12
dynamo_frontend_requests_rejected_total{tier="queue_shed", reason="priority_shed"}              847
dynamo_frontend_requests_rejected_total{tier="queue_shed", reason="max_queued_isl_tokens_exceeded"} 23
```

The three reasons map directly onto the code paths:

- `utilization_overload` — Tier 1 hard gate (`Client::instance_ids_free()` empty), 503.
- `priority_shed` — Tier 2 with priority feature enabled and request below the admission floor (new `RouterBackpressureReason::PriorityShed`), 429.
- `max_queued_isl_tokens_exceeded` — Tier 2 cache-miss-tiered cap reached (existing `RouterBackpressureReason::MaxQueuedIslTokensExceeded`), 429. Fires when the priority feature is disabled, or when a cap is set so tightly that even the highest tier fills up.

If you only see `priority_shed`, the system is healthy and doing its
job. If `utilization_overload` is climbing, you need to scale. If
`max_queued_isl_tokens_exceeded` is climbing, your cache-miss tiers
are too tight for the offered load.

With optional additional labels for deeper analysis:

```
dynamo_frontend_requests_rejected_total{tier="queue_shed", reason="priority_shed",                  objective_name="batch-jobs",   priority="-10"}     800
dynamo_frontend_requests_rejected_total{tier="queue_shed", reason="priority_shed",                  objective_name="standard-api", priority="0"}       47
dynamo_frontend_requests_rejected_total{tier="hard_gate",  reason="utilization_overload",           objective_name="",             priority=""}        12
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

# Forward Compatibility

This DEP picks up only `spec.priority` from `InferenceObjective`. A workload class is realistically more than one integer — production deployments will eventually want TTFT/TPOT SLOs, fairness/quota policies, class-bound LoRA adapters, queueing/eviction policies, and chargeback labels attached to the same named class.

For now those concerns are **delegated upstream** of Dynamo and, in the case of SLAs, **fragmented across three existing config surfaces**:

- **Latency SLOs** are *not* on the GlobalPlanner — it's a budget-and-policy execution layer that arbitrates GPU totals, not SLAs. Per-pool SLA targets exist in the single-endpoint multi-pool GlobalPlanner topology, but they're split across three config surfaces today: DGDR `spec.sla.{ttft,itl}` (drives profiling and worker-shape selection), GlobalRouter JSON `prefill_pool_selection_strategy.ttft_*` / `decode_pool_selection_strategy.itl_*` (drives runtime routing), and each pool's local Planner config `ttft` / `itl` with `optimization_target=sla` (drives autoscaling). For single-DGD deployments the SLO is set deployment-wide via `PlannerConfig.optimization_target=sla` and applies to the whole DGD, not per-class.
- **Fairness/quota and tenant rate limiting** live at the API gateway in front of Dynamo (Kong, Envoy, the GAIE gateway) — Dynamo treats inbound requests as already authenticated and already rate-limited.

The reflector and `InferenceObjectiveMap` introduced here are deliberately structured so future fields slot in by extending the struct, not by reshaping the integration. When upstream adds `targetTTFT`, `fairShare`, `allowedLoraAdapters`, etc., they become additional columns in the same map — and the SLA / quota / LoRA-authorization concerns currently delegated upstream and fragmented across DGDR / GlobalRouter / Planner can collapse into a single `InferenceObjective` instance that all three consume, without changing the request hot path or the trust model.

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

1. **Non-K8s deployments.** For bare-metal / Docker Compose setups,
   should there be a static config file alternative to the K8s watcher
   (e.g. a JSON/YAML map of objective name to priority loaded at
   startup)? The body-level `agent_hints.priority` path always works
   without K8s, so this is purely about supporting the named-objective
   ergonomic in non-K8s environments.

2. **Multiple frontends.** Each frontend runs its own kube-rs reflector
   against the same `InferenceObjective` resources, so the priority map
   is eventually consistent across replicas. Typical convergence is
   sub-second, bounded by the reflector resync interval. Is that
   sufficient, or does an SLA-sensitive deployment need a shared view
   (e.g. via a router-side fan-out)? Recommend: ship eventually
   consistent and expose the staleness metric from REQ 1 so operators
   can decide.

3. **Failure mode for unresolved objective names.** Today the proposal
   says "log at debug and pass through unchanged" if the header refers
   to an unknown objective. That fails *open* — the request runs at
   default priority. An alternative is fail *closed* — reject with 400
   ("unknown objective"). The behavior is operator-tunable, but which
   is the safer default?

4. **`priority_admission_floor` default value.** The DEP picks `1` as
   the default floor (any positive priority is exempt). The alternative
   is `i32::MAX` (priority feature has no admission effect unless an
   operator opts in). Pick once based on whether we want zero-config
   priority differentiation or zero-config behavior preservation.

5. **Engine-side priority sign convention.** The existing per-backend
   normalization (`-int(priority)` for vLLM, the `schedule_low_priority_values_first`
   flag for sglang, `[0,1]` float for trtllm — see
   `components/src/dynamo/{vllm,sglang,trtllm}/...`) is a known
   compatibility maze. The DEP relies on that normalization being
   correct already; this is a flag for the implementation PR to add
   integration tests that verify each backend honors the resolved
   priority correctly under load, not a design question to resolve
   here.

6. **`inference_objective_policy` default.** The DEP picks `BodyWins`
   as the default to preserve today's behavior for users who do not
   opt into priority. The alternative is to default to `HeaderWins`
   (or even `HeaderOnly`) on the grounds that priority is a
   security-relevant feature and should ship with secure defaults.
   The argument for `BodyWins` is "no surprises for existing users";
   the argument against is that `BodyWins` is the *unauthenticated*
   mode and the documentation has to be clear that it is unsafe in
   any multi-tenant deployment. Recommend: keep `BodyWins` default,
   require operators of multi-tenant deployments to flip it
   explicitly, and add a startup warning if `BodyWins` is set in a
   namespace that has any `InferenceObjective` resources defined
   (signaling the operator probably meant to opt into the CRD
   path). See §9.

7. **`priority_max` default.** This DEP suggests `1000` as a sensible
   ceiling — generous enough that legitimate use of named objectives
   never hits it, tight enough that a hostile client setting
   `i32::MAX` cannot starve the queue any worse than priority `1000`
   would. Operators with a defined priority range (e.g. `[0, 10]`)
   should tighten this further. Open question: should the default be
   tighter (e.g. `100`)?
