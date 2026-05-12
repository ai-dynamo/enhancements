# Backend Interface

**Status**: Under Review

**Authors**: Tanmay Verma

**Category**: Architecture

**Sponsor**: [nnshah1](https://github.com/nnshah1)

**Required Reviewers**: [alec-flowers](https://github.com/alec-flowers), [ishandhanani](https://github.com/ishandhanani), [nv-yna](https://github.com/nv-yna)

**Review Date**: 03/11/2026

**Pull Request**: [PR 8003](https://github.com/ai-dynamo/dynamo/pull/8003)

**Implementation PR / Tracking Issue**: [PR 8003](https://github.com/ai-dynamo/dynamo/pull/8003)

**Prior Art**: [PR 6384](https://github.com/ai-dynamo/dynamo/pull/6384) (earlier, more complex proposal — see [Alt 4](#alt-4-backendhandler-abc-hierarchy-pr-6384))

# Summary

This proposal introduces a two-class abstraction that separates **runtime integration** (common across all backends) from **engine logic** (vLLM, SGLang, TensorRT-LLM, etc.):

* **`LLMEngine`** — an abstract base class with five methods (`from_args`, `start`, `generate`, `abort`, `cleanup`) that engine authors implement to wrap their framework-specific inference logic.
* **`Worker`** — a concrete class (not subclassed) that handles all Dynamo runtime concerns: creating the distributed runtime, installing signal handlers, registering the model, wiring up cancellation monitoring, serving the endpoint, and invoking engine cleanup on shutdown.

Configuration flows through a single **`WorkerConfig`** dataclass that the engine's `from_args()` factory method populates alongside the engine instance. Typed contracts — `GenerateRequest` and `GenerateChunk` (TypedDicts) and `EngineConfig` (dataclass) — define the request/response boundary between Worker and engine.

The current implementation supports the **aggregated workflow** only. Disaggregated serving, multi-modality, metrics, KV event publishing, and other features are tracked in the [Feature Gap Table](#feature-gap-tracking) and will be addressed incrementally by backend leads.

# Motivation

Today, each backend (TensorRT-LLM, vLLM, SGLang) independently implements its own worker lifecycle, request handling, cancellation monitoring, metrics setup, KV publisher management, and cleanup logic. This has led to:

1. **Significant code duplication.** ~600-1,000 lines of nearly identical logic is repeated across backends for cancellation monitoring, output processing, metrics setup, worker lifecycle, and cleanup.

2. **Inconsistent behavior.** Each backend handles edge cases (shutdown, cancellation, non-leader nodes, health checks) slightly differently, leading to subtle bugs and inconsistent user experience.

3. **High maintenance burden.** Bug fixes and feature additions to common logic must be applied to each backend independently, risking drift and regressions.

4. **Limited testability.** Without a clear interface boundary, unit testing backend logic requires standing up the full runtime stack.

5. **Difficult onboarding.** Writing a new backend requires understanding the full Dynamo runtime setup from scratch. There is no reference architecture or minimal example to follow.

An earlier proposal ([PR 6384](https://github.com/ai-dynamo/dynamo/pull/6384)) addressed these problems with a more complex design that was rejected as over-engineered (see [Alt 4](#alt-4-backendhandler-abc-hierarchy-pr-6384)). The current proposal prioritizes **simplicity and incremental adoption**: engine authors implement five methods, get a working backend, and additional features are added to `Worker` as gaps are closed.

## Goals

* Eliminate duplicated runtime lifecycle, cancellation, and registration logic across backends.
* Provide a minimal interface — five methods on `LLMEngine` — that new backend authors can implement to have a fully functional worker.
* Enable isolated unit and e2e testing of backends via a CPU-only sample engine.
* Maintain backward compatibility with existing deployment patterns via an opt-in `--unified` flag.
* Establish typed contracts (`GenerateRequest`, `GenerateChunk`, `EngineConfig`) for the boundary between Worker and engine.
* Address existing tracking issues: [DIS-1441](https://linear.app/nvidia/issue/DIS-1441/10-api-backend-worker-structure-and-interface-standardization), [DIS-1432](https://linear.app/nvidia/issue/DIS-1432/consistent-file-structures-across-engines), [DYN-1903](https://linear.app/nvidia/issue/DYN-1903/test-3-test-pyramid-structure-unit-integration-e2e)

### Non Goals

* Changing the Dynamo runtime, frontend, or router architectures.
* Modifying the request/response wire format between frontend and backend.
* Unifying engine-specific configuration (e.g., TensorRT-LLM build profiles, vLLM scheduler settings). Each backend retains its own engine-specific config.
* Supporting disaggregated serving, multi-modality, metrics, or KV event publishing in the initial implementation (tracked in [Feature Gap Table](#feature-gap-tracking)).
* Replacing existing backend entry points. The unified path is opt-in via `--unified`.

## Requirements

### REQ 1 LLMEngine Abstract Base Class

The interface **MUST** provide an `LLMEngine` abstract base class with four abstract methods — `from_args()`, `start()`, `generate()`, `cleanup()` — and one optional method — `abort()`. Engine authors **MUST** implement the four abstract methods to have a fully functional worker. The `abort()` method **SHOULD** be overridden by engines that can release resources (KV cache, scheduler slots) on request cancellation, but defaults to a no-op.

### REQ 2 Concrete Worker

The runtime integration **MUST** be handled by a concrete `Worker` class that engine authors do not subclass. `Worker` **MUST** orchestrate the following lifecycle in order: create distributed runtime, install signal handlers, call `engine.start()`, register the model, serve the endpoint (delegating to `engine.generate()` with cancellation wrapping), and call `engine.cleanup()` on shutdown.

### REQ 3 Configuration

Configuration **MUST** flow through a single `WorkerConfig` dataclass carrying all fields needed by `Worker`. `LLMEngine.from_args()` **MUST** return both the engine instance and a populated `WorkerConfig`. A `WorkerConfig.from_runtime_config()` factory method **SHOULD** be provided for engines whose existing config objects already carry the relevant runtime fields.

### REQ 4 Cancellation and Shutdown

`Worker` **MUST** wrap each `engine.generate()` call with a cancellation monitor that: (a) watches for per-request cancellation via `Context.async_killed_or_stopped()`, (b) calls `engine.abort(context)` when cancellation is detected, and (c) stops yielding chunks when `context.is_stopped()`. The engine **MUST NOT** need to implement its own cancellation monitoring for the common case.

### REQ 5 Typed Contracts

The interface **MUST** define typed request/response contracts:
* `GenerateRequest` (TypedDict): inbound request with required `token_ids` and optional `sampling_options`, `stop_conditions`, `output_options`.
* `GenerateChunk` (TypedDict): each yielded chunk with required `token_ids`, optional `finish_reason` and `completion_usage` (required on final chunk).
* `EngineConfig` (dataclass): registration metadata returned by `start()` — model name, context length, KV cache block size, total KV blocks, max sequences, max batched tokens.

### REQ 6 Backward Compatibility

The unified entry point **MUST** be opt-in via a `--unified` flag on the backend launch scripts. Existing entry points **MUST** remain the default and continue to work unchanged.

### REQ 7 Error Handling

`Worker` **MUST** wrap engine exceptions in `DynamoException` subtypes: `CannotConnect` for runtime creation failures, `EngineShutdown` for engine initialization failures, and `Unknown` for unexpected errors during `generate()`. Engines that raise `DynamoException` directly **MUST** have those exceptions passed through unwrapped.

### REQ 8 Sample Engine

The interface **MUST** include a `SampleLLMEngine` reference implementation that runs on CPU only, demonstrates all five `LLMEngine` methods, and serves as a copy-paste template for engine leads.

# Proposal

## Overview

The Backend Interface introduces the following public components in the `dynamo.common.backend` package:

| Component | Role |
| :---- | :---- |
| `LLMEngine` | Abstract base class defining the engine contract (5 methods) |
| `Worker` | Concrete class handling all Dynamo runtime integration |
| `WorkerConfig` | Flat dataclass with all fields needed by Worker |
| `EngineConfig` | Dataclass for engine registration metadata (returned by `start()`) |
| `GenerateRequest` | TypedDict for inbound request format |
| `GenerateChunk` | TypedDict for streaming response chunks |
| `run()` | Entry-point function: `run(EngineCls)` wires everything together |

All components are re-exported from `dynamo.common.backend`.

### Architecture

```
                    ┌──────────────────────────────────┐
                    │        LLMEngine (ABC)            │
                    │                                   │
                    │  from_args(argv)  [classmethod]   │
                    │    → (engine, WorkerConfig)        │
                    │                                   │
                    │  start()  → EngineConfig           │
                    │  generate(request, context)        │
                    │    → AsyncGenerator[GenerateChunk] │
                    │  abort(context)   [optional]       │
                    │  cleanup()                         │
                    └──────────┬────────────────────────┘
                               │
          ┌────────────────────┼─────────────────────┐
          │                    │                      │
 ┌────────▼─────┐   ┌─────────▼──────┐   ┌──────────▼──────┐
 │ VllmLLMEngine│   │SglangLLMEngine │   │TrtllmLLMEngine  │
 └──────────────┘   └────────────────┘   └─────────────────┘

                    ┌──────────────────────────────────┐
                    │        Worker (concrete)          │
                    │                                   │
                    │  __init__(engine, config)          │
                    │  generate(request, context)        │
                    │    → wraps engine.generate()       │
                    │    → adds cancellation monitoring  │
                    │  run()                             │
                    │    1. configure_dynamo_logging()   │
                    │    2. create_runtime()             │
                    │    3. install_signal_handlers()    │
                    │    4. engine.start()               │
                    │    5. register_model()             │
                    │    6. serve_endpoint()             │
                    │    7. engine.cleanup()             │
                    └──────────────────────────────────┘
```

Each backend provides a thin entry point that wires the two together:

```python
from dynamo.common.backend.run import run
from dynamo.vllm.llm_engine import VllmLLMEngine

def main():
    run(VllmLLMEngine)   # → from_args() → Worker(engine, config).run()
```

## LLMEngine Base Class

```python
class LLMEngine(ABC):
    @classmethod
    @abstractmethod
    async def from_args(
        cls, argv: list[str] | None = None
    ) -> tuple[LLMEngine, WorkerConfig]:
        """Parse CLI args and construct the engine (not yet started).
        Returns a (engine, worker_config) pair."""
        ...

    @abstractmethod
    async def start(self) -> EngineConfig:
        """Start the engine and return registration metadata.
        After this returns, generate() MUST be ready to accept calls."""
        ...

    @abstractmethod
    async def generate(
        self, request: GenerateRequest, context: Context
    ) -> AsyncGenerator[GenerateChunk, None]:
        """Yield streaming response chunks for a single request.
        Called concurrently for multiple in-flight requests.
        Final chunk must include finish_reason and completion_usage."""
        ...
        yield  # type: ignore[misc]

    async def abort(self, context: Context) -> None:
        """Abort an in-flight request (optional, default no-op).
        Override to release engine resources (KV cache, scheduler slots)."""

    @abstractmethod
    async def cleanup(self) -> None:
        """Release all engine resources.  Called once on shutdown."""
        ...
```

Engine authors implement `from_args()`, `start()`, `generate()`, and `cleanup()`. Override `abort()` if the engine can release resources on cancellation. That is the entire contract.

## Worker Class

```python
class Worker:
    def __init__(self, engine: LLMEngine, config: WorkerConfig):
        self.config = config
        self.engine = engine

    async def generate(
        self, request: GenerateRequest, context: Context
    ) -> AsyncGenerator[GenerateChunk, None]:
        async def _monitor_cancel():
            await context.async_killed_or_stopped()
            try:
                await self.engine.abort(context)
            except Exception:
                logger.debug("Error during request abort", exc_info=True)

        cancel_task = asyncio.create_task(_monitor_cancel())
        try:
            async for chunk in self.engine.generate(request, context):
                if context.is_stopped():
                    break
                yield chunk
        except DynamoException:
            raise
        except Exception as exc:
            raise Unknown(f"Engine generate failed: {exc}") from exc
        finally:
            if not cancel_task.done():
                cancel_task.cancel()
                try:
                    await cancel_task
                except asyncio.CancelledError:
                    pass

    async def run(self) -> None:
        configure_dynamo_logging()
        cfg = self.config
        shutdown_event = asyncio.Event()

        runtime, loop = create_runtime(
            discovery_backend=cfg.discovery_backend,
            request_plane=cfg.request_plane,
            event_plane=cfg.event_plane,
            use_kv_events=cfg.use_kv_events,
        )
        endpoint = runtime.endpoint(f"{cfg.namespace}.{cfg.component}.{cfg.endpoint}")
        install_signal_handlers(loop, runtime, [endpoint], shutdown_event)

        engine_config = await self.engine.start()

        try:
            model_type = parse_endpoint_types(cfg.endpoint_types)
            served_name = cfg.served_model_name or cfg.model_name

            await register_model(
                cfg.model_input, model_type, endpoint,
                cfg.model_name, served_name,
                context_length=engine_config.context_length,
                kv_cache_block_size=engine_config.kv_cache_block_size,
                custom_template_path=cfg.custom_jinja_template,
            )
            await endpoint.serve_endpoint(
                self.generate, graceful_shutdown=True,
                metrics_labels=cfg.metrics_labels,
            )
        finally:
            await self.engine.cleanup()
```

## Typed Contracts

```python
class GenerateRequest(TypedDict, total=False):
    token_ids: Required[list[int]]
    sampling_options: dict[str, Any]
    stop_conditions: dict[str, Any]
    output_options: dict[str, Any]

class GenerateChunk(TypedDict, total=False):
    token_ids: Required[list[int]]
    finish_reason: str
    completion_usage: dict[str, int]

@dataclass
class EngineConfig:
    model: str
    served_model_name: Optional[str] = None
    context_length: Optional[int] = None
    kv_cache_block_size: Optional[int] = None
    total_kv_blocks: Optional[int] = None
    max_num_seqs: Optional[int] = None
    max_num_batched_tokens: Optional[int] = None
```

## Configuration

```python
@dataclass
class WorkerConfig:
    namespace: str
    component: str = "backend"
    endpoint: str = "generate"
    model_name: str = ""
    served_model_name: Optional[str] = None
    model_input: ModelInput = ModelInput.Tokens
    endpoint_types: str = "chat,completions"
    discovery_backend: str = "etcd"
    request_plane: str = "tcp"
    event_plane: str = "nats"
    use_kv_events: bool = False
    custom_jinja_template: Optional[str] = None
    metrics_labels: list = field(default_factory=list)
```

Engines that already have a config object carrying runtime fields can use `WorkerConfig.from_runtime_config(runtime_cfg, model_name, ...)` instead of constructing `WorkerConfig` manually. Engine-specific config stays in the engine — `WorkerConfig` only carries what `Worker` needs.

## Minimal Engine Example

A complete engine can be implemented in ~80 lines. The `SampleLLMEngine` (`dynamo.common.backend.sample_engine`) serves as both a reference implementation and a CPU-only test target. The key method is `generate()`:

```python
class SampleLLMEngine(LLMEngine):
    async def generate(self, request, context):
        token_ids = request.get("token_ids", [])
        max_new = request.get("stop_conditions", {}).get("max_tokens") or self.max_tokens
        for i in range(max_new):
            if context.is_stopped():
                yield {"token_ids": [], "finish_reason": "cancelled",
                       "completion_usage": {
                           "prompt_tokens": len(token_ids),
                           "completion_tokens": i,
                           "total_tokens": len(token_ids) + i}}
                break
            await asyncio.sleep(self.delay)
            out: GenerateChunk = {"token_ids": [(i + 1) % 32000]}
            if i == max_new - 1:
                out["finish_reason"] = "length"
                out["completion_usage"] = {
                    "prompt_tokens": len(token_ids),
                    "completion_tokens": max_new,
                    "total_tokens": len(token_ids) + max_new}
            yield out
```

Worker centralizes ~600-1,000 lines of duplicated runtime lifecycle, cancellation, registration, and cleanup logic (across 3 backends) into ~185 lines.

# Gap Analysis and Future Work

## Feature Gap Tracking

The following tables track features **not yet supported** in the unified path. These will be addressed incrementally by backend leads.

### Common Gaps (all three engines)

| # | Feature | Description | Priority |
| :---- | :---- | :---- | :---- |
| 1 | Disaggregated serving | Prefill/decode worker split, bootstrap coordination, KV transfer | High |
| 2 | Metrics & Prometheus | Engine-level metrics, KV cache utilization gauges, Prometheus multiprocess registry setup | High |
| 3 | KV event publishing | Prefix cache events (BlockStored/Removed) to router via ZMQ or NATS | High |
| 4 | Health check payloads | Per-engine custom health check payloads (BOS token probe, etc.) | High |
| 5 | Logprobs | Selected token + top-k log probability extraction and streaming | Medium |
| 6 | Guided decoding / structured outputs | JSON schema, regex, grammar, choice constraints | Medium |
| 7 | OpenTelemetry tracing | `build_trace_headers()`, request performance metrics, OTEL service name propagation | Medium |
| 8 | Engine routes | Profiling (start/stop), memory release/resume, weight update (disk/tensor/distributed/IPC) | Medium |
| 9 | Data-parallel routing | DP rank extraction from routing hints, DP-aware scheduling | Medium |
| 10 | Text-in-text-out mode | OpenAI-compatible chat/completion with engine-side tokenization (currently only token-in-token-out) | Medium |
| 11 | Custom Jinja chat templates | `--custom-jinja-template` for model-specific prompt formatting | Low |
| 12 | Snapshot/checkpoint | CRIU-based engine state save/restore, identity reloading | Low |
| 13 | Benchmark mode | Self-profiling on startup (prefill/decode sweeps) | Low |

### vLLM-Specific Gaps

| # | Feature | Description |
| :---- | :---- | :---- |
| 14 | LoRA adapters | Dynamic load/unload/list, ModelDeploymentCard publishing, per-LoRA serialization locks |
| 15 | Multimodal (images/video) | Image/video loading, embedding caching, NIXL RDMA transfer, Qwen VL mRoPE support |
| 16 | Separate encode worker | `EncodeWorkerHandler` for multimodal encode-only disaggregation |
| 17 | Sleep/wake/quiesce | 3-level engine lifecycle control (weights, buffers, everything) |
| 18 | Elastic EP scaling | `scale_elastic_ep` with Ray node management |
| 19 | GMS shadow mode | GPU Memory Service integration with failover lock |
| 20 | ModelExpress P2P | Distributed model loading via P2P |
| 21 | KV block clearing | Prefix cache reset endpoint |

### SGLang-Specific Gaps

| # | Feature | Description |
| :---- | :---- | :---- |
| 22 | Embedding inference | `async_encode()` path, OpenAI embedding response format |
| 23 | Image diffusion | `DiffGenerator` for text-to-image (FLUX, etc.) with TP/DP |
| 24 | Video generation | `DiffGenerator` for text-to-video (Wan2.1, etc.) |
| 25 | LLM diffusion (DLLM) | Diffusion language model algorithm support |
| 26 | Multimodal encode worker | Front-facing `MMEncoder`, embedding LRU cache, NIXL transfer |
| 27 | Multimodal worker | Aggregated/disaggregated multimodal inference with `EmbeddingsProcessor` |
| 28 | Deferred signal handling | Monkey-patching `loop.add_signal_handler` to capture SGLang's internal signal registrations |
| 29 | Output modalities override | Required for diffusion workers (default `["text"]` must become `["image"]`/`["video"]`) |

### TRT-LLM-Specific Gaps

| # | Feature | Description |
| :---- | :---- | :---- |
| 30 | Custom logits processors | `TrtllmDynamoLogitsAdapter` with CUDA stream support |
| 31 | Attention DP scheduling | `SchedulingParams` with `attention_dp_rank` and `attention_dp_relax` |
| 32 | Video diffusion | Auto-detect pipeline from `model_index.json`, MP4 encoding, `MediaOutput` |
| 33 | Multimodal processing | `MultimodalRequestProcessor`, image URL processing, embedding injection |
| 34 | Encode helper (EPD) | Remote encode via `encode_client`, NIXL tensor reading |
| 35 | KV cache connector | KVBM connector config, consolidator ZMQ integration |
| 36 | Disable request abort | `disable_request_abort` config flag (default: skip abort) |
| 37 | Fatal vs per-request errors | Distinguishing `RequestError` (per-request, recoverable) from fatal engine errors |

### Recommended Migration Order

1. **Metrics & health checks** — needed for production observability
2. **Disaggregated serving** — largest architectural change, unlocks PD split
3. **KV event publishing** — required for KV-aware routing
4. **Logprobs + guided decoding** — most-requested inference features
5. **Multimodal / LoRA / diffusion** — modality-specific, can be parallelized across leads

## Strategy for Closing Feature Gaps

The core design principle is **extend, don't replace**: each gap should be closable by adding optional methods/fields to the existing interface rather than restructuring it. The 37 gaps fall into six categories of interface change:

| Change Type | Gaps Addressed | Approach |
| :---- | :---- | :---- |
| **Worker-internal steps** | #2, #3, #7, #11 | Add steps to `Worker.run()` (metrics setup, KV publishers, tracing). Engines don't see these changes. |
| **New optional methods on LLMEngine** | #4, #8, #12, #13, #14, #17, #21, #36 | Add methods with default no-op/sensible-default implementations (same pattern as `abort()`). E.g., `get_health_check_payload()`, `get_additional_endpoints()`, `sleep()`/`wake()`. The "4 abstract methods" contract never grows. |
| **Extended TypedDict fields** | #5, #6, #9, #10, #30, #31 | Add optional fields to `GenerateRequest`/`GenerateChunk` (e.g., `log_probs`, `text`, `guided_decoding`, `routing_hints`). Backward-compatible — existing engines ignore new fields. |
| **WorkerConfig field additions** | #1, #35 | Add `disaggregation_mode` to `WorkerConfig`. Worker adjusts `ModelType` flags during registration; engine configures itself for prefill/decode in `from_args()`. |
| **Separate processes for separate modalities** | #15, #22, #23, #24, #26, #27, #29, #32, #33, #34 | Diffusion, embeddings, and encode workers run as separate Worker instances with modality-specific engine subclasses. Keeps each process simple (one Worker + one engine) with clean interfaces per modality. Multimodal *inputs* (text + images in the same LLM request) are handled by adding optional fields to `GenerateRequest`. |
| **Engine-internal (no interface change)** | #18, #19, #20, #25, #28, #37 | Implemented entirely within the `LLMEngine` subclass. E.g., elastic EP, GMS shadow mode, deferred signal handling. |

**Guiding principles:**

1. **Worker absorbs runtime complexity.** If a feature is about Dynamo infrastructure (metrics, tracing, KV pub/sub), it belongs in Worker, not in LLMEngine.
2. **Optional methods over required.** New LLMEngine methods MUST default to no-op. The "4 abstract methods" contract doesn't grow.
3. **Extend TypedDicts, don't replace.** New request/response fields are always optional. Existing engines continue to work.
4. **Separate processes for separate modalities.** Don't overload one Worker with fundamentally different request/response shapes.
5. **Engine-specific stays in the engine.** If only one backend needs it, it doesn't belong on the ABC.

## Testing

Pre-merge e2e tests for each unified entry point:

| Test | What's Verified |
| :---- | :---- |
| `test_vllm.py::test_serve_deployment[aggregated_unified]` | vLLM unified path with chat + completion payloads |
| `test_trtllm.py::test_deployment[aggregated_unified]` | TRT-LLM unified path with chat + completion payloads |
| `test_sglang.py::test_sglang_deployment[aggregated_unified]` | SGLang unified path with chat + completion payloads |
| `test_sample.py::test_sample_deployment[aggregated]` | Sample backend e2e (CPU-only, `gpu_0` marker) |

Existing aggregated tests continue to pass — the unified path is additive.

## Deferred to Implementation

* Exact Prometheus metric names and label standardization across backends — to be confirmed during Phase 1.
* Whether non-LLM modalities (diffusion, embeddings) should use separate engine ABCs with modality-specific TypedDicts, or whether `LLMEngine` can be generalized — to be decided when the first non-LLM modality is implemented in Phase 4.
* Whether `EngineConfig` should carry additional metadata for multi-node/tensor-parallel deployments (e.g., `tp_size`, `pp_size`, `rank`).
* The exact KV event callback protocol (push vs pull, ZMQ vs NATS) — to be decided during Phase 2.

# Implementation Phases

## Phase 0: Interface, Sample Engine, and Backend Subclasses

**Release Target**: Current PR ([PR 8003](https://github.com/ai-dynamo/dynamo/pull/8003))

**Effort Estimate**: 1 engineer, completed

**Delivered:**

* `LLMEngine` ABC and `Worker` concrete class in `dynamo.common.backend`
* Typed contracts: `GenerateRequest`, `GenerateChunk`, `EngineConfig`, `WorkerConfig`
* `run()` entry-point function
* `SampleLLMEngine` reference implementation (CPU-only)
* `VllmLLMEngine`, `SglangLLMEngine`, `TrtllmLLMEngine` subclasses
* `unified_main.py` entry points for all three backends + sample
* `--unified` flag on `agg.sh` launch scripts for opt-in
* Pre-merge e2e tests for all four engines

**Not Supported:** Everything in the [Feature Gap Table](#feature-gap-tracking)

## Phase 1: Production Observability

**Release Target**: [TBD]

**Effort Estimate**: 1 engineer, ~1-2 weeks

**Scope:**

* Add metrics collection steps to `Worker.run()` (gap #2)
* Add `LLMEngine.get_health_check_payload()` optional method (gap #4)
* Add trace header extraction to `Worker.generate()` via `Context` (gap #7)

## Phase 2: Disaggregated Serving

**Release Target**: [TBD]

**Effort Estimate**: 1-2 engineers, ~2-4 weeks

**Scope:**

* Add `disaggregation_mode` field to `WorkerConfig` (gap #1)
* Add KV publisher setup/teardown to `Worker.run()` (gap #3)
* Engine subclasses configure themselves for prefill or decode mode in `from_args()` (gap #1)
* KV cache connector integration for TRT-LLM (gap #35)

## Phase 3: Inference Features

**Release Target**: [TBD]

**Effort Estimate**: 1 engineer per backend, ~1-2 weeks each

**Scope:**

* Add `log_probs` and `top_logprobs` optional fields to `GenerateChunk` (gap #5)
* Standardize `guided_decoding` in `GenerateRequest.sampling_options` across engines (gap #6)
* Add `text` field to `GenerateRequest` for text-in-text-out mode (gap #10)

## Phase 4: Multi-Modality and Backend-Specific Features

**Release Target**: [TBD]

**Effort Estimate**: Backend leads, parallelizable

**Scope:**

* Add `LLMEngine.get_additional_endpoints()` for LoRA management (#14), KV block clearing (#21), engine routes (#8)
* Add `multimodal_inputs` field to `GenerateRequest` for multimodal LLM inference (#15, #27, #33)
* Define modality-specific engines for diffusion (#23, #24, #32) and embeddings (#22)
* Engine-internal features: elastic EP (#18), GMS shadow mode (#19), deferred signal handling (#28), fatal error classification (#37)

# Alternate Solutions

## Alt 1: Move Interface into the Rust Core

Push `Worker.run()` lifecycle into Rust via PyO3, exposing a single `run_backend()` that accepts Python callbacks.

**Rejected:** High FFI complexity for marginal benefit. The lifecycle orchestrates Python objects (engines, configs) — Rust enforcement adds PyO3 indirection without helping engine authors who work in Python. Could revisit once the interface stabilizes.

## Alt 2: Shared Utility Functions

Extract duplicated logic into standalone functions (`setup_dynamo_runtime()`, `register_model()`, etc.) that each backend calls explicitly in its own `main()`.

**Rejected:** Does not enforce lifecycle ordering — the orchestration sequence itself is the most error-prone code to duplicate. A new engine author must know which utilities to call and in what order; with `Worker` + `LLMEngine`, they implement 4 abstract methods and get a working backend. Adding cross-cutting features requires updating every backend's `main()` vs. one change in `Worker`.

**Note:** Worker internally uses utility-style function calls, so these are not mutually exclusive. Backends with unique lifecycles can use the underlying utilities directly.

## Alt 3: Configuration-Driven Factory

A single `GenericBackend` class configured via callbacks, no subclassing.

**Rejected:** Callback-heavy APIs are harder to debug and document. Real backends have enough complexity that subclassing `LLMEngine` provides better ergonomics.

## Alt 4: Backend+Handler ABC Hierarchy (PR 6384)

The original proposal introduced a `Backend` ABC (12-step Template Method lifecycle with 8+ hooks) and a `Handler` ABC (`generate()`, `process_generation_output()`, `_cancellation_monitor()`, etc.), three-tier configuration (`DynamoRuntimeConfig` → `DynamoBackendConfig` → per-backend config), and multi-handler multi-modality via `get_additional_endpoints()`.

**Rejected:** Over-engineered for the current state. The hook explosion (8+ optional overrides) created a large surface area for subtle bugs. Three-tier config added indirection without benefit. The Handler ABC was unnecessary — `generate()` is simple enough to implement directly. The simpler `Worker` + `LLMEngine` approach provides the core value (deduplication, consistent lifecycle, easy onboarding) with far less complexity. Multi-modality and metrics can be added incrementally rather than designed up-front.

## Alt 5: Status Quo

**Rejected:** Does not address code duplication. Bug fixes must be applied to each backend independently. No standardized structure for new backends. Inconsistencies grow as backends evolve.

# Background

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **LLMEngine** | Abstract base class defining the engine contract — what backend authors implement |
| **Worker** | Concrete class handling all Dynamo runtime integration — not subclassed by backend authors |
| **WorkerConfig** | Flat dataclass carrying all fields needed by Worker |
| **EngineConfig** | Dataclass for engine registration metadata returned by `start()` |
| **GenerateRequest** | TypedDict defining the inbound request format |
| **GenerateChunk** | TypedDict defining the streaming response chunk format |
| **Disaggregated Inference** | Splitting inference into separate prefill and decode phases, potentially on different hardware |

## Acronyms & Abbreviations

**ABC:** Abstract Base Class

**CLI:** Command-Line Interface

**DEP:** Dynamo Enhancement Proposal

**KV:** Key-Value (in context of KV cache events)

**TRT-LLM:** TensorRT Large Language Model (NVIDIA inference framework)

**vLLM:** Virtual Large Language Model (open-source inference framework)

**SGLang:** Structured Generation Language (open-source inference framework)
