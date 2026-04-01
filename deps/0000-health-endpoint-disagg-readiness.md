# Disaggregated Topology Readiness

**Status**: Draft

**Authors**: @jihao

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

The frontend has no awareness of disaggregated serving topology. `GET /v1/models` lists any model with at least one worker, regardless of whether its topology is complete. The `check_ready()` function gating request handlers is a no-op. This causes decode-only workers to receive aggregated requests and crash when their required counterparts (prefill, encode) are not yet available.

This DEP proposes that workers declare their role and dependencies at registration time (derived from existing CLI flags), enabling the frontend to perform per-model, per-namespace topology completeness checks. Readiness is surfaced through three mechanisms: `/v1/models` filtering, per-model request gating (HTTP 503), and a new `GET /v1/models/{model}/readiness` detail endpoint. The `/health` endpoint is **not in scope** — it reflects frontend process readiness, not model-level serving readiness.

# Motivation

In a disaggregated deployment, decode workers can start and register before prefill workers. Because the frontend cannot distinguish decode-only workers from aggregated workers (both register as `ModelType.Chat | Completions`), it lists the model in `/v1/models` and begins routing requests as soon as any worker is discovered. Decode-only workers (TRT-LLM, SGLang) then crash with:

```
ValueError: Disaggregated params are required for decode mode
```

This was first reported in the [swdl-dynamo-dev Slack thread (Feb 25, 2025)](https://nvidia.slack.com/archives/C06850J381Y/p1740513081785499) and affects TRT-LLM and SGLang backends. vLLM is less affected because its decode workers could fall back to aggregated mode, though this fallback behavior has been identified as confusing and is now disabled by default (`--decode-fallback=false`).

The problem is compounded by two additional gaps:

1. **No request gating**: The `check_ready()` function called by every request handler (`lib/llm/src/http/service/openai.rs:1743`) is a no-op — the readiness check is commented out. Even if `/health` reported "not ready," the frontend would still accept and route requests.
2. **No topology awareness**: The frontend discovers workers reactively and has no concept of what constitutes a "complete" serving topology for a given model.

## Goals

- `GET /v1/models` **MUST** only list models whose topology is complete (at least one namespace has all required roles present).
- The frontend **MUST** reject requests (HTTP 503) to models whose worker topology is incomplete in all namespaces.
- A new `GET /v1/models/{model}/readiness` endpoint **MUST** provide per-namespace topology detail for debugging and monitoring.
- The `/health` endpoint **MUST NOT** be modified — it reflects frontend process readiness only.
- The solution **MUST** support current pipeline topologies: aggregated, P/D (prefill/decode), and E/P/D (encode/prefill/decode).
- The solution **SHOULD** be extensible to future pipeline topologies without changes to the readiness logic.
- Readiness checks **MUST** be namespace-scoped to align with the WorkerSet architecture, which only pairs prefill and decode workers within the same namespace.

### Non Goals

- Expected replica counts (e.g., "we expected 4 decode workers and only 2 are up"). This requires the orchestration layer (K8s operator / DGD spec) to pass expected counts, which is out of scope.
- Changes to the K8s operator or DynamoGraphDeployment spec.
- Changes to the per-worker system status server (`DYN_SYSTEM_PORT`). That endpoint serves a different purpose (individual worker health).
- Changes to the `/health` endpoint. It reflects frontend process readiness and should not carry model-level topology information.

## Requirements

### REQ 1 `/v1/models` Filtering

`GET /v1/models` **MUST** only list models that have at least one namespace with a complete topology. A model is ready only when all worker roles required by its topology are present with at least one worker each, within the same namespace.

### REQ 2 Request Gating

The frontend **MUST** return HTTP 503 for inference requests to models whose topology is incomplete in all namespaces, rather than routing to workers that will fail.

### REQ 3 Readiness Detail Endpoint

A new `GET /v1/models/{model}/readiness` endpoint **MUST** provide per-namespace topology detail for debugging and monitoring.

### REQ 4 Topology Extensibility

The readiness mechanism **SHOULD** support new pipeline topologies (e.g., E/P/D) without requiring changes to the core readiness logic.

### REQ 5 Namespace-Scoped Readiness

Readiness checks **MUST** be scoped to namespaces to align with the WorkerSet architecture, which only pairs prefill and decode workers within the same namespace via `oneshot`-based prefill router activation.

### REQ 6 No New Worker Flags

Workers **SHOULD** derive their role and dependencies from existing configuration flags (`--disaggregation-mode`, `--route-to-encoder`, `--multimodal-encode-worker`, etc.) without requiring new CLI arguments.

### REQ 7 `/health` Unchanged

The `/health` endpoint **MUST NOT** be modified. It reflects frontend process readiness only.

# Proposal

## Background: Current Architecture

### `check_ready()` is a no-op

Every request handler calls `check_ready()` before processing, but the function always returns `Ok(())`:

```rust
fn check_ready(_state: &Arc<service_v2::State>) -> Result<(), ErrorResponse> {
    // if state.service_observer.stage() != ServiceStage::Ready {
    //     return Err(ErrorMessage::service_unavailable());
    // }
    Ok(())
}
```

### `/v1/models` has no topology awareness

`GET /v1/models` lists any model that passes `Model::is_displayable()`, which only checks that at least one WorkerSet has a serving engine and a non-zero worker count. It does not check whether the model's disaggregated topology is complete.

### WorkerSet architecture and namespace scoping

The `ModelManager` organizes workers hierarchically: `ModelManager → Model → WorkerSet(s)`. Each WorkerSet represents a group of workers deployed from the same configuration, identified by their shared namespace. In disaggregated deployments, prefill and decode workers in the same namespace form separate WorkerSets (keyed as `"ns"` and `"ns:prefill"` via `worker_set_key()`).

Critically, the prefill router coordination uses a `oneshot` channel keyed by `"model_name:namespace"` — only prefill and decode workers **within the same namespace** can be paired. A decode WorkerSet in `ns-old` cannot route to a prefill WorkerSet in `ns-new`. This means readiness must be checked per-namespace, not per-model.

### Design constraints

These were confirmed through codebase investigation and simplify the design:

1. **One model = one topology.** Mixed agg+disagg for the same model is not supported. The planner (`AggPlanner` vs `DisaggPlanner`), router, and checksum validation all assume a single mode per model.
2. **All workers of the same role for a model share the same configuration.** Checksum validation rejects mismatches.
3. **Workers know their own dependencies at startup** from existing CLI flags (`--disaggregation-mode`, `--route-to-encoder`, etc.).
4. **Encode workers are optional at deployment time, but required at runtime if configured.** A model with prefill worker configured with `--route-to-encoder` will not be considered ready if no encode worker is present.
5. **The frontend has no upfront knowledge** of how many workers or what types to expect.
6. **Readiness is namespace-scoped.** The WorkerSet architecture only pairs prefill and decode within the same namespace (via `oneshot`-based prefill router activation). Readiness must reflect actual routability, not just role presence across the model.

## Overview: Worker-Declared Dependencies

Each worker declares its **role** and **dependencies** at registration time, derived from existing CLI flags. The frontend builds a per-model dependency graph and checks completeness. No new frontend flags or global configuration needed.

### Reuse of `DisaggregationMode`

The worker's role maps directly to the existing `DisaggregationMode` enum:

- **Python** (`components/src/dynamo/common/constants.py`): currently defines `AGGREGATED`, `PREFILL`, `DECODE`
- **TRT-LLM** (`components/src/dynamo/trtllm/constants.py`): extends with `ENCODE`

We extend the common `DisaggregationMode` to include `ENCODE` and use it as the `role` type. The `needs` field is a new `List[DisaggregationMode]` added to the registration payload (via `ModelDeploymentCard`).

On the Rust side, we add corresponding `Decode` and `Encode` variants to `ModelType` bitflags in `lib/llm/src/model_type.rs`, and add a `needs: Vec<ModelType>` field to `ModelDeploymentCard`.

### Worker registration

Workers derive `{role, needs}` from existing CLI flags at startup — no new flags required:


| Worker configuration                                          | `role` (`DisaggregationMode`) | `needs`                                      |
| ------------------------------------------------------------- | ----------------------------- | -------------------------------------------- |
| `--disaggregation-mode decode`                                | `DECODE`                      | `[PREFILL]`                                  |
| `--disaggregation-mode prefill`                               | `PREFILL`                     | `[DECODE]`                                   |
| `--disaggregation-mode prefill --route-to-encoder`            | `PREFILL`                     | `[DECODE, ENCODE]`                           |
| `--disaggregation-mode encode` / `--multimodal-encode-worker` | `ENCODE`                      | `[PREFILL, DECODE]`                          |
| No disagg flags (aggregated)                                  | `AGGREGATED`                  | `[]` (self-sufficient)                       |


### Readiness check

Readiness is checked **per-namespace**, not per-model, because the WorkerSet architecture only pairs prefill and decode workers within the same namespace. A model-wide check would report "ready" when a decode exists in `ns-old` and a prefill exists in `ns-new`, even though they cannot actually route to each other.

For each model, the frontend:

1. Groups WorkerSets by base namespace (strip the `:prefill` suffix used in `worker_set_key`).
2. For each namespace:
   a. Collects the union of all `needs` across registered workers, plus the roles themselves → **required set**.
   b. Collects the set of roles with at least one registered worker → **present set**.
   c. **Namespace complete** = required set is a subset of the present set.
3. **Model ready** = at least one namespace is complete.
4. `AGGREGATED` workers (`needs: []`) are always self-sufficient — their namespace is trivially complete.

### Example: P/D startup (same namespace)

1. Decode worker registers in `ns-abc`: `{role: DECODE, needs: [PREFILL]}`
   - `ns-abc` required: `{PREFILL, DECODE}`, present: `{DECODE}` → **not ready** (missing PREFILL)
2. Prefill worker registers in `ns-abc`: `{role: PREFILL, needs: [DECODE]}`
   - `ns-abc` required: `{PREFILL, DECODE}`, present: `{DECODE, PREFILL}` → **ready**

### Example: P/D cross-namespace (not routable)

1. Decode worker registers in `ns-old`: `{role: DECODE, needs: [PREFILL]}`
2. Prefill worker registers in `ns-new`: `{role: PREFILL, needs: [DECODE]}`
   - `ns-old` required: `{PREFILL, DECODE}`, present: `{DECODE}` → **not ready**
   - `ns-new` required: `{PREFILL, DECODE}`, present: `{PREFILL}` → **not ready**
   - Model not ready — no namespace is complete, even though both roles exist across the model.

### Example: E/P/D startup

1. Decode registers in `ns-abc`: `{role: DECODE, needs: [PREFILL]}` → not ready
2. Prefill registers in `ns-abc`: `{role: PREFILL, needs: [DECODE, ENCODE]}` → not ready (missing ENCODE)
3. Encode registers in `ns-abc`: `{role: ENCODE, needs: [PREFILL, DECODE]}` → **ready**

### Example: Aggregated

1. Agg worker registers in `ns-abc`: `{role: AGGREGATED, needs: []}`
   - `ns-abc` required: `{AGGREGATED}`, present: `{AGGREGATED}` → **ready** (trivially complete)

### Mechanism 1: `/v1/models` filtering

`Model::is_displayable()` is extended with a topology check. A model only appears in `GET /v1/models` when at least one namespace has a complete topology. Clients polling `/v1/models` to discover available models will not see a model until it is actually routable.

```json
GET /v1/models
{
  "object": "list",
  "data": [
    {
      "id": "llama-3.1-70b",
      "object": "model",
      "created": 1711929600,
      "owned_by": "nvidia"
    }
  ]
}
```

A model with only decode workers (missing prefill) will not appear in this list.

### Mechanism 2: Per-model request gating

The existing `check_ready()` function in `lib/llm/src/http/service/openai.rs` (currently a no-op) is wired to perform per-model, per-namespace topology checks. When a request arrives for a model whose topology is incomplete (no namespace has all required roles), the frontend returns HTTP 503 immediately:

```json
HTTP 503
{
  "error": {
    "message": "Model 'llama-3.1-70b' is not ready: missing roles in all namespaces",
    "type": "service_unavailable"
  }
}
```

This gates all request handlers: chat completions, completions, embeddings, images, and responses.

```rust
fn check_model_ready(state: &State, model: &str) -> Result<(), ErrorResponse> {
    if let Some(model_obj) = state.manager().get_model(model) {
        if !model_obj.has_complete_namespace() {
            let summary = model_obj.readiness_summary();
            return Err(service_unavailable_with_reason(
                format!("Model '{}' is not ready: {}", model, summary)
            ));
        }
    }
    Ok(())
}
```

### Mechanism 3: Readiness detail endpoint `GET /v1/models/{model}/readiness`

A new endpoint exposes per-namespace topology detail for debugging and monitoring. This is where the full dependency graph is visible, without overloading `/health` or `/v1/models`.

```json
GET /v1/models/llama-3.1-70b/readiness
{
  "model": "llama-3.1-70b",
  "ready": false,
  "reason": "no namespace has complete topology",
  "namespaces": {
    "ns-abc12345": {
      "ready": false,
      "reason": "missing roles: prefill",
      "roles": {
        "decode": {"workers": 2, "needs": ["prefill"]},
        "prefill": {"workers": 0, "needs": ["decode"]}
      },
      "required": ["decode", "prefill"],
      "present": ["decode"]
    },
    "ns-def67890": {
      "ready": true,
      "roles": {
        "decode": {"workers": 2, "needs": ["prefill"]},
        "prefill": {"workers": 1, "needs": ["decode"]}
      },
      "required": ["decode", "prefill"],
      "present": ["decode", "prefill"]
    }
  }
}
```

For models that don't exist, returns HTTP 404. For aggregated models (`needs: []`), returns a trivially-ready response.

# Implementation Details

## Type changes

### Python: Extend `DisaggregationMode`

**File: `components/src/dynamo/common/constants.py`**

Add `ENCODE` to the common `DisaggregationMode` enum (currently only in TRT-LLM's `constants.py`):

```python
class DisaggregationMode(str, Enum):
    AGGREGATED = "agg"
    PREFILL = "prefill"
    DECODE = "decode"
    ENCODE = "encode"   # new
```

### Rust: Extend `ModelType` bitflags

**File: `lib/llm/src/model_type.rs`**

Add `Decode` and `Encode` variants:

```rust
pub struct ModelType: u8 {
    const Chat = 1 << 0;
    const Completions = 1 << 1;
    const Embedding = 1 << 2;
    const TensorBased = 1 << 3;
    const Prefill = 1 << 4;
    const Images = 1 << 5;
    const Audios = 1 << 6;
    const Videos = 1 << 7;
    // Note: u8 is full. May need to expand to u16 or use a separate field.
    // Alternatively, Decode and Encode can be tracked via the new `role`/`needs`
    // fields on ModelDeploymentCard without adding ModelType variants.
}
```

> **Note:** `ModelType` is a `u8` bitflag with all 8 bits used. Adding `Decode` and `Encode` requires either expanding to `u16` or tracking role/needs as separate fields on `ModelDeploymentCard` rather than as `ModelType` variants. The latter is simpler and avoids a breaking change to the bitflag width.

### Rust: Extend `ModelDeploymentCard`

**File: `lib/llm/src/model_card.rs`**

Add `role` and `needs` fields:

```rust
pub struct ModelDeploymentCard {
    // ... existing fields ...
    /// Worker's role: "decode", "prefill", "encode", "agg"
    /// Maps to DisaggregationMode on the Python side.
    pub role: Option<String>,
    /// Roles this worker needs for the model to be servable.
    pub needs: Vec<String>,
}
```

- `role` is `Option<String>` for backward compat — workers without it default to aggregated behavior.
- `role`/`needs` are NOT part of the model checksum (deployment topology, not model identity).

## Worker registration changes (Python)


| File                                                       | Change                                                                                                        |
| ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `components/src/dynamo/common/constants.py`                | Add `ENCODE` to `DisaggregationMode`                                                                          |
| `components/src/dynamo/vllm/args.py`                       | Derive `role` from `disaggregation_mode`, derive `needs` from `disaggregation_mode` + `route_to_encoder`      |
| `components/src/dynamo/trtllm/args.py`                     | Same (already has `DisaggregationMode.ENCODE`)                                                                |
| `components/src/dynamo/sglang/args.py`                     | Same (map `--multimodal-encode-worker` → `role=ENCODE`)                                                       |
| `components/src/dynamo/vllm/main.py` / `worker_factory.py` | Pass `role` and `needs` to `register_model()`                                                                 |
| `components/src/dynamo/trtllm/workers/llm_worker.py`       | Same                                                                                                          |
| `components/src/dynamo/sglang/__main__.py` / init files    | Same                                                                                                          |


## Rust registration & discovery changes


| File                                       | Change                                                                                               |
| ------------------------------------------ | ---------------------------------------------------------------------------------------------------- |
| `lib/llm/src/model_card.rs`                | Add `role: Option<String>` and `needs: Vec<String>` to `ModelDeploymentCard`                         |
| `lib/llm/src/model_type.rs`                | Evaluate adding `Decode`/`Encode` variants or tracking via card fields (see note above on u8 limit)  |
| `lib/llm/src/discovery/watcher.rs`         | Extract `role`/`needs` from card, call `model.register_role()` on add/remove                         |
| `lib/llm/src/discovery/model.rs`           | Add `RoleInfo` struct, `roles: DashMap`, `register_role()`, `is_namespace_complete()`, `has_complete_namespace()`, `first_complete_namespace()`, `missing_roles(namespace)`. Extend `is_displayable()` to require topology completeness. |
| `lib/llm/src/discovery/model_manager.rs`   | Add `NamespaceReadiness`, `ModelReadiness` structs, `readiness_summary()` using per-namespace graph completeness |
| `lib/bindings/python/src/dynamo/_core.pyi` | Update `register_model()` signature to accept `role` and `needs`                                     |


## Frontend request gating & readiness changes


| File                                       | Change                                                                  |
| ------------------------------------------ | ----------------------------------------------------------------------- |
| `lib/llm/src/http/service/openai.rs`       | Wire `check_ready()` to per-model topology check via `check_model_ready()`; extend `list_models_openai` to filter by `has_complete_namespace()`; add `GET /v1/models/{model}/readiness` handler |
| `lib/llm/src/http/service/service_v2.rs`   | Register the new `/v1/models/{model}/readiness` route                   |
| `lib/llm/src/http/service/openapi_docs.rs` | Add docs for `/v1/models/{model}/readiness` endpoint                    |


## Deferred to Implementation

- Whether `role`/`needs` should use string values or a shared Rust enum. Strings are simpler for cross-language serialization; enums are safer for type checking.
- Whether to expand `ModelType` to `u16` for `Decode`/`Encode` variants or keep role/needs as separate `ModelDeploymentCard` fields.
- `role`/`needs` are NOT part of the model checksum (deployment topology, not model identity).
- Exact placement of `check_model_ready()` in the request handler chain (before or after model name extraction).
- Namespace visibility in readiness endpoint — exposing internal namespace IDs (e.g., `ns-abc12345`) may be noisy for operators. Consider whether to show them, use aliases, or only surface them at a verbose detail level.
- `/v1/models` backward compatibility — clients that expect a model to appear as soon as any worker registers will now see it delayed until topology is complete. This is the desired behavior but may surprise existing deployments.
- `--decode-fallback` interaction — a decode worker with fallback enabled could be considered self-sufficient (no prefill needed). Whether this affects readiness semantics needs discussion.

# Alternate Solutions

## Alt 1: `--serving-mode` Frontend Flag

Add a `--serving-mode` CLI option to the frontend (`agg` / `disagg` / `auto`) that tells the frontend what topology to expect.

**Pros:**

- Minimal change surface, quick to implement
- No changes to worker registration needed

**Cons:**

- Global flag, not per-model — prevents mixed-topology multi-model frontends
- Does not scale to E/P/D or future topologies (would need new enum values for each)
- `auto` mode cannot distinguish decode-only from aggregated workers (both register as `ModelType.Chat | Completions`)

**Reason Rejected:**

- Does not meet REQ 4 (topology extensibility) or REQ 1 (`/v1/models` filtering for mixed deployments). Worker-declared dependencies solve both without a global flag.

## Alt 2: Static Dependency Table in Frontend

Define a hardcoded mapping of worker types to their dependencies (e.g., Decode requires Prefill, Encode requires Prefill + Decode).

**Pros:**

- No changes to worker registration

**Cons:**

- Dependencies are not static — a prefill worker may or may not need an encode worker depending on `--route-to-encoder`. The same worker type has different dependencies based on deployment configuration.
- Requires frontend code changes for every new topology.

**Reason Rejected:**

- Dependencies are deployment-time decisions, not type-level properties. Workers already know their dependencies from their CLI flags.

## Alt 3: Explicit `Decode` ModelType Only

Add a `Decode` variant to `ModelType` bitflags so decode-only workers are distinguishable from aggregated workers.

**Pros:**

- Solves the decode-vs-agg ambiguity

**Cons:**

- Does not address E/P/D or future topologies
- Would need additional types (`Encode`, etc.) for each new pipeline stage
- Does not capture per-deployment dependencies (same type, different needs)

**Reason Rejected:**

- Partially solves the problem but does not generalize. Worker-declared dependencies subsumes this approach.

# Background

## References

- [Slack thread: Disagg prefill fallback causes confusing errors (Feb 25, 2025)](https://nvidia.slack.com/archives/C06850J381Y/p1740513081785499)
- `check_ready()` no-op: `lib/llm/src/http/service/openai.rs:1743`
- `/v1/models` handler: `lib/llm/src/http/service/openai.rs` (`list_models_openai`)
- Model manager: `lib/llm/src/discovery/model_manager.rs`
- Model: `lib/llm/src/discovery/model.rs`
- WorkerSet: `lib/llm/src/discovery/worker_set.rs`
- WorkerSet architecture PR: [#6054](https://github.com/ai-dynamo/dynamo/pull/6054) — hierarchical Model/WorkerSet for multi-namespace support
- Prefill router coordination (oneshot namespace pairing): `lib/llm/src/discovery/model_manager.rs:583-689`

## Terminology & Definitions


| Term                            | Definition                                                                                                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Aggregated serving**          | A single worker handles both prefill and decode phases for a request.                                                                        |
| **Disaggregated serving (P/D)** | Prefill and decode are handled by separate, specialized workers.                                                                             |
| **E/P/D**                       | Encode/Prefill/Decode — a three-stage disaggregated pipeline for multimodal models where vision encoding is offloaded to a dedicated worker. |
| **Topology**                    | The set of worker roles required to serve a model end-to-end (e.g., {prefill, decode} for P/D).                                              |
| **ModelManager**                | Rust component in the frontend that tracks registered models and their workers. Hierarchy: ModelManager → Model → WorkerSet(s).              |
| **WorkerSet**                   | A group of workers deployed from the same configuration, identified by a shared namespace. Each WorkerSet owns its own pipeline (engines, KV router, worker monitor). |
| **Namespace**                   | An identifier shared by workers of the same configuration. Prefill and decode workers are only paired within the same namespace.              |
| **ModelDeploymentCard**         | Metadata about a model instance registered by a worker, including model name, capabilities, and configuration.                               |


## Acronyms & Abbreviations

**DEP:** Dynamo Enhancement Proposal

**P/D:** Prefill/Decode (disaggregated serving)

**E/P/D:** Encode/Prefill/Decode (multimodal disaggregated serving)

**DGD:** DynamoGraphDeployment (Kubernetes custom resource)