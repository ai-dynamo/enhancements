# Dynamo Python Bindings API Formalization

**Status**: Draft

**Authors**: @gkiar, @nnshah1

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**:
- https://github.com/ai-dynamo/dynamo/pull/5412
- https://github.com/ai-dynamo/dynamo/pull/5458

---

## Summary

This DEP defines the public API surface for Dynamo's Python bindings, establishes stability guarantees, and proposes reorganization to minimize the public interface while maintaining usability.

---

## Motivation

The current Python bindings expose approximately 65 classes and functions across `dynamo.runtime`, `dynamo.llm`, `dynamo.nixl_connect`, and `dynamo.logits_processing` modules. This creates several challenges:

1. **Large surface area**: More to document, test, and maintain
2. **No clear organization**: Classes with overlapping purposes (e.g., 3 indexer variants, 4 publisher variants)
3. **Internal exposure**: Implementation details like `Namespace`, `Component`, `CancellationToken` are exposed
4. **Evolved organically**: Interfaces weren't designed for external consumption

### Current Inventory

| Category | Classes/Functions | Used By |
|----------|------------------|---------|
| Core Runtime | DistributedRuntime, Namespace, Component, Endpoint, Client, CancellationToken, Context | All |
| Model Registration | register_llm, unregister_llm, fetch_llm, ModelType, ModelInput, ModelDeploymentCard, ModelRuntimeConfig | Backends |
| Router Config | RouterMode, RouterConfig, KvRouterConfig | Frontend |
| KV Indexing | KvIndexer, ApproxKvIndexer, RadixTree, OverlapScores | Router |
| KV Publishing | KvEventPublisher, ZmqKvEventPublisher, ZmqKvEventPublisherConfig, ZmqKvEventListener | Backends |
| KV Router | KvPushRouter, KvPushRouterStream | Router |
| Worker Metrics | WorkerMetricsPublisher, ForwardPassMetrics, WorkerStats, KvStats, SpecDecodeStats | Backends |
| HTTP/gRPC | HttpService, HttpAsyncEngine, KserveGrpcService, PythonAsyncEngine | Frontend |
| Preprocessor | OAIChatPreprocessor, MediaDecoder, MediaFetcher, Backend | Frontend |
| Entrypoint | EntrypointArgs, EngineConfig, EngineType, make_engine, run_input | Frontend |
| Prometheus | RuntimeMetrics, Counter, IntCounter, CounterVec, IntCounterVec, Gauge, IntGauge, GaugeVec, IntGaugeVec, Histogram | All |
| Block Mgmt | Layer, Block, BlockList, BlockManager, KvbmCacheManager, KvbmRequest | KVBM |
| Planner | VirtualConnectorCoordinator, VirtualConnectorClient, PlannerDecision | Planner |
| LoRA | LoRADownloader, lora_name_to_id | Backends |
| Misc | KvRecorder, compute_block_hash_for_seq_py, log_message | Various |

---

## Goals

1. Minimize the public interface for maintainability
2. Organize the public interface intuitively
3. Clearly separate public vs internal APIs
4. Establish stability guarantees for public APIs
5. Provide type stubs for IDE support

### Non-Goals

1. Complete rewrite of bindings implementation
2. Breaking compatibility for existing users immediately
3. Removing functionality (just reorganizing)

---

## Proposal

### 1. Move Internal Bindings to `dynamo._internal`

The following should NOT be in the public API:
- `Namespace` - internal runtime concept
- `Component` - internal runtime concept
- `CancellationToken` - internal async handling
- `Context` - internal context passing
- `ModelDeploymentCard` - internal model metadata
- `ModelRuntimeConfig` - internal configuration

**Action**: Change public API to not need direct access to these, then move them to `dynamo._internal` module.

**Question**: Can we merge creating an endpoint with serving it to hide `Endpoint`?

### 2. Simplify KV Router Interfaces

Current state: 15+ classes with confusing overlaps
- 3 indexer variants: `KvIndexer`, `ApproxKvIndexer`, `RadixTree`
- 4 publisher variants: `KvEventPublisher`, `ZmqKvEventPublisher`, `ZmqKvEventPublisherConfig`, `ZmqKvEventListener`
- Multiple stats classes: `ForwardPassMetrics`, `WorkerStats`, `KvStats`, `SpecDecodeStats`

**Proposed**: Create unified interfaces:
- `KvRouter` - internally uses KvIndexer or ApproxKvIndexer based on mode
- `KvPublisher` - internally uses ZmqKvEventPublisher or KvEventPublisher based on source

**Action**: Needs design input from KV router team.

### 3. Reorganize Prometheus Metrics

Current issues:
- Every metric type has an Int variant (may be unnecessary for Python)
- Multiple entry points for creating metrics

**Proposed**:
- Single entry point: `Metrics.counter(...)`, `Metrics.gauge(...)`
- Move to new `dynamo.metrics` module
- Consider removing Int variants (Python handles int↔float seamlessly)

### 4. Design Multimodal Interface

Current state: Evolving, not bound by v1 stability promise.

**Action**: Design the interface intentionally rather than letting it happen organically. Ideally a single type that interfaces to all things multi-modal.

### 5. Simplify `register_llm`

Current signature has many parameters:

```python
async def register_llm(
    model_input: ModelInput,
    model_type: ModelType,
    endpoint: Endpoint,
    model_path: str,
    model_name: Optional[str] = None,
    context_length: Optional[int] = None,
    kv_cache_block_size: Optional[int] = None,
    router_mode: Optional[RouterMode] = None,
    migration_limit: int = 0,
    runtime_config: Optional[ModelRuntimeConfig] = None,
    user_data: Optional[Dict[str, Any]] = None,
    custom_template_path: Optional[str] = None,
    lora_name: Optional[str] = None,
    base_model_path: Optional[str] = None,
) -> None
```

**Action**: Consider builder pattern or config object to reduce parameter count.

---

## Proposed Module Structure

```python
# Public API
dynamo/
├── runtime/           # Core runtime (public)
│   ├── DistributedRuntime
│   ├── Endpoint
│   ├── Client
│   └── dynamo_worker()
├── llm/               # LLM functionality (public)
│   ├── register_llm
│   ├── unregister_llm
│   ├── fetch_llm
│   ├── ModelType
│   ├── ModelInput
│   └── KvRouter       # Unified router interface
├── metrics/           # Prometheus metrics (public)
│   ├── counter()
│   ├── gauge()
│   └── histogram()
├── _internal/         # Internal (not stable)
│   ├── Namespace
│   ├── Component
│   ├── Context
│   ├── CancellationToken
│   ├── ModelDeploymentCard
│   └── ...
└── _core/             # Low-level bindings (not stable)
    └── ...
```

---

## Stability Guarantees

### Stable (v1.0+)

APIs under `dynamo.runtime`, `dynamo.llm`, `dynamo.metrics`:
- Semantic versioning applies
- Deprecation warnings before removal
- At least one minor version with deprecation before removal

### Unstable (Experimental)

APIs under `dynamo._internal`, `dynamo._core`:
- May change without notice
- Not recommended for external use
- No backwards compatibility guarantee

### Multimodal

Current multimodal APIs:
- Explicitly marked as experimental
- Will stabilize based on usage patterns

---

## Implementation Phases

### Phase 1: Remove Deprecated/Unused Features

**PRs**: #5412, #5458

**Status**: In Progress

**Action**: Identify more removals (challenging - we don't know what people are using externally).

### Phase 2: Create `dynamo._internal` Module

- Move internal types
- Update public API to not require direct access
- Add deprecation warnings for direct imports

### Phase 3: Unify KV Router Interfaces

- Design unified `KvRouter` and `KvPublisher`
- Deprecate individual variants
- Requires input from KV router team

### Phase 4: Reorganize Metrics

- Create `dynamo.metrics` module
- Consolidate metric types
- Consider removing Int variants

### Phase 5: Type Stubs

- Generate `.pyi` files for all public modules
- Integrate with mypy/pyright
- Publish to typeshed or package

---

## Type Safety Requirements

### Pydantic Models

For data classes that cross component boundaries:

```python
from pydantic import BaseModel, Field

class WorkerStats(BaseModel):
    request_active_slots: int
    request_total_slots: int
    num_requests_waiting: int = 0
    data_parallel_rank: Optional[int] = None

    class Config:
        extra = "forbid"  # Reject unknown fields
```

### Type Stubs

Generate and maintain `.pyi` files:

```python
# dynamo/llm/__init__.pyi
from typing import Optional
from enum import Enum

class ModelType(Enum):
    Chat: int
    Completions: int
    Embedding: int
    Tensor: int

class ModelInput(Enum):
    Tokens: int
    Text: int

async def register_llm(
    model_input: ModelInput,
    model_type: ModelType,
    endpoint: Endpoint,
    model_path: str,
    *,
    model_name: Optional[str] = None,
    context_length: Optional[int] = None,
    ...
) -> None: ...
```

---

## Related Proposals

- [DEP-0000: Dynamo 1.0 API and Non-Feature Requirements](0000-dynamo-api-and-non-feature-requirements.md)
- [DEP-0000: Backend-Frontend Contract Formalization](0000-backend-frontend-contract.md)
- [DEP-0000: Protocol Compatibility](0000-protocol-compatibility.md)

---

## Alternate Solutions

### Alt 1: Code Generation from Schemas

Generate Python bindings from JSON Schema or protobuf definitions.

**Pros**:
- Single source of truth
- Automatic type safety
- Documentation generation

**Cons**:
- Requires tooling investment
- May not fit all use cases

**Reason Deferred**: Good long-term approach, can adopt post-1.0.

### Alt 2: Separate Python Package

Create a separate `dynamo-python` package with pure Python implementations.

**Pros**:
- Independent versioning
- Easier testing

**Cons**:
- Duplicates functionality
- Sync overhead

**Reason Rejected**: PyO3 bindings are working well.

---

## Background

### Working Document

See `proposal-bindings-v2.md` for the original working document with detailed current state analysis.

### References

- [PyO3 User Guide](https://pyo3.rs/)
- [Python Packaging User Guide](https://packaging.python.org/)
- [PEP 561 - Distributing and Packaging Type Information](https://peps.python.org/pep-0561/)
