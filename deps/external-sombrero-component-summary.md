# Sombrero Component Summary (nvidia-lpu/Groq)

## Overview

Sombrero is Groq's LLM load balancer built on [Cloudflare Pingora](https://github.com/cloudflare/pingora). It transparently proxies gRPC requests from Orion (client service) to Neutrinos (inference proxies), enabling token-level and request-level load balancing across inference engines. Named after the Sombrero Galaxy.

## Location

- **Repository**: `https://github.com/nvidia-lpu/sombrero` (private)
- **Primary Source**: `src/` (~83K LOC Rust)
- **Verifier**: `verifier/`
- **Configuration**: `config.toml`

## Internal Dependencies

### Languages

| Language | Size | Purpose |
|----------|------|---------|
| Rust | 83,037 bytes | Core load balancer |
| HCL | 30,317 bytes | Terraform infrastructure |
| Dockerfile | 1,628 bytes | Container build |
| Shell | 1,173 bytes | Scripts |

### Key Rust Dependencies

| Category | Dependencies |
|----------|--------------|
| **Proxy Framework** | Pingora (Cloudflare) |
| **gRPC** | tonic, prost |
| **Async Runtime** | tokio |
| **TLS** | OpenSSL, rustls |
| **Serialization** | serde, toml |

### External Services

| Service | Direction | Purpose |
|---------|-----------|---------|
| Orion | Upstream | Client requests |
| Neutrino | Downstream | Inference proxies |

## Module Structure

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `src/` | Core Rust source code |
| `verifier/` | Request verification |
| `kustomize/` | Kubernetes manifests |
| `local-k8s/` | Local K8s testing |
| `assets/` | Static assets |
| `tests/` | Test fixtures and keys |

### Key Files

| File | Purpose |
|------|---------|
| `config.toml` | Default configuration |
| `tls_enabled_config.toml` | TLS configuration |
| `compose.yaml` | Docker Compose setup |
| `gen_keys.sh` | TLS certificate generation |
| `buf.gen.yaml` | Protobuf generation config |

## Public Interface

### Architecture

```
Orion (Clients)
     │
     ▼ (gRPC)
┌─────────────┐
│  Sombrero   │ ← Neutrino registration (gRPC server)
│  (Pingora)  │
└─────────────┘
     │
     ▼ (gRPC)
┌─────────────┐
│  Neutrino   │ (multiple instances)
└─────────────┘
```

### gRPC Server

Neutrinos register with Sombrero's gRPC server:
- Real-time statistics reporting
- Backend availability updates
- Token-level metrics for load balancing

### Load Balancing

| Level | Description |
|-------|-------------|
| Request-level | Distribute requests across Neutrinos |
| Token-level | Balance based on token generation rates |

### Ports (Default)

| Port | Service |
|------|---------|
| 8001 | Proxy (TLS) |
| 8002 | Proxy (non-TLS) |
| 10000 | Neutrino registration |

## User/Developer Interaction

### 1. Build

```bash
# Standard build
cargo build

# With sccache (recommended)
cargo install sccache --locked
# Add to ~/.cargo/config.toml:
# [build]
# rustc-wrapper = "/path/to/sccache"
```

### 2. Generate TLS Certificates

```bash
./gen_keys.sh
```

### 3. Run Sombrero

```bash
# Without TLS
cargo run

# With TLS
cargo run -- --config tls_enabled_config.toml
```

### 4. Integration Test (Local)

```bash
# Terminal 1: Run Nova with mock agents
cd nova
cargo run --bin nova -- \
  --config-filepath tests/app_config.toml \
  --model-config-filepath tests/spec_decode_model_config.toml \
  --allocation c1r1:c2r2,c2r3 \
  --instance-model-name test \
  --mock-agents

# Terminal 2: Run Neutrino with Sombrero
cd neutrino
NEUTRINO_CFG=./neutrino_cli_config.txt \
cargo run --bin neutrino -- --sombrero-address localhost:10000

# Terminal 3: Run Sombrero
cd sombrero
cargo run

# Terminal 4: Test request
cd neutrino
cargo run --bin test_client -- \
  --show-output \
  --address localhost:8002
```

### 5. Docker Compose

```bash
# Requires neutrino repo next to sombrero
./gen_keys.sh
docker compose up

# Test with TLS
./target/debug/test_client \
  --address 10.5.0.2:8001 \
  -n 1 \
  --concurrency 1 \
  -r ../sombrero/tests/keys/rootCA.crt
```

## Packaging & Containers

### Build Artifacts

| Artifact | Description |
|----------|-------------|
| `sombrero` | Main load balancer binary |
| `lb_sim` | Load balancer simulator |

### Docker

```dockerfile
# Multi-stage build
# Requires neutrino repo adjacent
```

### Kubernetes

- Kustomize manifests in `kustomize/`
- Local K8s testing in `local-k8s/`

## Service Interface & I/O Contract

### Configuration (TOML)

```toml
# Key sections
[proxy]
listen_addr = "0.0.0.0:8002"
tls_listen_addr = "0.0.0.0:8001"

[registration]
listen_addr = "0.0.0.0:10000"

[tls]
cert_path = "./tests/keys/server.crt"
key_path = "./tests/keys/server.key"
```

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| gRPC requests | Orion clients | Protobuf |
| Neutrino registration | Neutrino instances | gRPC |
| Real-time stats | Neutrinos | gRPC streaming |

### Outputs

| Output | Destination | Format |
|--------|-------------|--------|
| Proxied requests | Neutrinos | gRPC |
| Load balancing decisions | Internal | Pingora routing |

## Observability

### Load Balancer Simulator

```bash
./target/debug/lb_sim -c sim_config.toml
# Outputs to lb_sim_outputs/
```

### Metrics

- Request latency
- Backend health
- Token throughput
- Connection counts

### Logging

- Structured logging
- Request tracing
- Error diagnostics

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| Configuration | TOML | Well-structured |
| TLS | OpenSSL | Certificate generation script |
| gRPC | Protobuf | Standard schemas |
| Simulator | Built-in | `lb_sim` binary |
| Deployment | Docker, K8s | Multiple options |

## Customization & Extension

### Extension Points

1. **Load Balancing Algorithms** - Pingora plugin system
2. **Health Checks** - Custom backend health logic
3. **Metrics** - Additional Prometheus metrics
4. **TLS** - Custom certificate configuration

### Configuration Options

| Option | Description |
|--------|-------------|
| `proxy.listen_addr` | Non-TLS proxy address |
| `proxy.tls_listen_addr` | TLS proxy address |
| `registration.listen_addr` | Neutrino registration |
| `tls.*` | Certificate paths |

## Dynamo Equivalent

**Closest Match**: **KV Router**

| Aspect | Sombrero | Dynamo KV Router |
|--------|----------|------------------|
| **Purpose** | Load balance to Neutrinos | Route to workers |
| **Algorithm** | Token-level + request-level | KV-aware + round-robin |
| **Backend Discovery** | gRPC registration | etcd service discovery |
| **Protocol** | gRPC proxy | TCP/HTTP |
| **Framework** | Pingora (Cloudflare) | Custom Rust |

**Key Differences**:
- Sombrero uses Pingora framework; Dynamo has custom router implementation
- Sombrero does token-level balancing; Dynamo does KV cache-aware routing
- Sombrero is gRPC-focused; Dynamo supports HTTP/TCP/NATS
- Sombrero has explicit Neutrino registration; Dynamo uses etcd discovery

## Related Documentation

- [Pingora Documentation](https://github.com/cloudflare/pingora)
- Buildkite Pipeline: `https://buildkite.com/groq/sombrero`
- Conventional Commits: `feat:`, `fix:` for releases

## Tests

- **Unit Tests**: `src/` inline tests
- **Integration**: Via Docker Compose
- **TLS Tests**: Using generated certificates
- **Simulator**: `lb_sim` for load testing
