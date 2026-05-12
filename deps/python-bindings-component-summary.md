# Python Bindings Component Summary

## Overview

The Python Bindings provide a Python interface to Dynamo's Rust core libraries (`dynamo-llm` and `dynamo-runtime`). They enable Python-based workers and applications to leverage Dynamo's distributed computing capabilities, including the distributed runtime, KV routing, model discovery, preprocessing, and metrics collection.

## Location

- **Source**: `lib/bindings/python/`
- **Rust Code**: `lib/bindings/python/rust/`
- **Python Package**: `dynamo` (via PyO3/Maturin)
- **Current Version**: 0.8.0

## Internal Dependencies

### Rust Crates (Wrapped)

| Crate | Location | Provides |
|-------|----------|----------|
| `dynamo-llm` | `lib/llm/` | LLM-specific functionality (KV router, preprocessor, model cards) |
| `dynamo-runtime` | `lib/runtime/` | Distributed runtime, transports, discovery |

### Build Dependencies

| Tool | Purpose |
|------|---------|
| Maturin | Rust-Python binding build tool |
| PyO3 | Rust-Python FFI |
| pyo3-async-runtimes | Async Python-Rust bridge |

## Public Interface

### Core Modules Exposed

```python
import dynamo
from dynamo.runtime import DistributedRuntime, dynamo_worker
from dynamo.llm import (
    KvPushRouter,
    KvRouterConfig,
    ModelInput,
    ModelType,
    register_llm,
    fetch_llm,
)
```

### `dynamo.runtime` Module

| Class/Function | Description |
|----------------|-------------|
| `DistributedRuntime` | Main runtime class for distributed coordination |
| `dynamo_worker()` | Decorator for worker entry points |
| `Client` | Client for calling remote endpoints |
| `Endpoint` | Endpoint for serving requests |
| `Component` | Component registration |
| `Namespace` | Namespace scoping |

### `dynamo.llm` Module

| Class/Function | Description |
|----------------|-------------|
| `KvPushRouter` | KV-aware request router |
| `KvRouterConfig` | Router configuration |
| `ModelInput` | Enum (Tokens, Text) |
| `ModelType` | Enum (Chat, Completions, Prefill, etc.) |
| `ModelDeploymentCard` | Model metadata |
| `ModelRuntimeConfig` | Runtime configuration |
| `register_llm()` | Register a model with discovery |
| `fetch_llm()` | Download model from hub |
| `ZmqKvEventPublisher` | KV event publishing |
| `PythonAsyncEngine` | Python engine wrapper |

### `dynamo._core` Module (Low-level)

| Class | Description |
|-------|-------------|
| `VirtualConnectorCoordinator` | Planner coordination via etcd |
| `VirtualConnectorClient` | Client for planner coordination |
| `PlannerDecision` | Scaling decision data |

### Rust Binding Files

| File | Provides |
|------|----------|
| `lib.rs` | Main PyO3 module definition |
| `engine.rs` | Engine factory bindings |
| `llm/kv.rs` | KvPushRouter, KV event handling |
| `llm/entrypoint.rs` | Model registration, preprocessor |
| `planner.rs` | Planner coordination |
| `prometheus_metrics.rs` | Metrics collection |
| `http.rs` | HTTP server bindings |
| `context.rs` | Runtime context |

## User/Developer Interaction

### 1. Basic Worker Setup

```python
from dynamo.runtime import DistributedRuntime, dynamo_worker

@dynamo_worker()
async def worker(runtime: DistributedRuntime):
    component = runtime.namespace("my-ns").component("my-component")
    endpoint = component.endpoint("generate")

    async def handler(request):
        yield {"result": "hello"}

    await endpoint.serve_endpoint(handler)
```

### 2. Model Registration

```python
from dynamo.llm import register_llm, ModelInput, ModelType

await register_llm(
    model_input=ModelInput.Tokens,
    model_type=ModelType.Chat | ModelType.Completions,
    endpoint=endpoint,
    model_name="Qwen/Qwen3-0.6B",
    block_size=64,
)
```

### 3. Client Usage

```python
client = await endpoint.client()
async for response in client.generate(request):
    print(response)
```

## Packaging & Containers

### Build Process

```bash
# Development build
cd lib/bindings/python
uv venv && source .venv/bin/activate
uv pip install maturin
maturin develop --uv

# Release build
maturin build --release
```

### Package Structure

```
dynamo/
├── __init__.py
├── _core.so          # Compiled Rust bindings
├── runtime/          # Runtime submodule
├── llm/              # LLM submodule
├── common/           # Common utilities
└── ...
```

### Container Integration

- Bindings are pre-built in all runtime containers
- Path: `/workspace/lib/bindings/python/`

## Service Interface & I/O Contract

### Python-Rust Boundary

| Direction | Format | Notes |
|-----------|--------|-------|
| Python → Rust | PyO3 objects | Automatic conversion |
| Rust → Python | PyO3 objects | Automatic conversion |
| Async | pyo3-async-runtimes | tokio ↔ asyncio bridge |

### Performance Considerations

From README:
> The Python GIL is a global critical section and is ultimately the death of parallelism. If bouncing many small messages back-and-forth between Python and Rust event loops where Rust requires GIL access, moving the code from Python to Rust will give significant gains.

## Observability

### Prometheus Metrics

`prometheus_metrics.rs` provides:
- Custom metrics registration
- Metrics callback integration
- Counter, Gauge, Histogram support

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| Module structure | `dynamo.*` hierarchy | Document public API |
| Type hints | Partial | Complete type stubs |
| Documentation | README only | API documentation |
| Error handling | PyO3 exceptions | Standardize error types |
| Versioning | Matches `ai-dynamo` | Consider independent version |

## Customization & Extension

### Extension Points

1. **Custom Engines** - Implement `PythonAsyncEngine`
2. **Custom Handlers** - Use `endpoint.serve_endpoint()`
3. **Custom Metrics** - Use metrics callback registration

### Adding New Bindings

1. Add Rust code in `rust/` directory
2. Expose via `lib.rs` PyO3 module
3. Add Python wrapper if needed in `src/dynamo/`

## Related Documentation

- [Runtime Guide](docs/development/runtime-guide.md)
- [Backend Guide](docs/development/backend-guide.md)
- Examples: `lib/bindings/python/examples/`

## Tests

- Python tests: `lib/bindings/python/tests/`
- Examples: `lib/bindings/python/examples/hello_world/`
