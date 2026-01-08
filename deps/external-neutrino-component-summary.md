# Neutrino Component Summary (nvidia-lpu/Groq)

## Overview

Neutrino is a high-performance proxy service for Inference Engines (IEs) that run on GroqNodes. It serves as a unified API layer that bridges multiple communication protocols (gRPC, HTTP/OpenAI, QUIC) to Cap'n Proto-based inference engines via UNIX sockets. Neutrino is the primary interface between upstream services and Groq's LPU inference engines.

## Location

- **Repository**: `https://github.com/nvidia-lpu/neutrino` (private)
- **Primary Source**: `src/` (~15,573 LOC Rust)
- **OpenAI API**: `src/openai_api/`
- **Tokenizers**: `assets/`
- **Version**: 4.32.0

## Internal Dependencies

### Rust Crates

| Category | Dependencies |
|----------|--------------|
| **Async Runtime** | tokio 1.45.0 (multi-thread, macros, signal) |
| **gRPC/Protobuf** | tonic 0.14, prost 0.14, tonic-health |
| **HTTP Server** | axum 0.8.4, hyper 1.0 |
| **Serialization** | serde 1.0, serde_json, toml 0.9 |
| **Tracing** | tracing, tracing-subscriber, tracing-appender |
| **Metrics** | prometheus 0.14.0 |
| **Cap'n Proto** | capnp 0.23 (sync_reader) |
| **Tokenization** | tiktoken-rs, tokenizers |
| **Templating** | minijinja 2.3.1 |
| **HTTP Client** | reqwest 0.12.7 (rustls-tls) |
| **Concurrent** | scc 3.3.2, crossbeam-skiplist, hashbrown |
| **Cloud** | google-cloud-storage, google-cloud-auth |
| **Image** | image 0.25 (PNG/JPEG) |

### External Services

| Service | Purpose |
|---------|---------|
| Inference Engines | Cap'n Proto over UNIX sockets |
| Sombrero | Load balancer registration |
| Prometheus | Metrics exposition |
| OpenTelemetry | Distributed tracing |

## Module Structure

### Key Source Files

| Module | File | Purpose |
|--------|------|---------|
| **Main** | `main.rs` | Server entry point |
| **gRPC Server** | `neutrino_server.rs` | gRPC service implementation |
| **OpenAI API** | `openai_api/server.rs` (86KB) | HTTP/SSE endpoints |
| **API Types** | `openai_api/api.rs` | Request/response types |
| **State Machine** | `request_state_machine/mod.rs` | Request lifecycle |
| **IE Sockets** | `sockets/*.rs` | Cap'n Proto communication |
| **Tokenizer** | `tokenizer.rs` | Multi-format tokenization |
| **Metrics** | `metrics.rs` | Prometheus integration |
| **Sombrero Client** | `sombrero_client.rs` | Load balancer integration |

### OpenAI API Submodules

| File | Purpose |
|------|---------|
| `harmony_compat.rs` | Harmony protocol compatibility |
| `reasoning.rs` | Chain-of-thought support |
| `media.rs` | Image/audio processing |
| `logprobs.rs` | Log probability handling |
| `tools.rs` | Tool calling framework |
| `state.rs` | Template & session state |

### Assets

| Asset | Purpose |
|-------|---------|
| `templates.json` | 30+ model-specific chat templates |
| `llama3.tiktoken` | Llama 3 tokenizer |
| `llama4.tiktoken` | Llama 4 tokenizer |
| `kimi_k2.tiktoken` | Kimi K2 tokenizer |

## Public Interface

### gRPC Interface (`groq_protos::neutrino_v1`)

| Method | Description |
|--------|-------------|
| `GetTextModelInferenceRequest` | Text generation |
| `GetTextModelInferenceStreamRequest` | Streaming text |
| `GetAudioModelInferenceRequest` | Audio transcription/translation |
| `GetEmbeddingModelInferenceRequest` | Embeddings |
| `GetTtsModelInferenceRequest` | Text-to-speech |
| `GetObjectDetectionInferenceRequest` | Vision tasks |
| `GetRerankModelInferenceRequest` | Ranking |

### OpenAI-Compatible HTTP API

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Chat with streaming SSE |
| `/v1/completions` | POST | Text completions |
| `/v1/embeddings` | POST | Embedding generation |
| `/v1/audio/transcriptions` | POST | Audio to text |
| `/v1/audio/translations` | POST | Audio translation |
| `/v1/audio/speech` | POST | Text-to-speech |
| `/v1/models` | GET | List models |
| `/groq/v1/status` | GET | Health check |
| `/groq/v1/app_state` | GET | Application state |
| `/groq/v1/history` | GET | Request history |
| `/metrics` | GET | Prometheus metrics |

### Cap'n Proto Protocol (UNIX Socket)

Defined in `ie.capnp`:
- `MultimodalQuery/Result` - Text generation
- `AudioQuery/Result` - Audio processing
- `EmbeddingQuery/Result` - Embeddings
- `TTSQuery/Result` - Text-to-speech
- Tool calling, log probabilities, streaming support

## User/Developer Interaction

### 1. Local Development

```bash
# Install Rust toolchain
source ./.buildkite/scripts/install-rust-toolchain.sh

# Build
cargo build

# Run with configuration
export NEUTRINO_CFG=./neutrino_cli_config.txt
cargo run --bin neutrino
```

### 2. Integration with Sombrero

```bash
# Run with Sombrero load balancer
cargo run --bin neutrino -- --sombrero-address localhost:10000
```

### 3. Test Client

```bash
# Send test requests
./target/debug/test_client --show-output -n 1 -c 1
```

### 4. HTTP Request Example

```bash
curl -X POST "http://localhost:51055/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama3-70b-8192",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": true
  }'
```

## Packaging & Containers

### Build Artifacts

| Artifact | Description |
|----------|-------------|
| `neutrino` | Main server binary |
| `test_client` | Test client binary |
| `mock_ie` | Mock inference engine |

### Docker Build

Multi-stage build:
1. Protoc build stage
2. Cap'n Proto build stage
3. Rust compilation stage
4. Final stage (~8-15MB binary)

### Deployment

- **Production**: Systemd service (`groq-neutrino`)
- **Binary Path**: `/nfs/geg2/c2r9-data1/neutrino/bin/neutrino-<version>`
- **Config**: `/etc/neutrino.config`, `/etc/llama-config`

## Service Interface & I/O Contract

### Configuration (TOML)

```toml
# Sections
[model]           # Model specifications
[server]          # gRPC, HTTP, QUIC addresses
[tokenizer]       # Tokenizer configuration
[sombrero]        # Load balancer settings
[tracing]         # OpenTelemetry endpoints
[shutdown]        # Grace periods
```

### Request State Machine

States: `Start → Queued → Processing → Generating → Complete`

### Buffer Limits

| Buffer | Max Size |
|--------|----------|
| gRPC messages | 26 MB |
| Cap'n Proto frames | 101 MB |
| Default timeout | 120 seconds |

## Observability

### Prometheus Metrics

- Port: 9089
- Request latency histograms
- Token throughput counters
- Queue depth gauges
- Connection health metrics

### OpenTelemetry

- Configurable tracing endpoint
- Context propagation across requests
- Span-based request tracking

### Health Endpoints

| Endpoint | Status |
|----------|--------|
| `/groq/v1/status` | 200 if IE connected |
| `/groq/v1/app_state` | Full application state |
| `/groq/v1/history` | Request debugging |

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| API | OpenAI-compatible | Well-documented |
| gRPC | Protobuf schemas | Versioned |
| Cap'n Proto | ie.capnp | Internal protocol |
| Metrics | Prometheus | Comprehensive |
| Configuration | TOML | Environment-driven |
| Versioning | SemVer (4.32.0) | Release Please |

## Customization & Extension

### Extension Points

1. **Model Templates** - Add to `assets/templates.json`
2. **Tokenizers** - Add `.tiktoken` files
3. **Request Handlers** - Extend state machine
4. **Custom Metrics** - Add to `metrics.rs`

### Configuration Options

| Option | Description |
|--------|-------------|
| Server addresses | gRPC, HTTP, QUIC ports |
| Tokenizer type | Tiktoken, HuggingFace |
| Sombrero | Load balancer connection |
| Tracing | OpenTelemetry endpoint |

## Dynamo Equivalent

**Closest Match**: **Frontend**

| Aspect | Neutrino | Dynamo Frontend |
|--------|----------|-----------------|
| **Purpose** | HTTP/gRPC entry point | HTTP entry point |
| **API** | OpenAI-compatible | OpenAI-compatible |
| **Protocols** | gRPC, HTTP, QUIC, Cap'n Proto | HTTP, gRPC |
| **Routing** | Via Sombrero | Via KV Router |
| **Streaming** | SSE | SSE |

**Key Differences**:
- Neutrino uses Cap'n Proto for IE communication (Groq-specific)
- Dynamo Frontend uses NATS/TCP for backend communication
- Neutrino has built-in tokenization; Dynamo delegates to backends
- Neutrino includes request state machine; Dynamo is stateless pass-through

## Related Documentation

- [Neutrino Developer Documentation](https://www.notion.so/groq/Neutrino-Developer-Documentation)
- `AGENTS.md` - AI development guidelines
- `CHANGELOG.md` - Version history

## Tests

- **Unit Tests**: Throughout `src/`
- **Benchmarks**: `benches/` (tokenizers, queue estimator, decoder)
- **Mocks**: `src/mocks/` (mock_inference_engine, mock_sombrero_server)
- **Integration**: Via test_client binary
