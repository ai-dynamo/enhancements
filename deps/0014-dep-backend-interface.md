# Backend Interface

**Status**: Draft

**Authors**: Tanmay Verma

**Category**: Architecture

**Sponsor**: [nnshah1](https://github.com/nnshah1)

**Required Reviewers**: [alec-flowers](https://github.com/alec-flowers), [ishandhanani](https://github.com/ishandhanani), [nv-yna](https://github.com/nv-yna)

**Review Date**: [TBD]

**Pull Request**: [PR 6384](https://github.com/ai-dynamo/dynamo/pull/6384)

**Implementation PR / Tracking Issue**: [TBD]

# Summary

This proposal introduces a standardized `Backend` and `Handler` abstract base class interface for all Dynamo LLM backend workers. The interface centralizes common lifecycle management, request handling, cancellation monitoring, metrics collection, and cleanup logic that is currently duplicated across the TensorRT-LLM, vLLM, and SGLang backends. A shared `DynamoRuntimeConfig` and `DynamoRuntimeArgGroup` provide consistent configuration and CLI argument handling across all backends.

The interface achieves feature parity with existing backends by supporting multiple modalities — text-only LLM generation, multimodal (vision/image inputs), image/video diffusion, and embeddings — through a multi-handler architecture where each modality is served by a separate `Handler` subclass registered on its own endpoint.

# Motivation

Today, each backend (TensorRT-LLM, vLLM, SGLang) independently implements its own worker lifecycle, request handling, cancellation monitoring, metrics setup, KV publisher management, and cleanup logic. This has led to:

1. **Significant code duplication.** An estimated 1,300-2,200 lines of nearly identical logic is repeated across backends for cancellation monitoring, output processing, trace headers, metrics setup, worker lifecycle, and cleanup.

2. **Inconsistent behavior.** Each backend handles edge cases (shutdown, cancellation, non-leader nodes, health checks) slightly differently, leading to subtle bugs and inconsistent user experience.

3. **High maintenance burden.** Bug fixes and feature additions to common logic (e.g., adding a new metrics label, changing cancellation semantics) must be applied to each backend independently, risking drift and regressions.

4. **Difficult onboarding.** Writing a new backend requires understanding the full Dynamo runtime setup, component registration, model registration, metrics, and KV publisher patterns from scratch. There is no reference architecture or minimal example to follow.

5. **Limited testability.** Without a clear interface boundary, unit testing backend logic requires standing up the full runtime stack. A well-defined interface enables mocking and isolated testing.

## Goals

* Eliminate duplicated lifecycle, handler, and configuration logic across backends.
* Enable isolated unit testing of both the base interface and individual backend implementations. 
* Standardize logging, metrics, health checks, and shutdown behavior across all backends.
* Maintain full backward compatibility with existing deployment patterns and CLI flags.
* Provide a minimal, well-documented interface that new backend authors can implement in under 100 lines of framework-specific code.
* Linear issue interface will help addressing: [DIS-1441](https://linear.app/nvidia/issue/DIS-1441/10-api-backend-worker-structure-and-interface-standardization), [DIS-1432](https://linear.app/nvidia/issue/DIS-1432/consistent-file-structures-across-engines), [DYN-1903](https://linear.app/nvidia/issue/DYN-1903/test-3-test-pyramid-structure-unit-integration-e2e)

### Non Goals

* Changing the Dynamo runtime, frontend, or router architectures.
* Modifying the request/response wire format between frontend and backend.
* Unifying engine-specific configuration (e.g., TensorRT-LLM build profiles, vLLM scheduler settings). Each backend retains its own engine-specific config.
* Replacing existing backend entry points. Backends can migrate incrementally.

## Requirements

### REQ 1 Abstract Base Classes

The interface **MUST** provide a `Backend` abstract base class and a `Handler` abstract base class. Backend authors **MUST** implement a minimal set of abstract methods (`create_engine`, `create_handler`, `get_health_check_payload` on Backend; `generate` on Handler) to have a fully functional worker.

### REQ 2 Lifecycle Orchestration

The `Backend` base class **MUST** orchestrate the complete worker lifecycle in a deterministic order: runtime setup, component creation, engine creation, metrics setup, handler creation, KV publisher setup, route registration, model registration, and serving. Backend authors **SHOULD** only override specific hook methods rather than reimplementing the lifecycle.

### REQ 3 Shared Configuration

A `DynamoRuntimeConfig` class extending `ConfigBase` **MUST** provide all common configuration fields (namespace, component, endpoint, model, runtime planes, connectors, etc.). Backend-specific configs **MUST** extend this base class and set the `engine` attribute to a typed engine-specific config object so that standard Dynamo fields (accessed as `config.<field>`) are clearly separated from engine-specific fields (accessed as `config.engine.<field>`). A `DynamoRuntimeArgGroup` extending `ArgGroup` **MUST** provide reusable argparse argument definitions for all common fields.

### REQ 4 Cancellation and Shutdown

The `Handler` base class **MUST** provide a `_cancellation_monitor()` async context manager that monitors both per-request cancellation (via `Context`) and global shutdown events. All backends **MUST** use this mechanism instead of implementing their own.

### REQ 5 Metrics Standardization

The `Backend` base class **MUST** support a standard metrics collection pattern using a shared Prometheus `CollectorRegistry` (`DYNAMO_COMPONENT_REGISTRY`). Backends **SHOULD** use the `setup_metrics()` hook for framework-specific metric registration.

### REQ 6 Output Processing Utilities

The `Handler` base class **MUST** provide a `process_generation_output()` utility method that extracts token deltas, logprobs, and finish reasons from cumulative engine output. Backends **SHOULD** use this method instead of implementing their own extraction logic.

### REQ 7 Backward Compatibility

Migrating a backend to the new interface **MUST NOT** change its external behavior: CLI flags, deployment patterns, request/response formats, and metrics names **MUST** remain unchanged. 

### REQ 8 Extensibility via Hooks

The `Backend` base class **MUST** provide optional hook methods for customization points including: `pre_runtime_setup()`, `engine_context()`, `setup_metrics()`, `is_non_leader_node()`, `handle_non_leader_node()`, `setup_kv_publishers()`, `register_engine_routes()`, `extract_runtime_config()`, `register_and_serve()`, and cleanup hooks. Backends **MAY** override any subset of these hooks. This is for ease of migration. As these implementations mature, we can reduce the number of hooks. 

### REQ 9 Multi-Modality Support

The interface **MUST** support multiple modalities beyond text-only LLM generation. Specifically:

* Multimodal inputs (vision/image) **MUST** be supportable via separate `Handler` subclasses that process multimodal request formats (e.g., `MultimodalHandler` alongside `LLMHandler`).
* Image and video diffusion **MUST** be supportable by registering a diffusion `Handler` on a dedicated endpoint (e.g., `images/generations`).
* Embeddings **MUST** be supportable by registering an embedding `Handler` on a dedicated endpoint (e.g., `embeddings`).
* Backends **MUST** be able to register multiple handlers on separate endpoints via `get_additional_endpoints()`.
* The `Handler` base class **MUST NOT** assume LLM-specific response formats (e.g., `token_ids`). The `generate()` contract yields `Dict` and each modality defines its own response schema.

### REQ 10 Feature Parity

Migrating a backend to the new interface **MUST** preserve all existing modality support:

* **vLLM:** multimodal (vision), LoRA hot-loading, disaggregated multimodal prefill/decode.
* **SGLang:** multimodal, image diffusion, embeddings, multimodal encode workers, multimodal preprocessor workers.
* **TRT-LLM:** multimodal, video generation, encode workers, disaggregated prefill/decode.

# Proposal

## Overview

The Backend Interface introduces four public components in the `dynamo.backend` package:

| Component | Role |
| :---- | :---- |
| `Backend` | Abstract base class orchestrating the full worker lifecycle via Template Method pattern |
| `Handler` | Abstract base class defining the request handling contract with built-in utilities |
| `DynamoRuntimeConfig` | Base dataclass for all backend configurations |
| `DynamoRuntimeArgGroup` | Reusable argparse group for common CLI flags |

### Architecture

```
                    ┌─────────────────────────────────┐
                    │         Backend (ABC)            │
                    │                                  │
                    │  run()  ─── lifecycle steps:     │
                    │    1. pre_runtime_setup()        │
                    │    2. setup_runtime()            │
                    │    3. setup_component()          │
                    │    4. engine_context()           │
                    │    5. create_engine()       [*]  │
                    │    6. setup_metrics()            │
                    │    7. is_non_leader_node()       │
                    │    8. create_handler()      [*]  │
                    │    9. setup_kv_publishers()      │
                    │   10. register_engine_routes()   │
                    │   11. register_and_serve()       │
                    │   12. cleanup                    │
                    │                                  │
                    │  get_additional_endpoints()      │
                    │    → register modality handlers  │
                    │                                  │
                    │  [*] = abstract (must implement) │
                    └──────────┬──────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──┐   ┌────────▼───┐   ┌───────▼────┐
    │ TrtllmBack │   │ VllmBack   │   │ SglangBack │
    │   end      │   │   end      │   │   end      │
    └────────────┘   └────────────┘   └────────────┘

                    ┌─────────────────────────────────┐
                    │         Handler (ABC)            │
                    │                                  │
                    │  generate()              [*]     │
                    │  process_generation_output()     │
                    │  get_trace_header()              │
                    │  _cancellation_monitor()         │
                    │  cleanup()                       │
                    └──────────┬──────────────────────┘
                               │
         ┌─────────────┬───────┴───────┬──────────────┐
         │             │               │              │
    ┌────▼─────┐ ┌─────▼────┐  ┌──────▼───┐  ┌──────▼─────┐
    │ LLM      │ │Multimodal│  │Diffusion │  │ Embedding  │
    │ Handler  │ │ Handler  │  │ Handler  │  │  Handler   │
    └──────────┘ └──────────┘  └──────────┘  └────────────┘
       /v1/        /v1/         /v1/images/    /v1/
     chat/       chat/        generations    embeddings
    completions completions
```

### Multi-Handler Architecture

A single `Backend` instance can serve multiple modalities by registering separate `Handler` subclasses on different endpoints. The primary handler (returned by `create_handler()`) serves the main `generate` endpoint. Additional handlers for other modalities are registered via `get_additional_endpoints()`:

```
┌──────────────────────────────────────────────────────────┐
│                  SglangBackend.run()                      │
│                                                          │
│  Engine ──► create_handler() ──► LLMHandler              │
│     │         (main endpoint: "generate")                │
│     │                                                    │
│     └──► get_additional_endpoints() ──►                  │
│            ├─ "images/generations" → DiffusionHandler     │
│            ├─ "embeddings"         → EmbeddingHandler     │
│            └─ "encode"             → MultimodalEncHandler │
└──────────────────────────────────────────────────────────┘
```

Each handler is a full `Handler` subclass with its own `generate()` implementation and response format. The `Backend` base class registers all endpoints in `register_and_serve()` and manages lifecycle (startup, metrics, cleanup) for all handlers uniformly.

## Design Patterns

1. **Template Method** - `Backend.run()` defines the algorithm skeleton; subclasses override specific steps.
2. **Factory Method** - `create_engine()` and `create_handler()` let subclasses determine which objects to create.
3. **Hook Method** - Optional override points (`pre_runtime_setup()`, `engine_context()`, etc.) allow customization without modifying the base lifecycle.
4. **Async Context Manager** - `engine_context()` and `_cancellation_monitor()` provide clean resource management.

## Backend Base Class

The `Backend` class provides a `run()` method that orchestrates the complete worker lifecycle:

```python
class Backend(ABC):
    def __init__(self, config: DynamoRuntimeConfig, runtime=None):
        self.config = config
        self.runtime = runtime
        self.shutdown_event = asyncio.Event()
        # ... other attributes

    @abstractmethod
    async def create_engine(self) -> Any:
        """Create and return the framework-specific inference engine."""

    @abstractmethod
    def create_handler(self, engine, component, endpoint) -> Handler:
        """Create and return a Handler instance for this backend."""

    @abstractmethod
    def get_health_check_payload(self, engine) -> Dict[str, Any]:
        """Return the payload used for health check requests."""

    async def run(self):
        """Orchestrate the complete worker lifecycle."""
        self.pre_runtime_setup()
        async with self.setup_runtime() as runtime:
            self.runtime = runtime
            self.component, self.endpoint = await self.setup_component()
            async with self.engine_context():
                self.engine = await self.create_engine()
                self._metrics_task = self.setup_metrics(self.endpoint)
                if self.is_non_leader_node():
                    await self.handle_non_leader_node()
                    return
                self.handler = self.create_handler(...)
                await self.setup_kv_publishers()
                await self.register_engine_routes()
                await self.register_and_serve()
                # cleanup on exit
```

Backend authors implement only the three abstract methods. All lifecycle plumbing is handled by the base class.

## Handler Base Class

The `Handler` class provides the request handling contract and reusable utilities:

```python
class Handler(ABC):
    def __init__(self, component, shutdown_event=None, kv_publishers=None):
        self.component = component
        self.shutdown_event = shutdown_event
        self.kv_publishers = kv_publishers

    @abstractmethod
    async def generate(self, request: Dict, context: Context) -> AsyncGenerator[Dict]:
        """Process a request and yield response chunks."""

    def process_generation_output(self, output, num_output_tokens_so_far):
        """Extract token deltas, logprobs, finish_reason from cumulative output."""

    def get_trace_header(self, context) -> Optional[Dict]:
        """Create W3C traceparent header from request context."""

    @asynccontextmanager
    async def _cancellation_monitor(self, context, abort_callback=None):
        """Monitor for request cancellation and shutdown events."""

    def cleanup(self):
        """Clean up KV publishers, temp dirs, and engine resources."""
```

### Response Formats by Modality

The `generate()` method yields `Dict` objects whose schema varies by modality. The `Handler` base class does not enforce a specific schema — each modality defines its own:

**LLM Generation (text-only and multimodal):**
```python
{
    "token_ids": [123, 456],       # Token IDs (delta, not cumulative)
    "finish_reason": "stop",       # Optional: "stop", "length", etc.
    "log_probs": [...],            # Optional: per-token log probabilities
    "top_logprobs": [...],         # Optional: top-k log probabilities
    "stop_reason": "...",          # Optional: stop string/token
}
```

**Image Diffusion (single yield at completion):**
```python
{
    "images": [{"url": "...", "b64_json": "..."}],
    "model": "stable-diffusion-xl",
    "finish_reason": "stop",
}
```

**Embeddings (single yield at completion):**
```python
{
    "embeddings": [[0.1, 0.2, ...]],
    "model": "text-embedding-3",
    "usage": {"prompt_tokens": 10, "total_tokens": 10},
    "finish_reason": "stop",
}
```

The base class utilities (`process_generation_output()`, etc.) are available for LLM handlers but are not mandatory — diffusion and embedding handlers use their own response construction logic.

## Shared Configuration

`DynamoRuntimeConfig` provides all common fields:

```python
@dataclass
class DynamoRuntimeConfig:
    namespace: str
    component: str = "backend"
    endpoint: str = "generate"
    model: str
    served_model_name: Optional[str] = None
    store_kv: str          # "etcd", "file", "mem"
    request_plane: str     # "tcp", "nats", "http"
    event_plane: str       # "nats", "zmq"
    use_kv_events: bool = False
    connector: list[str]
    # ... inference options, debug options
```

Backend-specific configs simply extend it:

```python
@dataclass
class VllmConfig(DynamoRuntimeConfig):
    gpu_memory_utilization: float = 0.9
    tensor_parallel_size: int = 1
    # ... vLLM-specific fields
```

## Minimal Backend Example

A complete backend can be implemented in ~100 lines of framework-specific code:

```python
# backends.py
class ExampleBackend(Backend):
    async def create_engine(self):
        return None  # No real engine needed

    def create_handler(self, engine, component, endpoint):
        return ExampleHandler(component=component,
                              shutdown_event=self.shutdown_event)

    def get_health_check_payload(self, engine):
        return {"token_ids": [1], "sampling_options": {"temperature": 0.0},
                "stop_conditions": {"max_tokens": 1}}

# handlers.py
class ExampleHandler(Handler):
    async def generate(self, request, context):
        async with self._cancellation_monitor(context):
            for i, token_id in enumerate(reply_ids):
                yield {"token_ids": [token_id]}
            yield {"token_ids": [], "finish_reason": "stop"}

# __main__.py
async def _run():
    config = parse_args()
    await ExampleBackend(config=config).run()
```

## What Gets Deduplicated

| Duplicated Logic | Current Lines (x3 backends) | After Migration |
| :---- | :---- | :---- |
| Worker lifecycle management | ~200-300 per backend | 0 (in base `Backend.run()`) |
| Cancellation monitoring | ~50-100 per backend | 0 (in base `Handler._cancellation_monitor()`) |
| Output delta extraction | ~40-60 per backend | 0 (in base `Handler.process_generation_output()`) |
| Trace header creation | ~20-30 per backend | 0 (in base `Handler.get_trace_header()`) |
| Metrics & publisher setup | ~100-200 per backend | 0 (in base `Backend.setup_metrics()`) |
| Cleanup & shutdown | ~30-50 per backend | 0 (in base `Handler.cleanup()`) |
| CLI argument definitions | ~80-120 per backend | 0 (in `DynamoRuntimeArgGroup`) |
| **Total** | **~1,300-2,200 lines** | **Centralized in ~920 lines of base classes** |

# Implementation Details

## Disaggregation Support

The interface supports disaggregated inference modes via the `disaggregation_mode` config field (`"aggregated"`, `"prefill"`, `"decode"`). The `Backend` base class:

- Passes the mode to `register_model()` for correct `ModelType` flags.
- Automatically disables tool/reasoning parsers in prefill mode.
- Each backend can create different handler instances based on the mode in `create_handler()`.

## Multi-Node Support

For tensor-parallel or pipeline-parallel deployments:

- Override `is_non_leader_node()` to return `True` for non-rank-0 processes.
- Default `handle_non_leader_node()` blocks indefinitely (non-leaders don't serve requests).
- Can be overridden for custom non-leader behavior (e.g., health reporting).

## Multimodal and Vision Support

Multimodal requests (e.g., images alongside text in chat completions) are handled by separate `Handler` subclasses rather than adding multimodal logic to the base LLM handler. This keeps handlers focused and testable.

**Current state across backends:**

| Backend | Multimodal Approach |
| :---- | :---- |
| **vLLM** | `MultimodalDecodeWorkerHandler`, `MultimodalPDWorkerHandler` — separate handler subclasses selected based on `enable_multimodal` config |
| **SGLang** | `MultimodalWorkerHandler`, `MultimodalPrefillWorkerHandler`, `MultimodalEncodeWorkerHandler`, `MultimodalProcessorHandler` — separate workers per role |
| **TRT-LLM** | `MultimodalRequestProcessor` preprocessing with `PrefillHandler`/`AggregatedHandler` — integrated into existing handlers with a preprocessor |

**Proposed approach:** Each backend implements multimodal support by providing a separate handler class:

```python
class VllmBackend(Backend):
    def create_handler(self, engine, component, endpoint):
        if self.config.enable_multimodal:
            return VllmMultimodalHandler(engine, component, ...)
        return VllmLLMHandler(engine, component, ...)
```

For disaggregated multimodal (separate encode/prefill/decode workers), each worker is a separate `Backend` instance with its own handler:

```python
# Launched as separate processes
await VllmMultimodalEncodeBackend(config).run()   # Encode worker
await VllmMultimodalPrefillBackend(config).run()  # Prefill worker
await VllmMultimodalDecodeBackend(config).run()   # Decode worker
```

This matches the existing deployment pattern (separate processes per worker role) while standardizing each worker's lifecycle through the `Backend` interface.

## Image and Video Diffusion Support

Diffusion models (image/video generation) have fundamentally different request/response patterns from LLM generation — they accept a text prompt and return complete images rather than streaming tokens. The interface handles this via a dedicated `Handler` subclass registered on a separate endpoint.

**Current state across backends:**

| Backend | Diffusion Approach |
| :---- | :---- |
| **SGLang** | `ImageDiffusionWorkerHandler` — uses `DiffGenerator`, serves `/v1/images/generations` |
| **TRT-LLM** | `VideoGenerationHandler` — uses TRT pipeline, serves video generation endpoint |
| **vLLM** | Not implemented |

**Proposed approach:**

```python
class SglangDiffusionBackend(Backend):
    async def create_engine(self):
        return DiffGenerator(self.config.model, ...)

    def create_handler(self, engine, component, endpoint):
        return DiffusionHandler(engine, component, ...)

    def get_health_check_payload(self, engine):
        return {"prompt": "test", "size": "64x64", "num_inference_steps": 1}
```

The `DiffusionHandler.generate()` yields a single response dict containing the generated image(s):

```python
class DiffusionHandler(Handler):
    async def generate(self, request, context):
        async with self._cancellation_monitor(context):
            images = await self.engine.generate_images(request)
            yield {"images": images, "finish_reason": "stop"}
```

Note that diffusion handlers still benefit from the base `Handler` utilities: `_cancellation_monitor()` for shutdown handling, `get_trace_header()` for distributed tracing, and `cleanup()` for resource management. Only `process_generation_output()` is LLM-specific and not used by diffusion handlers.

## Embeddings Support

Embedding models return fixed-size vectors rather than streaming tokens. Like diffusion, they are served via a dedicated handler on a separate endpoint.

**Current state:** Only SGLang implements `EmbeddingWorkerHandler`, which calls `engine.async_encode()` and returns OpenAI-compatible embedding responses.

**Proposed approach:**

```python
class SglangEmbeddingBackend(Backend):
    async def create_engine(self):
        return create_embedding_engine(self.config)

    def create_handler(self, engine, component, endpoint):
        return EmbeddingHandler(engine, component, ...)
```

## Additional Endpoints

Backends can serve extra endpoints beyond the main `generate` endpoint by overriding `get_additional_endpoints()`. This is the mechanism for registering modality-specific handlers alongside the primary handler:

```python
def get_additional_endpoints(self):
    return {
        "images/generations": self.diffusion_handler.generate,
        "embeddings": self.embedding_handler.generate,
        "clear_kv_blocks": self.handler.clear_kv_blocks,
        "load_lora": self.handler.load_lora,
    }
```

The `Backend` base class registers all additional endpoints in `register_and_serve()`:

```python
async def register_and_serve(self):
    await self.register_model(self.endpoint, self.engine)
    serve_coros = [self.endpoint.serve_endpoint(self.handler.generate, ...)]
    for ep_name, ep_handler in self.get_additional_endpoints().items():
        additional_ep = self.component.endpoint(ep_name)
        serve_coros.append(additional_ep.serve_endpoint(ep_handler, ...))
    await asyncio.gather(*serve_coros)
```

## Feature Parity Matrix

The following table shows which modalities each backend currently supports and confirms the interface can handle all of them:

| Feature | vLLM | SGLang | TRT-LLM | Interface Mechanism |
| :---- | :---- | :---- | :---- | :---- |
| Text-only LLM | Yes | Yes | Yes | Primary `Handler` via `create_handler()` |
| Multimodal (vision) | Yes | Yes | Yes | Separate `MultimodalHandler` subclass |
| Disaggregated prefill/decode | Yes | Yes | Yes | Separate `Backend` instances per worker role |
| Disaggregated multimodal | Yes | Yes | Yes | Separate `Backend` + `MultimodalHandler` per role |
| Image diffusion | No | Yes | No | `DiffusionHandler` via `get_additional_endpoints()` or standalone `Backend` |
| Video generation | No | No | Yes | `VideoHandler` via `get_additional_endpoints()` or standalone `Backend` |
| Embeddings | No | Yes | No | `EmbeddingHandler` via `get_additional_endpoints()` or standalone `Backend` |
| LoRA hot-loading | Yes | No | No | Additional endpoints (`load_lora`, `unload_lora`) |
| KV block management | Yes | No | No | Additional endpoint (`clear_kv_blocks`) |
| Multimodal encode worker | No | Yes | Yes | Standalone `Backend` with encode `Handler` |
| Multimodal preprocessor | No | Yes | Yes | Standalone `Backend` with preprocessor `Handler` |

## Unit Testing

A key goal of the Backend Interface is enabling isolated unit testing without requiring the full Dynamo runtime stack. The interface's clean separation of concerns — with small, mockable methods and dependency injection via `Context` — makes this possible.

### Test Coverage

The interface ships with demonstration unit tests covering both the base classes and the example backend:

**Handler base class tests** (`dynamo/backend/tests/test_handler.py`):

| Test Area | Tests | What's Verified |
| :---- | :---- | :---- |
| `process_generation_output()` | 5 | Token delta extraction, finish/stop reason inclusion, token count updates |
| `get_trace_header()` | 4 | W3C traceparent header formatting, None handling for missing trace fields |
| `cleanup()` | 4 | KV publisher shutdown, temp dir cleanup, error resilience, None handling |
| `add_temp_dir()` | 2 | Temp dir tracking, None rejection |
| `_cancellation_monitor()` | 2 | Task creation, automatic cancellation on context exit |

**Configuration tests** (`dynamo/backend/tests/test_config.py`):

| Test Area | Tests | What's Verified |
| :---- | :---- | :---- |
| `DynamoRuntimeConfig.validate()` | 5 | durable_kv_events → enable_local_indexer derivation, Jinja template path validation |
| `DynamoRuntimeConfig.get_model_name()` | 2 | served_model_name preference, model fallback |

**Example backend tests** (`dynamo/example_backend/tests/test_example_handler.py`):

| Test Area | Tests | What's Verified |
| :---- | :---- | :---- |
| `ExampleHandler` token_delay | 7 | Zero delay speed, latency scaling, cancellation, token format, fallback reply IDs |

### Testing Pattern

Tests use lightweight mocks for the `Context` interface and engine output objects, avoiding any dependency on the Rust runtime or real LLM engines:

```python
from dynamo.backend.handler import Handler

# Concrete subclass for testing the base class
class _ConcreteHandler(Handler):
    async def generate(self, request, context):
        yield {"token_ids": [1]}

# Mock engine output
output = MagicMock()
output.token_ids = [10, 20, 30]
output.finish_reason = "stop"

# Test the base class utility directly
result, count = Handler.process_generation_output(output, num_output_tokens_so_far=1)
assert result["token_ids"] == [20, 30]
assert result["finish_reason"] == "stop"
```

This demonstrates that backend authors can write unit tests for their `Handler` subclasses with simple mocks, validating generation logic, cancellation behavior, and output formatting without standing up the full serving stack.

## Deferred to Implementation

* Exact Prometheus metric names and label standardization across backends (to be confirmed during vLLM/TRT-LLM migration PRs).
* Whether `engine_context()` should be mandatory or optional for backends that don't use async context managers for their engines.
* Whether multimodal/diffusion/embedding modalities should use standalone `Backend` instances (separate processes) or a single `Backend` with multiple handlers via `get_additional_endpoints()`. Both patterns are supported; the right choice depends on resource isolation and deployment flexibility needs per backend.
* Whether a `HandlerType` enum or registry should be formalized for handler selection based on config (currently left to `create_handler()` logic in each backend).

# Implementation Phases

## Phase 0: Interface and Example Backend

**Release Target**: Current PR

**Effort Estimate**: 1 engineer, completed

**Supported API / Behavior:**

* `Backend` and `Handler` abstract base classes with full lifecycle orchestration
* `DynamoRuntimeConfig` and `DynamoRuntimeArgGroup` for shared configuration
* `example_backend` demonstrating minimal implementation
* Unit tests for Handler base class (17 tests), DynamoRuntimeConfig (7 tests), and ExampleHandler (7 tests)
* All hook methods with sensible defaults

**Not Supported:**

* Migration of existing backends (deferred to Phase 1-3)

## Phase 1: TensorRT-LLM Migration

**Release Target**: [TBD]

**Effort Estimate**: 1 engineer, ~1-2 weeks

**Work Item(s):** [TBD]

**Supported API / Behavior:**

* `TrtllmBackend` extending `Backend` with TensorRT-LLM engine creation
* TRT-LLM handlers extending `Handler` for aggregated, prefill, and decode modes
* All existing CLI flags preserved via `TrtllmConfig` extending `DynamoRuntimeConfig`

**Not Supported:**

* Changes to TensorRT-LLM engine internals

## Phase 2: vLLM Migration

**Release Target**: [TBD]

**Effort Estimate**: 1 engineer, ~1-2 weeks

**Work Item(s):** [TBD]

**Supported API / Behavior:**

* `VllmBackend` extending `Backend` with vLLM engine creation
* vLLM handlers extending `Handler` for all modes including multimodal and LoRA
* Existing deployment patterns and CLI flags preserved

**Not Supported:**

* Changes to vLLM engine internals

## Phase 3: SGLang Migration

**Release Target**: [TBD]

**Effort Estimate**: 1 engineer, ~1-2 weeks

**Work Item(s):** [TBD]

**Supported API / Behavior:**

* `SglangBackend` extending `Backend` with SGLang engine creation
* SGLang handlers extending `Handler` for LLM, diffusion, and embedding modes
* Existing deployment patterns and CLI flags preserved

**Not Supported:**

* Changes to SGLang engine internals

# Alternate Solutions

## Alt 1: Move Interface Implementation into the Rust Core

The Dynamo Rust runtime (`lib/runtime/src/`) already implements much of the infrastructure that the Python `Backend` base class orchestrates: `DistributedRuntime` handles component/endpoint creation, `register_llm()` (via PyO3 in `lib/bindings/python/rust/llm/`) publishes model metadata to discovery, `endpoint.serve_endpoint()` connects Python handlers to the Rust request plane, and `SystemHealth` manages health state. This alternative would push the lifecycle orchestration (the `Backend.run()` template) down into Rust, exposing it to Python as a single `run_backend()` call that accepts Python callbacks for the framework-specific steps.

**What would move to Rust:**

* Lifecycle ordering (the 12-step `run()` sequence) enforced in Rust, not Python.
* Component/endpoint creation, model registration, and health check coordination (already in Rust, currently re-orchestrated from Python).
* Graceful shutdown signal handling and cleanup ordering.
* Configuration validation (Rust's type system catches errors at compile time).

**What must stay in Python:**

* `create_engine()` - TRT-LLM, vLLM, and SGLang engines are Python-only.
* `generate()` handler logic - framework-specific async generators.
* `extract_runtime_config()` - requires dynamic introspection of Python engine objects.
* Framework-specific metrics hooks (Prometheus collectors tied to Python engine internals).

**Pros:**

* Rust's type system enforces lifecycle correctness at compile time rather than relying on Python ABC conventions.
* Shutdown and cancellation handling in Rust is more robust (no GIL contention, deterministic resource cleanup).
* Reduces the Python-side surface area - backend authors only provide callbacks, not a subclass.
* Aligns with the project's long-term direction of having more logic in the Rust core.
* However, none of these pieces seems to be on the critical path.

**Cons:**

* **High implementation cost.** The PyO3 boundary between Rust and Python async is already complex (`PythonAsyncEngine` in `lib/bindings/python/rust/lib.rs` uses manual future polling). Adding lifecycle callbacks (async context managers, factory methods returning Python objects) significantly increases FFI complexity.
* **Harder to iterate.** Changing a lifecycle hook requires modifying Rust code, recompiling, and updating PyO3 bindings - a much slower development cycle than editing a Python base class. Given that we are still discovering the right hook points (see Deferred to Implementation), this rigidity is premature.
* **Debugging difficulty.** When lifecycle issues arise, developers must trace across the Rust-Python boundary. Stack traces, breakpoints, and logging become fragmented. Backend authors (who work primarily in Python) lose the ability to read and step through the orchestration code.
* **Limited flexibility for hooks.** Python's hook/override pattern (e.g., `engine_context()` as an async context manager, `register_and_serve()` override for custom serve patterns) maps poorly to Rust callbacks passed via PyO3. Expressing "optionally override this async context manager" across FFI requires significant boilerplate.
* **Backend authors must still write Python.** Since engines and handlers remain in Python, the primary audience (backend developers) gains little from the Rust enforcement - they still subclass in Python, just with a less readable orchestration layer.

**Reason Rejected:**

* The cost-benefit ratio is unfavorable at this stage. The Rust core already handles the low-level infrastructure (networking, discovery, request routing). The lifecycle orchestration layer is inherently a Python-domain concern because it coordinates Python objects (engines, handlers, configs). Moving it to Rust adds FFI complexity without meaningfully benefiting backend authors. This could be revisited in the future once the interface stabilizes and the hook points are proven, potentially as a "Phase 4" hardening step.

## Alt 2: Shared Utility Functions Instead of an Interface

Instead of an abstract base class hierarchy, this alternative extracts the duplicated logic into standalone utility functions that each backend calls explicitly:

```python
# dynamo/backend/utils.py
async def setup_dynamo_runtime(config) -> DistributedRuntime: ...
async def create_component_and_endpoint(runtime, config) -> (Component, Endpoint): ...
async def register_model(endpoint, config, runtime_config) -> None: ...
def start_metrics_collection(endpoint, engine, registry) -> Task: ...
async def setup_kv_publishers(component, config) -> list: ...
def create_cancellation_monitor(context, shutdown_event, abort_cb) -> AsyncContextManager: ...
def process_generation_output(output, num_tokens_so_far) -> (dict, int): ...
def build_trace_header(context) -> Optional[dict]: ...
async def cleanup_handler(handler, publishers, temp_dirs) -> None: ...
```

Each backend would call these functions in its own `main()`:

```python
# vllm/main.py
async def main():
    config = parse_vllm_args()
    runtime = await setup_dynamo_runtime(config)
    component, endpoint = await create_component_and_endpoint(runtime, config)
    engine = create_vllm_engine(config)  # framework-specific
    metrics_task = start_metrics_collection(endpoint, engine, registry)
    handler = VllmHandler(engine, component)
    await register_model(endpoint, config, extract_vllm_runtime_config(engine))
    await endpoint.serve_endpoint(handler.generate)
```

**Pros:**

* **No inheritance hierarchy.** Avoids the "fragile base class" problem. Each backend owns its control flow and can call utilities in any order.
* **Lower migration barrier.** Existing backends can adopt utilities incrementally - replace one duplicated block at a time without restructuring into a subclass.
* **Easier to understand in isolation.** Each backend's `main()` reads top-to-bottom with explicit function calls. No need to understand a base class and which hooks are called when.
* **More Pythonic.** "Flat is better than nested" and "explicit is better than implicit" (PEP 20). Utility functions align with Python's preference for composition over inheritance.

**Cons:**

* **Does not enforce lifecycle ordering.** The primary goal of the interface is ensuring all backends follow the same lifecycle sequence (runtime before component, component before engine, engine before handler, etc.). Utility functions provide the building blocks but not the ordering contract. A backend could call `register_model()` before `create_component_and_endpoint()` and get a runtime error instead of a compile/lint-time error.
* **Duplication migrates rather than disappears.** The 10-15 line `main()` orchestration sequence (call setup_runtime, then create_component, then create_engine, then setup_metrics, ...) must still be written identically in every backend. This "glue code" is the most error-prone part to duplicate because ordering bugs are silent until runtime.
* **No standardized structure for new backends.** A new backend author must know which utilities to call and in what order. With the ABC, they implement 3 abstract methods and get a working backend. With utilities, they must assemble the lifecycle themselves from documentation.
* **Harder to add cross-cutting features.** Adding a new lifecycle step (e.g., a pre-serve health check, or a graceful drain period before shutdown) requires updating every backend's `main()`. With the ABC, it's added once in `Backend.run()`.
* **No single point for testing the lifecycle.** The ABC's `run()` method can be tested once to verify lifecycle ordering. With utilities, each backend's `main()` must be independently verified for correct ordering.
* **Discoverability.** An ABC with abstract methods makes requirements explicit via the type system. Missing a utility call produces no warning until the missing behavior is observed at runtime.

**Comparison Table:**

| Concern | Interface (ABC) | Utility Functions |
| :---- | :---- | :---- |
| Lifecycle ordering guarantee | Enforced by `run()` | Not enforced, up to each backend |
| Minimum to implement new backend | 3 abstract methods | ~15-20 lines of orchestration calls |
| Adding a new lifecycle step | 1 change in base class | N changes (one per backend) |
| Incremental adoption | Requires restructuring into subclass | Drop-in replacement of individual blocks |
| Control flow visibility | Implicit (must read base class) | Explicit (visible in each `main()`) |
| Risk of ordering bugs | Low (base class enforces) | Medium (manual ordering per backend) |

**Reason Rejected:**

* While utility functions address code duplication for individual operations, they do not solve the core problem: ensuring consistent lifecycle ordering across backends. The orchestration sequence itself is the most duplicated and error-prone code. An ABC with the Template Method pattern provides both deduplication and correctness by construction. Additionally, as Dynamo adds more backends and lifecycle steps over time, the N-way update cost of the utility approach grows, while the ABC approach remains O(1).

**Notes:**

* The proposed `Backend` ABC internally uses utility-style methods for each lifecycle step, so the two approaches are not mutually exclusive. If a backend has a unique lifecycle that doesn't fit the template, it can use the underlying utilities directly (they are importable from `dynamo.backend`). The ABC is the recommended default, not the only option.

## Alt 3: Mixin-Based Approach

**Pros:**

* More flexible composition - backends can pick and choose which mixins to use.
* No rigid lifecycle ordering.

**Cons:**

* Harder to reason about execution order when multiple mixins interact.
* Method resolution order (MRO) in Python can lead to surprising behavior.
* Doesn't enforce a consistent lifecycle, which is the primary goal.

**Reason Rejected:**

* The Template Method pattern provides clearer guarantees about lifecycle ordering and is easier for new backend authors to follow.

## Alt 4: Configuration-Driven Factory

**Pros:**

* Single `GenericBackend` class configured entirely via config/callbacks.
* No subclassing needed.

**Cons:**

* Loses type safety and IDE support for backend-specific methods.
* Callback-heavy APIs are harder to debug and document.
* Complex backends (e.g., TRT-LLM with multiple handler types) don't fit a flat callback model well.

**Reason Rejected:**

* Real backends have enough framework-specific complexity that subclassing provides better ergonomics and maintainability than a callback-driven approach.

## Alt 5: Status Quo

**Pros:**

* No migration effort required.
* Each backend retains full control over its lifecycle.

**Cons:**

* Does not address code duplication.
* Bug fixes must still be applied to each backend independently.
* No standardized structure for new backends.

**Reason Rejected:**

* The maintenance burden and inconsistency will grow as more backends are added and existing ones evolve.

# Background

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **Backend** | A Dynamo worker process that runs an LLM inference engine and serves generation requests |
| **Handler** | The component within a backend that processes individual generation requests |
| **Template Method** | A design pattern where a base class defines an algorithm skeleton, deferring some steps to subclasses |
| **Hook Method** | An optional method in a base class that subclasses can override to customize behavior at specific points |
| **Disaggregated Inference** | Splitting inference into separate prefill and decode phases, potentially on different hardware |
| **Multimodal** | Requests containing multiple input modalities (e.g., text + images) processed together for generation |
| **Diffusion Model** | A generative model that creates images or video from text prompts via iterative denoising |
| **Embedding Model** | A model that maps text inputs to fixed-size vector representations |
| **Modality** | A type of input/output (text, image, video, embedding) that may require a distinct handler and endpoint |

## Acronyms & Abbreviations

**ABC:** Abstract Base Class

**CLI:** Command-Line Interface

**DEP:** Dynamo Enhancement Proposal

**KV:** Key-Value (in context of KV cache events)

**TRT-LLM:** TensorRT Large Language Model (NVIDIA inference framework)

**vLLM:** Virtual Large Language Model (open-source inference framework)

**SGLang:** Structured Generation Language (open-source inference framework)
