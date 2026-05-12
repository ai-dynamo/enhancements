# Core Libraries Component Summary

## Overview

The Core Libraries are Dynamo's foundational Rust crates providing the distributed runtime, LLM-specific functionality, and supporting utilities. These libraries are consumed by Python bindings, C bindings, and potentially other language integrations.

## Location

- **Runtime**: `lib/runtime/` (`dynamo-runtime`)
- **LLM**: `lib/llm/` (`dynamo-llm`)
- **Parsers**: `lib/parsers/` (`dynamo-parsers`)

## Crate Structure

### `dynamo-runtime` (`lib/runtime/`)

Core distributed runtime functionality.

**Key Modules:**

| Module | File | Purpose |
|--------|------|---------|
| `component` | `component.rs` | Component registration and management |
| `config` | `config.rs` | Configuration management |
| `discovery` | `discovery/` | Service discovery via etcd |
| `distributed` | `distributed.rs` | Distributed runtime coordination |
| `engine` | `engine.rs` | Engine management |
| `health_check` | `health_check.rs` | Health checking infrastructure |
| `logging` | `logging.rs` | Structured logging |
| `metrics` | `metrics.rs` | Prometheus metrics |
| `pipeline` | `pipeline.rs` | Request pipeline |
| `protocols` | `protocols/` | Wire protocols |
| `runtime` | `runtime.rs` | Tokio runtime management |
| `service` | `service.rs` | Service abstractions |
| `storage` | `storage/` | Storage backends |
| `transports` | `transports/` | Network transports (NATS, TCP, HTTP) |
| `worker` | `worker.rs` | Worker lifecycle |

### `dynamo-llm` (`lib/llm/`)

LLM-specific functionality.

**Key Modules:**

| Module | File | Purpose |
|--------|------|---------|
| `backend` | `backend.rs` | Backend abstractions |
| `block_manager` | `block_manager/` | KV cache block management (KVBM) |
| `discovery` | `discovery/` | Model discovery |
| `engines` | `engines.rs` | Engine types |
| `entrypoint` | `entrypoint/` | Model entry points |
| `grpc` | `grpc/` | gRPC server |
| `http` | `http/` | HTTP server (OpenAI API) |
| `kv_router` | `kv_router/` | KV-aware routing |
| `kv` | `kv/` | KV cache abstractions |
| `migration` | `migration.rs` | KV migration |
| `model_card` | `model_card.rs` | Model deployment cards |
| `preprocessor` | `preprocessor/` | Request preprocessing |
| `protocols` | `protocols/` | LLM-specific protocols |
| `tokenizers` | `tokenizers.rs` | Tokenization |
| `tokens` | `tokens.rs` | Token handling |

### `dynamo-parsers` (`lib/parsers/`)

Parsing utilities.

## Public Interface

### Runtime Types

```rust
// From dynamo-runtime
pub struct DistributedRuntime { ... }
pub struct Component { ... }
pub struct Endpoint { ... }
pub struct Client { ... }
pub struct Runtime { ... }
```

### LLM Types

```rust
// From dynamo-llm
pub struct KvRouter { ... }
pub struct KvPushRouter { ... }
pub struct KvRouterConfig { ... }
pub struct ModelDeploymentCard { ... }
pub struct Preprocessor { ... }
pub struct HttpServer { ... }
```

## Internal Dependencies

### External Crates

| Crate | Purpose |
|-------|---------|
| `tokio` | Async runtime |
| `async-nats` | NATS client |
| `etcd-client` | etcd client |
| `axum` | HTTP framework |
| `tonic` | gRPC framework |
| `prometheus` | Metrics |
| `tracing` | Structured logging |
| `serde` | Serialization |
| `prost` | Protobuf |

### Transport Layer

| Transport | Module | Use Case |
|-----------|--------|----------|
| TCP | `transports/tcp/` | High-performance request plane |
| HTTP | `transports/http/` | Universal compatibility |
| NATS | `transports/nats/` | Pub/sub, request-reply |
| etcd | `transports/etcd/` | KV store, discovery |

## User/Developer Interaction

### 1. Rust Application

```rust
use dynamo_runtime::DistributedRuntime;
use dynamo_llm::http::HttpServer;

#[tokio::main]
async fn main() {
    let runtime = DistributedRuntime::new(...).await?;
    // Use runtime...
}
```

### 2. Via Python Bindings

Most users interact via Python bindings rather than directly with Rust.

### 3. Via C Bindings

C/C++ applications use the C API wrapper.

## Packaging & Containers

### Build Process

```bash
cargo build --release -p dynamo-runtime
cargo build --release -p dynamo-llm
```

### Workspace Structure

```toml
# Root Cargo.toml
[workspace]
members = [
    "lib/runtime",
    "lib/llm",
    "lib/parsers",
    "lib/bindings/python",
    "lib/bindings/c",
]
```

## Service Interface & I/O Contract

### Wire Protocols

| Protocol | Location | Format |
|----------|----------|--------|
| Request/Response | `protocols/` | Protobuf |
| KV Events | `kv_router/protocols.rs` | Protobuf |
| Health Check | `health_check.rs` | JSON |
| Metrics | `metrics.rs` | Prometheus |

### Configuration

Environment variables:
- `ETCD_ENDPOINTS` - etcd connection
- `NATS_SERVER` - NATS connection
- `DYN_*` - Dynamo-specific config

## Observability

### Logging

`tracing` crate with structured logging:
- Log levels: trace, debug, info, warn, error
- JSON output support
- Span-based context

### Metrics

Prometheus metrics via `prometheus` crate:
- Request latency histograms
- Throughput counters
- Resource utilization gauges

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| API stability | In development | Semantic versioning |
| Error types | Various | Unified error hierarchy |
| Configuration | Env vars | Document all options |
| Wire protocol | Protobuf | Version, document schema |
| Logging format | Structured | Document format |
| Metrics | Various | Document all metrics |

## Customization & Extension

### Extension Points

1. **Custom Transports** - Implement transport traits
2. **Custom Storage** - Implement storage backends
3. **Custom Protocols** - Add to `protocols/`

### Feature Flags

```toml
[features]
default = ["nats", "etcd"]
nats = ["async-nats"]
etcd = ["etcd-client"]
grpc = ["tonic"]
```

## Related Documentation

- [Runtime Guide](docs/development/runtime-guide.md)
- Rust API docs (via `cargo doc`)

## Tests

- Unit tests in each module
- Integration tests in `tests/`
- Benchmark tests in `benches/`
