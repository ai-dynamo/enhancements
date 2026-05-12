# KV Router Component Summary

## Overview

The KV Router provides intelligent KV cache-aware routing for LLM inference requests. It directs requests to workers with the most relevant cached data while maintaining load balance through worker utilization metrics. The router supports both aggregated serving and disaggregated prefill/decode architectures.

## Location

- **Source**: `components/src/dynamo/router/`
- **Core Implementation**: `lib/llm/src/kv_router/` (Rust)
- **Package**: `ai-dynamo` (part of main package)
- **Current Version**: 0.8.0

## Components That Call Router

The KV Router is a central orchestration component. Here are all components that interact with it:

### Callers (Request Routing)

| Component | File | Interaction |
|-----------|------|-------------|
| **Frontend** | `components/src/dynamo/frontend/main.py` | Primary caller - embeds router for request dispatch |
| **Standalone Router** | `components/src/dynamo/router/__main__.py` | Dedicated routing service using `KvPushRouter` |

### Publishers (KV Events & Metrics)

| Component | File | Events Published |
|-----------|------|------------------|
| **vLLM Worker** | `components/src/dynamo/vllm/publisher.py` | KV block events, worker metrics, cache stats |
| **TRT-LLM Worker** | `components/src/dynamo/trtllm/publisher.py` | `BlockStored`, `BlockRemoved`, `AllBlocksCleared` |
| **SGLang Worker** | `components/src/dynamo/sglang/publisher.py` | KV events via ZMQ, worker metrics |
| **Mocker Worker** | `components/src/dynamo/mocker/` | Mock KV events for testing |

### Configuration Providers

| Component | File | Configuration |
|-----------|------|---------------|
| **Frontend CLI** | `components/src/dynamo/frontend/main.py` | `--router-mode`, `--kv-overlap-score-weight`, etc. |
| **Python Bindings** | `lib/bindings/python/src/dynamo/llm/__init__.py` | Exports `KvRouterConfig`, `RouterConfig`, `RouterMode` |

### Interaction Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                         REQUEST FLOW                                 │
│                                                                      │
│   HTTP Request                                                       │
│        │                                                             │
│        ▼                                                             │
│   ┌─────────┐    KvPushRouter.generate()    ┌──────────────┐        │
│   │Frontend │ ─────────────────────────────►│  KV Router   │        │
│   └─────────┘                               │  (Rust Core) │        │
│                                             └──────┬───────┘        │
│                                                    │                 │
│                         ┌──────────────────────────┼────────────┐   │
│                         ▼                          ▼            ▼   │
│                   ┌──────────┐              ┌──────────┐  ┌────────┐│
│                   │vLLM      │              │TRT-LLM   │  │SGLang  ││
│                   │Worker    │              │Worker    │  │Worker  ││
│                   └────┬─────┘              └────┬─────┘  └───┬────┘│
│                        │                         │            │     │
└────────────────────────┼─────────────────────────┼────────────┼─────┘
                         │                         │            │
┌────────────────────────┼─────────────────────────┼────────────┼─────┐
│                        ▼          EVENT FLOW     ▼            ▼     │
│                   ┌─────────────────────────────────────────────┐   │
│                   │           NATS JetStream                    │   │
│                   │     (KV Events: BlockStored, BlockRemoved)  │   │
│                   └─────────────────────┬───────────────────────┘   │
│                                         │                           │
│                                         ▼                           │
│                                   ┌──────────────┐                  │
│                                   │  KV Router   │                  │
│                                   │  (Indexer)   │                  │
│                                   └──────────────┘                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Router Methods Called

| Method | Caller | Purpose |
|--------|--------|---------|
| `KvPushRouter.generate_from_request()` | Frontend, Standalone Router | Route request and stream response |
| `KvPushRouter.best_worker_id()` | Standalone Router | Query best worker without routing |
| `KvRouterConfig()` | Frontend, Standalone Router | Configure routing behavior |

### Worker Registration Flags

Workers register their type with the router via CLI flags:

| Flag | Worker Types | Effect on Router |
|------|--------------|------------------|
| `--is-prefill-worker` | vLLM, TRT-LLM, SGLang, Mocker | Router knows worker handles prefill only |
| `--is-decode-worker` | vLLM, TRT-LLM, SGLang, Mocker | Router knows worker handles decode only |
| `--enable-local-indexer` | All workers | Worker maintains local KV index |

## Internal Dependencies

### Dynamo Python Modules

| Module | Usage |
|--------|-------|
| `dynamo.llm` | `KvPushRouter`, `KvRouterConfig` - core routing classes |
| `dynamo.runtime` | `DistributedRuntime`, `Client`, `dynamo_worker` decorator |
| `dynamo.runtime.logging` | `configure_dynamo_logging()` - unified logging |

### Dynamo Rust Crates

| Crate | Location | Usage |
|-------|----------|-------|
| `dynamo_llm` | `lib/llm/` | `KvRouter`, `KvPushRouter`, `KvRouterConfig`, indexer, scheduler |
| `dynamo_runtime` | `lib/runtime/` | Distributed runtime, NATS/etcd transport |

**Key Rust modules in `lib/llm/src/kv_router/`:**
- `indexer.rs` - Radix tree for KV cache block tracking
- `scheduler.rs` - Worker selection algorithm
- `prefill_router.rs` - Prefill-specific routing logic
- `publisher.rs` - KV event publishing
- `subscriber.rs` - KV event consumption
- `approx.rs` - Approximate/predictive routing (no KV events)

### External Python Dependencies

| Package | Usage |
|---------|-------|
| `uvloop` | High-performance async event loop |

### External Services

| Service | Required For | Notes |
|---------|--------------|-------|
| NATS JetStream | KV event streaming | State persistence, event replay |
| NATS Object Store | Snapshots | Radix tree state snapshots |
| etcd | Worker discovery | Via DistributedRuntime |

## Public Interface

### Standalone Router Endpoints (via Dynamo Runtime)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `{namespace}.router.generate` | Streaming | Route request to best worker, stream response |
| `{namespace}.router.best_worker_id` | Unary | Get best worker ID without routing |

### StandaloneRouterHandler Class

```python
class StandaloneRouterHandler:
    async def initialize() -> None
    async def generate(request: dict) -> AsyncIterator[dict]
    async def best_worker_id(token_ids: list, router_config_override: dict = None) -> str
```

### KvRouterConfig (from `dynamo.llm`)

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `overlap_score_weight` | float | 1.0 | Weight for KV overlap in worker selection |
| `router_temperature` | float | 0.0 | Softmax temperature (0 = deterministic) |
| `use_kv_events` | bool | True | Use KV events from workers |
| `router_replica_sync` | bool | False | Sync state across router replicas |
| `router_snapshot_threshold` | int | 1000000 | Messages before snapshot |
| `router_reset_states` | bool | False | Clear state on startup |
| `router_track_active_blocks` | bool | True | Track blocks in use |
| `router_ttl_secs` | float | 120.0 | Block TTL (no-events mode) |
| `router_max_tree_size` | int | 2^20 | Max tree size before pruning |
| `router_prune_target_ratio` | float | 0.8 | Target ratio after pruning |

## User/Developer Interaction

### 1. Integrated with Frontend (Primary Use Case)

The router is typically embedded in the frontend, not deployed standalone:

```bash
python -m dynamo.frontend \
    --router-mode kv \
    --kv-overlap-score-weight 1.0 \
    --http-port 8000
```

**Router modes:**
- `kv` - KV cache-aware routing (default when enabled)
- `round-robin` - Simple round-robin
- `random` - Random worker selection

### 2. Standalone Router (Advanced)

For separate prefill routing or custom architectures:

```bash
python -m dynamo.router \
    --endpoint dynamo.prefill.generate \
    --block-size 64 \
    --router-reset-states \
    --no-track-active-blocks
```

### 3. K8s Deployment

**Via Frontend (embedded):**
```yaml
Frontend:
  componentType: frontend
  envs:
    - name: DYN_ROUTER_MODE
      value: kv
```

**Standalone (separate service):**
```yaml
Router:
  componentType: worker  # Note: uses worker type
  extraPodSpec:
    mainContainer:
      command: [python3, -m, dynamo.router]
      args:
        - --endpoint=dynamo.prefill.generate
        - --block-size=64
```

### 4. Programmatic Usage

```python
from dynamo.llm import KvPushRouter, KvRouterConfig

config = KvRouterConfig(
    overlap_score_weight=1.0,
    router_temperature=0.0,
    use_kv_events=True,
)

router = KvPushRouter(
    endpoint=worker_endpoint,
    block_size=64,
    kv_router_config=config,
)

async for output in await router.generate_from_request(request):
    yield output
```

## Packaging & Containers

### Container

- **NOT a separate container** - included in main runtime images
- **Registry**: `nvcr.io/nvidia/ai-dynamo/`
- **Images**: `vllm-runtime`, `tensorrtllm-runtime`, `sglang-runtime`
- **Path in container**: `/workspace/components/src/dynamo/router/`

### Integration Modes

| Mode | Description | Use Case |
|------|-------------|----------|
| Embedded in Frontend | Router runs inside frontend process | Standard deployments |
| Standalone Service | Separate router process | Disaggregated prefill routing |
| Multiple Replicas | Synced via NATS | High availability |

## Service Interface & I/O Contract

### Current Architecture

The router operates in two modes:

**1. Embedded Mode (in Frontend):**
- No separate service interface
- Called internally by frontend request pipeline
- HTTP API exposed via frontend (`/v1/completions`, `/v1/chat/completions`)

**2. Standalone Mode:**
- Exposes Dynamo runtime endpoints (not HTTP)
- Clients must use Dynamo runtime client to call endpoints

```
┌─────────────────────────────────────────────────────────────────┐
│                      STANDALONE ROUTER                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │   Receive    │───►│   Select     │───►│   Forward    │      │
│  │   Request    │    │   Worker     │    │   Request    │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
         ▲                    │                    │
         │                    ▼                    ▼
   ┌─────┴─────┐       ┌──────────────┐    ┌──────────────┐
   │  Dynamo   │       │NATS JetStream│    │   Workers    │
   │  Runtime  │       │  (KV events) │    │  (backends)  │
   └───────────┘       └──────────────┘    └──────────────┘
```

### Inputs

| Input | Source | Format | Notes |
|-------|--------|--------|-------|
| **CLI Arguments** | Startup | argparse | Block size, weights, modes |
| **Requests** | Dynamo runtime | Dict with `token_ids` | Pre-tokenized requests |
| **KV Events** | NATS JetStream | Protobuf/JSON | Block create/delete events |
| **Worker State** | etcd | KV watch | Worker instance discovery |

### Outputs

| Output | Destination | Format | Notes |
|--------|-------------|--------|-------|
| **Routed Requests** | Workers | Dynamo runtime | Forwarded to selected worker |
| **Responses** | Caller | Streaming dict | `LLMEngineOutput` format |
| **KV Events** | NATS (replica sync) | Protobuf/JSON | When `--router-replica-sync` |
| **Logs** | stdout/stderr | Structured logs | Via `dynamo.runtime.logging` |

### What's Missing (No External API)

Currently there is **no way to**:
- Query router state via REST/HTTP (radix tree, worker scores)
- Get routing statistics/history
- Manually override routing decisions
- Dynamically update routing weights
- Health check the router independently (when embedded)

### 1.0 Standardization Recommendations

#### Option A: Add HTTP API for Standalone Router

```yaml
# Proposed endpoints
GET  /health                    # Health check
GET  /status                    # Router state (tree size, workers, etc.)
GET  /workers                   # Connected workers and scores
GET  /config                    # Current configuration
PUT  /config                    # Update weights dynamically
GET  /metrics                   # Prometheus metrics
POST /route                     # Manual routing query (debug)
```

#### Option B: Standardize Embedded Router Observability

If router remains embedded:
1. **Frontend endpoints** for router status (`/router/status`)
2. **Prometheus metrics** for routing decisions
3. **Structured logging** for routing traces

### Contract Versioning Needs

| Component | Current | 1.0 Requirement |
|-----------|---------|-----------------|
| CLI args | Documented in README | Versioned schema |
| Request format | Implicit (`token_ids` required) | JSON schema |
| KV event format | Protobuf in Rust | Documented, versioned |
| NATS stream names | Hardcoded | Configurable, documented |
| Worker selection algorithm | Internal | Document scoring formula |

## Observability

### Prometheus Metrics

Router metrics are exposed via the frontend when embedded. Key metrics include:
- Request routing latency
- Worker selection distribution
- KV cache hit rates
- Tree size and pruning events

### Logging

Uses `dynamo.runtime.logging` with structured output:
- Routing decisions
- Worker scores
- KV event processing
- State synchronization

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| Python exports | Minimal (`__version__` only) | Document public API |
| CLI entry point | `python -m dynamo.router` | Standardize, document args |
| Config schema | `KvRouterConfig` class | Versioned, validated |
| Request format | Implicit | JSON schema, validation |
| KV event format | Rust protobuf | Document wire format |
| NATS integration | Hardcoded streams | Configurable, documented |
| Error handling | Basic exceptions | Standardize error codes |
| Versioning | Part of `ai-dynamo` package | Consider independent versioning |

## Customization & Extension

### Extension Points

#### 1. Custom Router Mode

Currently hardcoded modes (`kv`, `round-robin`, `random`). Extension would require:
- Modifying `RouterMode` enum in Rust
- Implementing new scheduling strategy in `scheduler.rs`

#### 2. Custom Scoring Function

The worker selection scoring is in Rust (`scheduler.rs`). To customize:
- Fork `lib/llm/src/kv_router/scheduler.rs`
- Modify `calculate_worker_score()` function
- Rebuild Python bindings

#### 3. Configuration Tuning

**CLI Arguments:**
```bash
python -m dynamo.router \
  --kv-overlap-score-weight 0.5 \  # Balance KV reuse vs load
  --router-temperature 0.1 \        # Add randomness
  --router-ttl 60 \                 # Shorter TTL for dynamic workloads
  --router-max-tree-size 500000     # Smaller tree for memory
```

**Environment Variables:**
| Variable | Description |
|----------|-------------|
| `NATS_SERVER` | NATS server URL |
| `DYN_REQUEST_PLANE` | Request transport (tcp/http/nats) |
| `DYN_ROUTER_MODE` | Router mode for frontend |

### Current Limitations (Not Easily Extensible)

| Area | Limitation | Workaround |
|------|------------|------------|
| Scoring algorithm | Rust implementation | Fork and modify |
| Event format | Protobuf in Rust | Match existing format |
| Tree structure | Radix tree hardcoded | Cannot change |
| Transport | NATS required for events | Use `--no-kv-events` |

## Related Documentation

- [KV Cache Routing](docs/router/kv_cache_routing.md) - Detailed routing architecture
- [Frontend README](components/src/dynamo/frontend/README.md) - Embedded router usage
- [Router Benchmarking](benchmarks/router/README.md) - Performance testing

## Tests

- E2E tests: `tests/router/test_router_e2e_with_*.py`
- Common utilities: `tests/router/common.py`
- Backend-specific: vLLM, SGLang, TRT-LLM, Mockers
