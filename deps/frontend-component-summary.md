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

**Server Configuration:**
- `--http-host` (default: `0.0.0.0`)
- `--http-port` (default: `8000`)
- `--tls-cert-path`, `--tls-key-path` - TLS configuration
- `--namespace` - Scope model discovery

**Router Configuration:**
- `--router-mode` - `round-robin`, `random`, `kv`
- `--kv-overlap-score-weight` - KV cache reuse weight
- `--router-temperature` - Worker selection randomness
- `--no-kv-events` - Disable KV event tracking
- `--router-replica-sync` - Multi-replica synchronization

**Backend Configuration:**
- `--request-plane` - `tcp`, `http`, `nats`
- `--store-kv` - `etcd`, `file`, `mem`

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
