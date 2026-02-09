# Frontend Component Summary

## Overview

The Frontend is Dynamo's HTTP entry point providing OpenAI-compatible API endpoints for LLM inference. It bundles an HTTP server, request preprocessor (tokenization, prompt templating), and router (round-robin, random, or KV-aware) in a single process. Workers are auto-discovered via etcd when they call `register_llm`.

## Location

- **Source**: `components/src/dynamo/frontend/`
- **Package**: `ai-dynamo` (part of main package)
- **Current Version**: 0.8.0

## Internal Dependencies

### Dynamo Python Modules

| Module | Usage |
|--------|-------|
| `dynamo.llm` | `EntrypointArgs`, `KvRouterConfig`, `RouterConfig`, `RouterMode`, `make_engine`, `PythonAsyncEngine` |
| `dynamo.runtime` | `DistributedRuntime` |
| `dynamo.runtime.logging` | `configure_dynamo_logging()` |
| `dynamo.common.config_dump` | Configuration dumping utilities |

### Dynamo Rust Crates (via Python Bindings)

| Crate | Location | Usage |
|-------|----------|-------|
| `dynamo_llm` | `lib/llm/` | HTTP server, preprocessor, router, model discovery |
| `dynamo_runtime` | `lib/runtime/` | Distributed runtime, etcd/NATS transport |

**Key Rust modules:**
- `lib/llm/src/http/` - HTTP server implementation
- `lib/llm/src/preprocessor/` - Tokenization, prompt templating
- `lib/llm/src/kv_router/` - KV-aware routing
- `lib/llm/src/entrypoint/` - Engine factory and lifecycle

### External Python Dependencies

| Package | Usage |
|---------|-------|
| `uvloop` | High-performance async event loop |

### External Services

| Service | Required For | Notes |
|---------|--------------|-------|
| etcd | Worker discovery | Model registration via `register_llm` |
| NATS JetStream | KV events | Optional, for KV-aware routing |

## Public Interface

### HTTP Endpoints (OpenAI-Compatible)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/completions` | POST | Text completion |
| `/v1/chat/completions` | POST | Chat completion |
| `/v1/models` | GET | List available models |
| `/health` | GET | Health check |
| `/metrics` | GET | Prometheus metrics |
| `/busy_threshold` | PUT | Dynamic busy threshold update |

### CLI Arguments

See [DEP: Frontend CLI Formalization](0000-frontend-cli-formalization.md) for complete analysis.

**Server Configuration:**

| Argument | Default | Env Var | Notes |
|----------|---------|---------|-------|
| `--http-host` | `0.0.0.0` | `DYN_HTTP_HOST` | Listen address |
| `--http-port` | `8000` | `DYN_HTTP_PORT` | Listen port |
| `--tls-cert-path` | None | `DYN_TLS_CERT_PATH` | TLS certificate |
| `--tls-key-path` | None | `DYN_TLS_KEY_PATH` | TLS key (mutual with cert) |

**Model Configuration:**

| Argument | Default | Env Var | Notes |
|----------|---------|---------|-------|
| `--model-name` | None | - | Model identifier |
| `--model-path` | None | - | Local model directory |
| `--namespace` | None | `DYN_NAMESPACE` | Discovery namespace |

**Router Configuration:**

| Argument | Default | Env Var | Notes |
|----------|---------|---------|-------|
| `--router-mode` | `round-robin` | `DYN_ROUTER_MODE` | `round-robin`, `random`, `kv` |
| `--kv-cache-block-size` | None | `DYN_KV_CACHE_BLOCK_SIZE` | KV mode only |
| `--kv-overlap-score-weight` | `1.0` | `DYN_KV_OVERLAP_SCORE_WEIGHT` | Cache reuse weight |
| `--router-temperature` | `0.0` | `DYN_ROUTER_TEMPERATURE` | Selection randomness (0-1) |
| `--kv-events/--no-kv-events` | True | `DYN_KV_EVENTS` | Event tracking |
| `--router-ttl` | `120.0` | `DYN_ROUTER_TTL` | Block TTL (no-events mode) |
| `--router-replica-sync` | False | `DYN_ROUTER_REPLICA_SYNC` | Multi-replica sync |
| `--router-snapshot-threshold` | 1000000 | `DYN_ROUTER_SNAPSHOT_THRESHOLD` | Snapshot interval |
| `--router-reset-states` | False | `DYN_ROUTER_RESET_STATES` | Reset on startup |
| `--track-active-blocks` | True | `DYN_ROUTER_TRACK_ACTIVE_BLOCKS` | Track active blocks |

**Worker Load Detection:**

| Argument | Default | Env Var | Notes |
|----------|---------|---------|-------|
| `--active-decode-blocks-threshold` | None | `DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD` | Busy threshold (0-1) |
| `--active-prefill-tokens-threshold` | None | `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD` | Token threshold |

**Metrics & Observability:**

| Argument | Default | Env Var | Notes |
|----------|---------|---------|-------|
| `--metrics-prefix` | None | `DYN_METRICS_PREFIX` | Metrics name prefix |
| `--kserve-grpc-server` | False | `DYN_KSERVE_GRPC_SERVER` | Enable gRPC server |
| `--grpc-metrics-port` | `8788` | `DYN_GRPC_METRICS_PORT` | gRPC metrics port |
| `--custom-backend-metrics-endpoint` | `nim.backend.runtime_stats` | `CUSTOM_BACKEND_ENDPOINT` | Backend metrics |
| `--custom-backend-metrics-polling-interval` | `0` | `CUSTOM_BACKEND_METRICS_POLLING_INTERVAL` | Poll interval |

**Backend Configuration:**

| Argument | Default | Env Var | Notes |
|----------|---------|---------|-------|
| `--store-kv` | `etcd` | `DYN_STORE_KV` | `etcd`, `file`, `mem` |
| `--request-plane` | `tcp` | `DYN_REQUEST_PLANE` | `tcp`, `http`, `nats` |

**General:**

| Argument | Default | Notes |
|----------|---------|-------|
| `--interactive`, `-i` | False | Interactive mode |
| `--exp-python-factory` | False | [EXPERIMENTAL] Python engine |
| `--dump-config-to` | None | Dump configuration |

## User/Developer Interaction

### 1. Standalone Deployment (Primary Use Case)

```bash
python -m dynamo.frontend \
    --http-port 8000 \
    --router-mode kv \
    --namespace my-deployment
```

### 2. K8s Deployment (via DGD)

```yaml
Frontend:
  dynamoNamespace: my-deployment
  componentType: frontend
  replicas: 1
  extraPodSpec:
    mainContainer:
      image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1
  envs:
    - name: DYN_ROUTER_MODE
      value: kv
    - name: DYN_HTTP_PORT
      value: "8000"
```

### 3. Interactive Mode (Development)

```bash
python -m dynamo.frontend --interactive
```

### 4. Client Usage

```python
import openai

client = openai.OpenAI(base_url="http://localhost:8000/v1", api_key="dummy")
response = client.chat.completions.create(
    model="Qwen/Qwen3-0.6B",
    messages=[{"role": "user", "content": "Hello!"}],
)
```

## Packaging & Containers

### Container

- **NOT a separate container** - included in main runtime images
- **Registry**: `nvcr.io/nvidia/ai-dynamo/`
- **Images**: `vllm-runtime`, `tensorrtllm-runtime`, `sglang-runtime`
- **Path in container**: `/workspace/components/src/dynamo/frontend/`

### K8s Operator Support

- `componentType: frontend` - Recognized by Dynamo operator
- Auto-configures service ports (8000 for HTTP)
- Supports horizontal scaling (multiple replicas with router-replica-sync)

## Service Interface & I/O Contract

### Current Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         FRONTEND                                │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │    HTTP      │───►│ Preprocessor │───►│    Router    │      │
│  │   Server     │    │  (tokenize)  │    │  (select)    │      │
│  └──────────────┘    └──────────────┘    └──────────────┘      │
└─────────────────────────────────────────────────────────────────┘
         ▲                                         │
         │                                         ▼
   ┌─────┴─────┐                           ┌──────────────┐
   │  Clients  │                           │   Workers    │
   │  (HTTP)   │                           │  (backends)  │
   └───────────┘                           └──────────────┘
```

### Inputs

| Input | Source | Format | Notes |
|-------|--------|--------|-------|
| **HTTP Requests** | Clients | OpenAI API format | `/v1/completions`, `/v1/chat/completions` |
| **CLI Arguments** | Startup | argparse | Server, router, backend config |
| **Environment Vars** | Runtime | Key-value | `DYN_*` variables |
| **Model Discovery** | etcd | MDC (Model Deployment Card) | Auto-discovered workers |

### Outputs

| Output | Destination | Format | Notes |
|--------|-------------|--------|-------|
| **HTTP Responses** | Clients | OpenAI API format | JSON, SSE for streaming |
| **Prometheus Metrics** | `/metrics` | Prometheus format | `dynamo_frontend_*` prefix |
| **Logs** | stdout/stderr | Structured logs | Via `dynamo.runtime.logging` |

### API Versioning

Current API is OpenAI-compatible. Key considerations for 1.0:
- Document supported OpenAI API version
- Define Dynamo-specific extensions (headers, parameters)
- Version custom endpoints (`/busy_threshold`, etc.)

## Observability

### Prometheus Metrics (prefix: `dynamo_frontend_`)

| Metric | Type | Description |
|--------|------|-------------|
| `dynamo_frontend_request_total` | Counter | Total requests |
| `dynamo_frontend_request_duration_seconds` | Histogram | Request latency |
| `dynamo_frontend_tokens_total` | Counter | Tokens processed |
| `dynamo_frontend_active_requests` | Gauge | In-flight requests |

### Health Check

- `GET /health` - Returns 200 if healthy

### Logging

Structured logging via `dynamo.runtime.logging`:
- Request lifecycle
- Model discovery events
- Routing decisions

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| OpenAI compatibility | Partial implementation | Document supported endpoints/versions |
| CLI args | Many options | Versioned schema, document all |
| Environment vars | `DYN_*` prefix | Document all, deprecation policy |
| Metrics | `dynamo_frontend_*` | Document all metrics, labels |
| Error responses | Basic | Standardize error codes/messages |
| API extensions | Ad-hoc | Document Dynamo-specific headers |

---

## CLI Formalization Requirements

See [DEP: Frontend CLI Formalization](0000-frontend-cli-formalization.md) for complete details.

### Critical Issues (31 arguments)

| Category | Issue | Impact |
|----------|-------|--------|
| **Missing Validation** | 15+ args lack range/bounds checks | Silent failures, runtime errors |
| **Missing Env Vars** | 10+ args without `DYN_*` support | K8s deployments limited |
| **Inconsistent Booleans** | Mix of `store_true` and `BooleanOptionalAction` | UX confusion |
| **Implicit Dependencies** | KV args accepted in non-KV modes | Invalid configs |
| **No Argument Groups** | 31 args in flat list | Poor discoverability |

### Validation Requirements

| Type | Arguments | Validation |
|------|-----------|------------|
| Port range | `--http-port`, `--grpc-metrics-port` | 1-65535 |
| Float [0,1] | `--router-temperature`, `--router-prune-target-ratio`, `--active-decode-blocks-threshold` | 0.0-1.0 |
| Positive | `--router-ttl`, `--router-max-tree-size`, `--router-snapshot-threshold`, `--kv-cache-block-size` | >0 |
| Non-negative | `--kv-overlap-score-weight`, `--custom-backend-metrics-polling-interval` | ≥0 |
| Mode check | 10+ KV router args | Only with `--router-mode=kv` |

### Missing Environment Variables

| Argument | Proposed Env Var |
|----------|------------------|
| `--tls-cert-path` | `DYN_TLS_CERT_PATH` |
| `--tls-key-path` | `DYN_TLS_KEY_PATH` |
| `--router-replica-sync` | `DYN_ROUTER_REPLICA_SYNC` |
| `--router-snapshot-threshold` | `DYN_ROUTER_SNAPSHOT_THRESHOLD` |
| `--router-reset-states` | `DYN_ROUTER_RESET_STATES` |
| `--track-active-blocks` | `DYN_ROUTER_TRACK_ACTIVE_BLOCKS` |
| `--active-decode-blocks-threshold` | `DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD` |
| `--active-prefill-tokens-threshold` | `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD` |
| `--kserve-grpc-server` | `DYN_KSERVE_GRPC_SERVER` |
| `--grpc-metrics-port` | `DYN_GRPC_METRICS_PORT` |

### Proposed Argument Groups

```
HTTP Server (4 args)
Model Configuration (3 args)
Request Routing (1 arg)
KV Router Options (3 args) - requires --router-mode=kv
Router Tuning (7 args)
Worker Load Detection (2 args)
Metrics & Observability (5 args)
Backend Configuration (2 args)
Experimental (1 arg)
General (3 args)
```

### Implementation Priority

1. **Phase 1 (Critical)**: Add validation, mode dependency checks
2. **Phase 2**: Add missing environment variables
3. **Phase 3**: Standardize booleans, add argument groups
4. **Phase 4**: Documentation generation from argparse

---

## Backend Contract Enforcement

See [DEP: Backend-Frontend Contract](0000-backend-frontend-contract.md) for complete details.

### Contract Enforcement Layers

| Layer | Location | Enforcement | Coverage |
|-------|----------|-------------|----------|
| HTTP Validation | `lib/llm/src/http/service/validate.rs` | 25+ validators | **Strong** |
| Protocol (Serde) | `lib/llm/src/protocols/` | Type deserialization | Partial |
| Python Bridge | `handlers.py` | `Dict[str, Any]` | **None** |
| Response | `BackendOutput` | Structure only | Minimal |

### Critical Gap: Python Bridge

```
Frontend (Rust)                    Backend (Python)
PreprocessedRequest  ──────────►  Dict[str, Any]
(typed struct)                    (no type safety)
```

**Current code** (`handlers.py`):
```python
for key, value in request["sampling_options"].items():
    if hasattr(sampling_params, key):
        setattr(sampling_params, key, value)  # NO TYPE CHECK
```

### What IS Validated (Rust HTTP Layer)

| Parameter | Validation | Range |
|-----------|------------|-------|
| temperature | `validate_temperature()` | 0.0 - 2.0 |
| top_p | `validate_top_p()` | 0.0 - 1.0 |
| top_k | `validate_top_k()` | -1 or ≥1 |
| frequency_penalty | `validate_frequency_penalty()` | -2.0 - 2.0 |
| presence_penalty | `validate_presence_penalty()` | -2.0 - 2.0 |
| n | `validate_n()` | 1 - 128 |
| stop | `validate_stop()` | max 4, non-empty |
| messages | `validate_messages()` | non-empty array |
| unknown fields | `validate_no_unsupported_fields()` | rejected |

### What is NOT Validated

| Gap | Impact |
|-----|--------|
| Python receives `Dict[str, Any]` | Type mismatches propagate silently |
| No capability negotiation | Wrong backend can claim model support |
| Response token IDs not validated | Invalid IDs can panic |
| No version negotiation | Future incompatibility undetected |

### Request/Response Schema Locations

| Schema | Location |
|--------|----------|
| Chat Completions | `lib/llm/src/protocols/openai/chat_completions.rs` |
| Common Extensions | `lib/llm/src/protocols/openai/common_ext.rs` |
| NVIDIA Extensions | `lib/llm/src/protocols/openai/nvext.rs` |
| Backend Output | `lib/llm/src/protocols/common/llm_backend.rs` |
| KServe Proto | `lib/llm/src/grpc/protos/kserve.proto` |

### Formalization Requirements

1. **Add Pydantic models** to Python handlers for type safety
2. **Add capability declaration** to worker registration
3. **Add response validation** for semantic correctness
4. **Add protocol versioning** headers

## Customization & Extension

### Extension Points

#### 1. Custom Backend Metrics

```bash
python -m dynamo.frontend \
    --custom-backend-metrics-endpoint nim.backend.runtime_stats \
    --custom-backend-metrics-polling-interval 9.2
```

#### 2. Router Mode Selection

```bash
# Pure load balancing
python -m dynamo.frontend --router-mode round-robin

# KV-aware routing
python -m dynamo.frontend --router-mode kv --kv-overlap-score-weight 1.0
```

#### 3. TLS Configuration

```bash
python -m dynamo.frontend \
    --tls-cert-path cert.pem \
    --tls-key-path key.pem \
    --http-port 8443
```

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Custom preprocessing | Rust implementation | Fork and modify |
| Custom routing | Limited to built-in modes | Use standalone router |
| Authentication | Not built-in | Use external proxy |

## Related Documentation

- [KV Cache Routing](docs/router/kv_cache_routing.md)
- [Backend Guide](docs/development/backend-guide.md)

## Tests

- Integration tests via backend-specific tests
- Router E2E tests: `tests/router/`
