# Frontend Health Endpoint: Disaggregated Serving Readiness

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

The frontend's `/health` endpoint (`localhost:8000/health`) always returns HTTP 200 with `"status": "healthy"` as long as any worker instance exists in service discovery. It has no awareness of disaggregated serving topology. This causes decode-only workers to receive aggregated requests and crash when their required counterparts (prefill, encode) are not yet available.

This DEP proposes that workers declare their role and dependencies at registration time (derived from existing CLI flags), enabling the frontend to perform per-model topology completeness checks. The frontend's `/health` endpoint and request handlers are updated to report readiness and reject requests when a model's topology is incomplete.

# Motivation

In a disaggregated deployment, decode workers can start and register before prefill workers. Because the frontend cannot distinguish decode-only workers from aggregated workers (both register as `ModelType.Chat | Completions`), it reports "healthy" and begins routing requests as soon as any worker is discovered. Decode-only workers (TRT-LLM, SGLang) then crash with:

```
ValueError: Disaggregated params are required for decode mode
```

This was first reported in the [swdl-dynamo-dev Slack thread (Feb 25, 2025)](https://nvidia.slack.com/archives/C06850J381Y/p1740513081785499) and affects TRT-LLM and SGLang backends. vLLM is less affected because its decode workers could fall back to aggregated mode, though this fallback behavior has been identified as confusing and is now disabled by default (`--decode-fallback=false`).

The problem is compounded by two additional gaps:

1. **No request gating**: The `check_ready()` function called by every request handler (`lib/llm/src/http/service/openai.rs:1743`) is a no-op — the readiness check is commented out. Even if `/health` reported "not ready," the frontend would still accept and route requests.
2. **No topology awareness**: The frontend discovers workers reactively and has no concept of what constitutes a "complete" serving topology for a given model.

## Goals

- The frontend `/health` endpoint **MUST** report per-model readiness based on whether the model's worker topology is complete.
- The frontend **MUST** reject requests (HTTP 503) to models whose worker topology is incomplete.
- The solution **MUST** support current pipeline topologies: aggregated, P/D (prefill/decode), and E/P/D (encode/prefill/decode).
- The solution **SHOULD** be extensible to future pipeline topologies without changes to the health check logic.
- The solution **SHOULD** provide actionable information in the health response (what roles are present, what is missing).

### Non Goals

- Expected replica counts (e.g., "we expected 4 decode workers and only 2 are up"). This requires the orchestration layer (K8s operator / DGD spec) to pass expected counts, which is out of scope.
- Changes to the K8s operator or DynamoGraphDeployment spec.
- Changes to the per-worker system status server (`DYN_SYSTEM_PORT`). That endpoint serves a different purpose (individual worker health).

## Requirements

### REQ 1 Per-Model Readiness

The `/health` endpoint **MUST** report readiness per model. A model is ready only when all worker roles required by its topology are present with at least one worker each.

### REQ 2 Request Gating

The frontend **MUST** return HTTP 503 for inference requests to models whose topology is incomplete, rather than routing to workers that will fail.

### REQ 3 Topology Extensibility

The readiness mechanism **SHOULD** support new pipeline topologies (e.g., E/P/D) without requiring changes to the core health check logic.

### REQ 4 Backward Compatibility

The `/health` response **SHOULD** retain the existing `endpoints` and `instances` fields for backward compatibility while adding new per-model readiness information.

### REQ 5 No New Worker Flags

Workers **SHOULD** derive their role and dependencies from existing configuration flags (`--disaggregation-mode`, `--route-to-encoder`, `--multimodal-encode-worker`, etc.) without requiring new CLI arguments.

# Proposal

## Background: Current Architecture

### Two `/health` endpoints


| Endpoint                 | Port                           | What it checks                                                                                                  |
| ------------------------ | ------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| **System status server** | `DYN_SYSTEM_PORT` (e.g., 8081) | Per-worker endpoint readiness (e.g., "is my `generate` endpoint up?"). Per-worker, not topology-aware.          |
| **Frontend HTTP server** | `DYN_HTTP_PORT` (e.g., 8000)   | Lists all discovered instances from service discovery. Always returns 200 "healthy" if frontend is initialized. |


Neither checks whether discovered workers form a complete serving topology.

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

### Design constraints

These were confirmed through codebase investigation and simplify the design:

1. **One model = one topology.** Mixed agg+disagg for the same model is not supported. The planner (`AggPlanner` vs `DisaggPlanner`), router, and checksum validation all assume a single mode per model.
2. **All workers of the same role for a model share the same configuration.** Checksum validation rejects mismatches.
3. **Workers know their own dependencies at startup** from existing CLI flags (`--disaggregation-mode`, `--route-to-encoder`, etc.).
4. **Encode workers are optional at deployment time, but required at runtime if configured.** A model with prefill worker configured with `--route-to-encoder` will not be considered healthy if no encode worker is present.
5. **The frontend has no upfront knowledge** of how many workers or what types to expect.

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

For each model, the frontend:

1. Collects the union of all `needs` across registered workers, plus the roles themselves → **required set**.
2. Collects the set of roles with at least one registered worker → **present set**.
3. **Ready** = required set is a subset of the present set.

### Example: P/D startup

1. Decode worker registers: `{role: DECODE, needs: [PREFILL]}`
   - Required: `{PREFILL, DECODE}`, Present: `{DECODE}` → **not ready** (missing PREFILL)
2. Prefill worker registers: `{role: PREFILL, needs: [DECODE]}`
   - Required: `{PREFILL, DECODE}`, Present: `{DECODE, PREFILL}` → **ready**

### Example: E/P/D startup

1. Decode registers: `{role: DECODE, needs: [PREFILL]}` → not ready
2. Prefill registers: `{role: PREFILL, needs: [DECODE, ENCODE]}` → not ready (missing ENCODE)
3. Encode registers: `{role: ENCODE, needs: [PREFILL, DECODE]}` → **ready**

### Example: Aggregated

1. Agg worker registers: `{role: AGGREGATED, needs: []}`
   - Required: `{AGGREGATED}`, Present: `{AGGREGATED}` → **ready**

### Top-level status


| Condition                   | Status      | HTTP Code |
| --------------------------- | ----------- | --------- |
| All models ready            | `ready`     | 200       |
| Some models ready, some not | `degraded`  | 200       |
| No models ready / no models | `not_ready` | 503       |


### Health response format

```json
{
  "status": "not_ready",
  "models": {
    "llama-3.1-70b": {
      "status": "not_ready",
      "reason": "missing roles: prefill",
      "roles": {
        "decode": {"workers": 2, "needs": ["prefill"]},
        "prefill": {"workers": 0, "needs": ["decode"]}
      },
      "required": ["decode", "prefill"],
      "present": ["decode"]
    }
  },
  "endpoints": ["dyn://dynamo.backend.generate", "..."],
  "instances": [{"component": "backend", "...": "..."}]
}
```

The `endpoints` and `instances` fields are retained for backward compatibility.

### Per-model request gating

Update `check_ready()` in `openai.rs` to check per-model topology completeness. Requests to a model with incomplete topology return HTTP 503 immediately instead of being routed to workers that will fail:

```rust
fn check_model_ready(state: &State, model: &str) -> Result<(), ErrorResponse> {
    if let Some(model_obj) = state.manager().get_model(model) {
        if !model_obj.is_topology_complete() {
            let missing = model_obj.missing_roles();
            return Err(service_unavailable_with_reason(
                format!("Model '{}' is not ready: missing roles: {}", model, missing.join(", "))
            ));
        }
    }
    Ok(())
}
```

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
| `lib/llm/src/discovery/model.rs`           | Add `RoleInfo` struct, `roles: DashMap`, `register_role()`, `is_topology_complete()`, `missing_roles()` |
| `lib/llm/src/discovery/model_manager.rs`   | Add `ModelReadiness` struct, `readiness_summary()` using per-model graph completeness                |
| `lib/bindings/python/src/dynamo/_core.pyi` | Update `register_model()` signature to accept `role` and `needs`                                     |


## Frontend health & request gating changes


| File                                       | Change                                                                  |
| ------------------------------------------ | ----------------------------------------------------------------------- |
| `lib/llm/src/http/service/health.rs`       | Rewrite `health_handler` to use per-model readiness from `ModelManager` |
| `lib/llm/src/http/service/openai.rs`       | Implement `check_model_ready()` for per-model request gating            |
| `lib/llm/src/http/service/openapi_docs.rs` | Update `/health` endpoint description                                   |


## Deferred to Implementation

- Whether `role`/`needs` should use string values or a shared Rust enum. Strings are simpler for cross-language serialization; enums are safer for type checking.
- Whether to expand `ModelType` to `u16` for `Decode`/`Encode` variants or keep role/needs as separate `ModelDeploymentCard` fields.
- `role`/`needs` are NOT part of the model checksum (deployment topology, not model identity).
- Exact placement of `check_model_ready()` in the request handler chain (before or after model name extraction).

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

- Does not meet REQ 3 (topology extensibility) or REQ 1 (per-model readiness for mixed deployments). Worker-declared dependencies solve both without a global flag.

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
- Frontend health handler: `lib/llm/src/http/service/health.rs`
- System status server health handler: `lib/runtime/src/system_status_server.rs`
- `check_ready()` no-op: `lib/llm/src/http/service/openai.rs:1743`
- Model manager: `lib/llm/src/discovery/model_manager.rs`
- Model: `lib/llm/src/discovery/model.rs`
- Worker set: `lib/llm/src/discovery/worker_set.rs`

## Terminology & Definitions


| Term                            | Definition                                                                                                                                   |
| ------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| **Aggregated serving**          | A single worker handles both prefill and decode phases for a request.                                                                        |
| **Disaggregated serving (P/D)** | Prefill and decode are handled by separate, specialized workers.                                                                             |
| **E/P/D**                       | Encode/Prefill/Decode — a three-stage disaggregated pipeline for multimodal models where vision encoding is offloaded to a dedicated worker. |
| **Topology**                    | The set of worker roles required to serve a model end-to-end (e.g., {prefill, decode} for P/D).                                              |
| **ModelManager**                | Rust component in the frontend that tracks registered models and their workers.                                                              |
| **ModelDeploymentCard**         | Metadata about a model instance registered by a worker, including model name, capabilities, and configuration.                               |


## Acronyms & Abbreviations

**DEP:** Dynamo Enhancement Proposal

**P/D:** Prefill/Decode (disaggregated serving)

**E/P/D:** Encode/Prefill/Decode (multimodal disaggregated serving)

**DGD:** DynamoGraphDeployment (Kubernetes custom resource)