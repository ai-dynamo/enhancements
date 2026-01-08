# llm-d Inference Scheduler Component Summary

## Overview

The llm-d Inference Scheduler provides an "Endpoint Picker (EPP)" component that makes optimized routing decisions for inference requests. It extends the [Gateway API Inference Extension (GIE)](https://github.com/kubernetes-sigs/gateway-api-inference-extension) project with llm-d-specific features like P/D disaggregation, prefix cache awareness, and predicted latency-based scheduling.

## Location

- **Repository**: `https://github.com/llm-d/llm-d-inference-scheduler` (public)
- **Language**: Go
- **Stars**: 115+
- **Documentation**: `docs/`

## Internal Dependencies

### Go Dependencies

| Category | Dependencies |
|----------|--------------|
| **Kubernetes** | k8s.io/api, k8s.io/client-go, sigs.k8s.io/controller-runtime |
| **Gateway API** | sigs.k8s.io/gateway-api |
| **GIE** | Gateway API Inference Extension |
| **Caching** | ttlcache, lru-cache, Ristretto |
| **Messaging** | ZeroMQ (zmq4) |
| **Redis** | go-redis |
| **Observability** | Prometheus client, OpenTelemetry |
| **Serialization** | Protocol Buffers, msgpack, CBOR |
| **AI/ML** | OpenAI Go client |
| **Testing** | Ginkgo, Gomega |

### External Services

| Service | Purpose |
|---------|---------|
| Envoy | Gateway proxy (ext-proc) |
| vLLM Pods | Inference backends |
| Prometheus | Metrics |
| Redis | Optional cache store |
| ZeroMQ | KV event streaming |

## Module Structure

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `cmd/` | CLI entry points |
| `pkg/scheduler/` | Core scheduling logic |
| `pkg/scheduler/plugins/` | Filter and scorer plugins |
| `pkg/scheduler/framework/` | Plugin framework |
| `docs/` | Architecture documentation |
| `config/` | K8s manifests |
| `hack/` | Development scripts |
| `test/` | Integration tests |

### Plugin System

| Type | Purpose |
|------|---------|
| **Filters** | Exclude unsuitable pods |
| **Scorers** | Rank suitable pods |
| **Pre-Dispatch** | Pre-processing hooks |
| **Post-Dispatch** | Post-processing hooks |

## Public Interface

### Scheduler Plugins

#### Filters (Exclude Unsuitable Pods)

| Filter | Description |
|--------|-------------|
| `decode-filter` | Select decode-only pods |
| `prefill-filter` | Select prefill-only pods |
| `by-label-selector` | Custom pod selection via labels |
| Custom filters | Implement Filter interface |

#### Scorers (Rank Pod Candidates)

| Scorer | Description |
|--------|-------------|
| `prefix-cache-scorer` | KV-cache locality scoring |
| `load-aware-scorer` | Load balancing (queue depth) |
| `session-affinity-scorer` | Session stickiness |
| `predicted-latency-scorer` | SLA-aware routing |
| Custom scorers | Implement Scorer interface |

### Configuration

```yaml
apiVersion: inference.gateway.io/v1alpha1
kind: InferencePool
metadata:
  name: my-model-pool
spec:
  targetModel: llama-70b
  selector:
    matchLabels:
      model: llama-70b
---
# Scheduler profile
profiles:
  - name: default
    plugins:
      filter:
        enabled:
          - name: decode-filter
      score:
        enabled:
          - name: prefix-cache-scorer
            weight: 100
          - name: load-aware-scorer
            weight: 50
```

### ext-proc Interface

The EPP receives routing requests via Envoy's [External Processing](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_proc_filter) filter:

```
Client → Envoy Gateway
              │
              ▼ (ext-proc gRPC)
         EPP Scheduler
              │
              ▼ (routing decision)
         vLLM Pod (selected)
```

## User/Developer Interaction

### 1. Installation

```bash
# Clone repository
git clone https://github.com/llm-d/llm-d-inference-scheduler.git
cd llm-d-inference-scheduler

# Build
make build

# Deploy to K8s
kubectl apply -f config/
```

### 2. Configuration

```yaml
# Scheduler configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: epp-config
data:
  config.yaml: |
    profiles:
      - name: default
        plugins:
          filter:
            enabled:
              - name: decode-filter
          score:
            enabled:
              - name: prefix-cache-scorer
                weight: 100
```

### 3. Enable Features

```bash
# Enable prefix cache awareness
kubectl apply -f examples/prefix-cache/

# Enable P/D disaggregation
kubectl apply -f examples/pd-disagg/

# Enable predicted latency scoring
kubectl apply -f examples/latency-based/
```

### 4. Verify Routing

```bash
# Check EPP logs
kubectl logs -l app=inference-scheduler

# Verify routing decisions
curl -v http://gateway/v1/chat/completions
```

## Packaging & Containers

### Build

```bash
# Build binary
make build

# Build container
make docker-build

# Push container
make docker-push
```

### Deployment

- Kubernetes Deployment
- Envoy ext-proc sidecar or standalone
- Prometheus ServiceMonitor

### Container Image

```dockerfile
FROM gcr.io/distroless/static:nonroot
COPY inference-scheduler /
ENTRYPOINT ["/inference-scheduler"]
```

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| HTTP request headers | Envoy ext-proc | gRPC |
| Pod metrics | Prometheus/vLLM | Metrics API |
| KV cache events | ZeroMQ | Structured events |
| InferencePool | K8s API | CRD |

### Outputs

| Output | Destination | Format |
|--------|-------------|--------|
| Routing decision | Envoy | ext-proc response |
| Selected pod | HTTP header | Pod address |
| Metrics | Prometheus | Exposition |

### Scheduling Flow

1. Request arrives at Envoy Gateway
2. ext-proc filter calls EPP
3. EPP runs filters to get candidate pods
4. EPP runs scorers to rank candidates
5. EPP returns highest-scoring pod
6. Envoy routes request to selected pod

## Observability

### Prometheus Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `epp_scheduling_duration_seconds` | Histogram | Scheduling latency |
| `epp_filter_duration_seconds` | Histogram | Per-filter latency |
| `epp_scorer_duration_seconds` | Histogram | Per-scorer latency |
| `epp_routing_decisions_total` | Counter | Total routing decisions |
| `epp_cache_hits_total` | Counter | Prefix cache hits |

### Logging

- Structured logging (JSON)
- Request tracing
- Plugin execution logs

### Tracing

- OpenTelemetry integration
- Distributed tracing support

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| Plugin API | Stable | Filter/Scorer interfaces |
| Configuration | YAML profiles | Versioned |
| Metrics | Prometheus | Well-defined |
| CRDs | InferencePool | Via GIE |
| Documentation | Architecture docs | Comprehensive |

## Customization & Extension

### Custom Filter Plugin

```go
type MyFilter struct{}

func (f *MyFilter) Name() string {
    return "my-filter"
}

func (f *MyFilter) Filter(ctx context.Context, state *framework.CycleState, pod *v1.Pod) *framework.Status {
    // Custom filtering logic
    if shouldExclude(pod) {
        return framework.NewStatus(framework.Unschedulable, "excluded")
    }
    return framework.NewStatus(framework.Success, "")
}
```

### Custom Scorer Plugin

```go
type MyScorer struct{}

func (s *MyScorer) Name() string {
    return "my-scorer"
}

func (s *MyScorer) Score(ctx context.Context, state *framework.CycleState, pod *v1.Pod) (int64, *framework.Status) {
    // Custom scoring logic
    score := calculateScore(pod)
    return score, framework.NewStatus(framework.Success, "")
}
```

### Extension Points

1. **Custom Filters** - Implement Filter interface
2. **Custom Scorers** - Implement Scorer interface
3. **Custom Metrics** - Add to Prometheus registry
4. **KV Cache Integration** - Custom event sources

## Dynamo Equivalent

**Closest Match**: **KV Router**

| Aspect | llm-d Scheduler | Dynamo KV Router |
|--------|-----------------|------------------|
| **Purpose** | Route to optimal pod | Route to optimal worker |
| **Plugin System** | Filters + Scorers | Routing modes |
| **Cache Awareness** | prefix-cache-scorer | Radix tree matching |
| **Load Balancing** | load-aware-scorer | Round-robin fallback |
| **Framework** | GIE + ext-proc | Custom Rust |

**Key Differences**:
- llm-d uses Envoy ext-proc; Dynamo has embedded router
- llm-d has pluggable filter/scorer framework; Dynamo has fixed modes
- llm-d uses K8s Gateway API; Dynamo uses etcd discovery
- llm-d is Go; Dynamo router is Rust

## Related Documentation

- [Architecture](docs/architecture.md)
- [P/D Disaggregation](docs/disagg_pd.md)
- [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension)
- [Development Guide](DEVELOPMENT.md)

## Tests

- **Unit Tests**: `pkg/` test files
- **Integration Tests**: `test/` directory
- **E2E Tests**: Via llm-d main repo
- **Benchmarks**: Performance tests

## Community

- **Slack**: `#sig-inference-scheduler` in llm-d workspace
- **Meetings**: Bi-weekly Wednesday 10AM PDT
- **Meeting Notes**: [Google Doc](https://docs.google.com/document/d/1Pf3x7ZM8nNpU56nt6CzePAOmFZ24NXDeXyaYb565Wq4)
