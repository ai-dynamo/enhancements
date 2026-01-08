# llm-d Main Component Summary

## Overview

llm-d is a Kubernetes-native distributed inference serving platform designed to achieve state-of-the-art (SOTA) performance for serving large language models across various hardware accelerators. The main repository serves as the orchestration layer, providing documentation, deployment recipes, infrastructure configurations, and coordination across the llm-d ecosystem of components.

## Location

- **Repository**: `https://github.com/llm-d/llm-d` (public)
- **Website**: `https://llm-d.github.io`
- **Documentation**: `docs/`
- **Guides**: `guides/`
- **Docker**: `docker/`
- **Stars**: 2,326+

## Internal Dependencies

### Languages

| Language | Purpose |
|----------|---------|
| Shell/Bash | Build scripts, automation |
| Python | Integration tests, examples |
| Makefile | Build orchestration |
| YAML | Kubernetes manifests, Helm |
| Markdown | Documentation |
| Dockerfile | Multi-architecture containers |
| Jupyter | Benchmarking notebooks |

### Ecosystem Repositories

| Repository | Purpose | Language |
|------------|---------|----------|
| `llm-d-inference-scheduler` | Request routing (EPP) | Go |
| `llm-d-kv-cache` | KV cache tracking | Go |
| `llm-d-inference-sim` | vLLM simulator | Go |
| `llm-d-routing-sidecar` | Prefill routing (deprecated) | Go |
| `llm-d-benchmark` | Benchmarking suite | Python/Jupyter |

### External Dependencies

| Dependency | Purpose |
|------------|---------|
| vLLM | Model serving backend |
| Kubernetes Gateway API | Ingress routing |
| Envoy | Gateway proxy |
| Gateway API Inference Extension (GIE) | Scheduling framework |
| Prometheus | Metrics |
| Helm | Package management |

## Module Structure

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `docs/` | Core documentation |
| `docs/accelerators/` | Hardware-specific guides |
| `docs/infra-providers/` | Cloud platform guides |
| `docs/monitoring/` | Observability setup |
| `docs/proposals/` | Design documents |
| `guides/` | Production deployment guides |
| `guides/recipes/` | Pre-built configurations |
| `docker/` | Multi-architecture Dockerfiles |
| `helpers/` | Utility scripts |
| `patches/` | NVSHMEM patches |
| `scripts/` | Build automation |

### Deployment Guides (`guides/`)

| Guide | Purpose |
|-------|---------|
| `inference-scheduling/` | Intelligent routing setup |
| `pd-disaggregation/` | Prefill/decode separation |
| `precise-prefix-cache-aware/` | Cache-aware routing |
| `predicted-latency-based-scheduling/` | SLA-aware routing |
| `tiered-prefix-cache/` | CPU/disk cache offloading |
| `wide-ep-lws/` | Wide expert-parallelism (MoE) |
| `workload-autoscaling/` | Dynamic scaling |
| `QUICKSTART.md` | Getting started |

### Infrastructure Providers (`docs/infra-providers/`)

| Provider | Status |
|----------|--------|
| AWS EKS | Supported |
| Azure AKS | Supported |
| Google GKE | Supported |
| OpenShift | Supported |
| DigitalOcean | Supported |
| Minikube | Supported (dev) |

## Public Interface

### Supported Hardware

| Accelerator | Notes |
|-------------|-------|
| NVIDIA GPUs | H100, A100, L40S, etc. |
| AMD MI250 | ROCm support |
| Google TPU v5e | Via XLA |
| Intel XPU | Via IPEX |
| CPU | For development/testing |

### Docker Images

| Variant | Dockerfile |
|---------|------------|
| CPU | `Dockerfile.cpu` |
| CUDA | `Dockerfile.cuda` |
| XPU (Intel) | `Dockerfile.xpu` |
| CUDA-EFA | AWS EFA support |

### Build Commands

```bash
# Build container
make image-build DEVICE=cuda VERSION=v0.2.1

# Push to registry
make image-push

# Install on K8s
make install-k8s NAMESPACE=default
```

### API Compatibility

- **OpenAI-compatible** (via vLLM)
- `POST /v1/completions`
- `POST /v1/chat/completions`
- `GET /v1/models`
- `GET /health`

## User/Developer Interaction

### 1. Quickstart

```bash
# Clone repository
git clone https://github.com/llm-d/llm-d.git
cd llm-d

# Follow quickstart guide
cat guides/QUICKSTART.md
```

### 2. Deploy with Helm

```bash
# Add Helm repository
helm repo add llm-d https://llm-d.github.io/llm-d-infra

# Install platform
helm install llm-d llm-d/llm-d-platform \
  --namespace llm-d \
  --create-namespace
```

### 3. Configure Gateway

See `docs/customizing-your-gateway.md` for:
- Istio integration
- Kgateway setup
- Custom ext-proc configuration

### 4. Enable Scheduling Features

```bash
# Enable prefix cache awareness
kubectl apply -f guides/precise-prefix-cache-aware/

# Enable P/D disaggregation
kubectl apply -f guides/pd-disaggregation/
```

## Packaging & Containers

### Multi-Architecture Support

| Architecture | Platforms |
|--------------|-----------|
| amd64 | Standard x86_64 |
| arm64 | AWS Graviton, Apple Silicon |

### Container Variants

```bash
# CUDA variant
docker build -f docker/Dockerfile.cuda -t llm-d:cuda .

# CPU variant
docker build -f docker/Dockerfile.cpu -t llm-d:cpu .

# Intel XPU variant
docker build -f docker/Dockerfile.xpu -t llm-d:xpu .
```

### Helm Charts (via `llm-d-infra`)

- Gateway configuration
- Prefill/Decode pod deployment
- Resource management
- Autoscaling

## Service Interface & I/O Contract

### Architecture Pattern

```
Client → Gateway (Envoy)
       ↓ (ext-proc callback)
  Inference Scheduler (EPP)
       ↓ (routes to optimal pod)
  vLLM Pods (Prefill ↔ Decode)
```

### Kubernetes Resources

| Resource | Purpose |
|----------|---------|
| `InferencePool` | Logical group of model-serving pods |
| `HTTPRoute` | Gateway API routing rules |
| `GatewayClass` | Gateway configuration |
| `ServiceMonitor` | Prometheus metrics |

### Configuration

| File | Purpose |
|------|---------|
| `values.yaml` | Helm values |
| Scheduling profiles | YAML-based plugin config |
| ConfigMap | Gateway customization |

## Observability

### Metrics

- Prometheus ServiceMonitor/PodMonitor
- Request latency histograms
- Token throughput
- Cache hit rates
- Queue depths

### Monitoring Setup

See `docs/monitoring/` for:
- Grafana dashboards
- Alert rules
- Log aggregation

### Health Probes

See `docs/readiness-probes.md` for:
- Liveness configuration
- Readiness checks
- Startup probes

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| API | OpenAI-compatible | Via vLLM |
| CRDs | Gateway API + GIE | K8s native |
| Configuration | Helm + YAML | Well-documented |
| Multi-hardware | 5+ accelerators | Comprehensive |
| Documentation | Extensive | Guides + docs |
| Governance | SIGs + maintainers | Community-driven |

## Customization & Extension

### Extension Points

1. **Scheduler Plugins** - Custom filters and scorers
2. **Gateway Integration** - Istio, Kgateway, custom
3. **Hardware Support** - Add accelerator Dockerfiles
4. **Deployment Recipes** - Custom configurations

### Special Interest Groups (SIGs)

| SIG | Focus |
|-----|-------|
| SIG Inference Scheduler | Request routing |
| SIG Benchmarking | Performance testing |
| SIG PD-Disaggregation | Prefill/decode architectures |
| SIG KV-Disaggregation | Cache management |
| SIG Installation | K8s integration |
| SIG Autoscaling | Dynamic scaling |
| SIG Observability | Monitoring |

### Contributing

- Bi-weekly community meetings
- Lazy consensus decision-making
- RFC/Proposal process for significant changes
- Conventional Commits

## Dynamo Equivalent

**Closest Match**: **Recipes + Deployment**

| Aspect | llm-d | Dynamo |
|--------|-------|--------|
| **Purpose** | Orchestration layer | Full platform |
| **Backend** | vLLM (external) | vLLM, SGLang, TRT-LLM (wrapped) |
| **K8s Native** | Yes (Gateway API) | Yes (Custom CRDs) |
| **Scheduling** | EPP + GIE | KV Router |
| **Configuration** | Helm + YAML | K8s Operator + Helm |

**Key Differences**:
- llm-d is an orchestration layer over vLLM; Dynamo is a full platform
- llm-d uses Gateway API Inference Extension; Dynamo has custom router
- llm-d emphasizes recipes/guides; Dynamo provides integrated components
- llm-d is community-driven (SIGs); Dynamo is NVIDIA-led

## Related Documentation

- [Architecture](docs/architecture.md)
- [Getting Started](guides/QUICKSTART.md)
- [Gateway Customization](docs/customizing-your-gateway.md)
- [Readiness Probes](docs/readiness-probes.md)
- [Contributing](CONTRIBUTING.md)

## Tests

- **Integration Tests**: Python-based
- **Recipe Validation**: Per-guide testing
- **CI/CD**: GitHub Actions
- **Benchmarks**: Via `llm-d-benchmark` repo
