# llm-d KV Cache Component Summary

## Overview

The llm-d KV Cache component provides distributed KV cache scheduling and offloading libraries. It tracks KV cache block locality across vLLM pods, enabling cache-aware routing decisions in the inference scheduler. The component maintains a global view of which pods have cached which prompt prefixes, optimizing for cache reuse.

## Location

- **Repository**: `https://github.com/llm-d/llm-d-kv-cache` (public)
- **Language**: Go
- **Stars**: 94+
- **Description**: Distributed KV cache scheduling & offloading libraries

## Internal Dependencies

### Go Dependencies

| Category | Dependencies |
|----------|--------------|
| **Kubernetes** | k8s.io/api, k8s.io/client-go |
| **Caching** | ttlcache, lru-cache |
| **Messaging** | ZeroMQ (zmq4) |
| **Serialization** | Protocol Buffers, msgpack |
| **Observability** | Prometheus client |
| **Testing** | Ginkgo, Gomega |

### External Services

| Service | Purpose |
|---------|---------|
| vLLM Pods | KV event source |
| ZeroMQ | Event streaming |
| Inference Scheduler | Cache-aware routing |
| Prometheus | Metrics |

## Module Structure

### Key Components

| Component | Purpose |
|-----------|---------|
| **Event Receiver** | Receives KV events from vLLM |
| **Block Index** | Tracks block locations |
| **Locality Tracker** | Maintains prefix-to-pod mapping |
| **Cache Scorer** | Provides scores to scheduler |
| **Offload Manager** | Handles tiered caching |

### Event Types

| Event | Description |
|-------|-------------|
| `BlockCreated` | New KV block allocated |
| `BlockEvicted` | KV block removed |
| `BlockMigrated` | KV block moved between pods |
| `PrefixCached` | Prefix stored in cache |

## Public Interface

### KV Events API

```go
// KV event structure
type KVEvent struct {
    Type      EventType
    BlockID   uint64
    PodID     string
    Prefix    []byte
    Timestamp time.Time
}

// Event types
const (
    BlockCreated EventType = iota
    BlockEvicted
    BlockMigrated
    PrefixCached
)
```

### Locality Query

```go
// Query cache locality for a prefix
type LocalityQuery struct {
    Prefix    []byte
    ModelID   string
}

type LocalityResult struct {
    Pods      []PodLocality
    CacheHits int
    Coverage  float64
}

type PodLocality struct {
    PodID       string
    BlockCount  int
    MatchLength int
    Score       float64
}
```

### Scorer Interface

```go
// Interface for scheduler integration
type CacheScorer interface {
    Score(prefix []byte, pod string) float64
    GetCandidates(prefix []byte) []ScoredPod
}
```

## User/Developer Interaction

### 1. vLLM Integration

vLLM pods emit KV events via ZeroMQ:

```python
# In vLLM configuration
kv_event_publisher:
  type: zmq
  endpoint: tcp://kv-cache-service:5555
```

### 2. Scheduler Integration

The inference scheduler uses the cache scorer:

```yaml
# Scheduler profile with cache scorer
profiles:
  - name: cache-aware
    plugins:
      score:
        enabled:
          - name: prefix-cache-scorer
            weight: 100
            args:
              kvCacheEndpoint: "kv-cache-service:8080"
```

### 3. Deploy KV Cache Service

```bash
# Deploy to Kubernetes
kubectl apply -f config/kv-cache/

# Verify service
kubectl get pods -l app=kv-cache
kubectl logs -l app=kv-cache
```

### 4. Query Cache Locality

```bash
# Check cache status (if API exposed)
curl http://kv-cache-service:8080/v1/locality?prefix=<hash>
```

## Packaging & Containers

### Build

```bash
# Build binary
make build

# Build container
make docker-build

# Run tests
make test
```

### Deployment

- Kubernetes Deployment
- ZeroMQ listener for vLLM events
- gRPC/HTTP API for scheduler queries

### Container Image

```dockerfile
FROM gcr.io/distroless/static:nonroot
COPY kv-cache /
ENTRYPOINT ["/kv-cache"]
```

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| KV events | vLLM pods | ZeroMQ messages |
| Locality queries | Scheduler | gRPC/HTTP |
| Pod metadata | K8s API | Watch events |

### Outputs

| Output | Destination | Format |
|--------|-------------|--------|
| Locality scores | Scheduler | gRPC response |
| Metrics | Prometheus | Exposition |
| Block index | In-memory | Data structure |

### Event Flow

```
vLLM Pod 1 ──┐
vLLM Pod 2 ──┼──(ZeroMQ)──→ KV Cache Service ──(gRPC)──→ Scheduler
vLLM Pod N ──┘                    │
                                  ▼
                            Block Index
                        (Prefix → Pods mapping)
```

## Observability

### Prometheus Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `kv_cache_blocks_total` | Gauge | Total tracked blocks |
| `kv_cache_events_received` | Counter | Events from vLLM |
| `kv_cache_query_duration` | Histogram | Query latency |
| `kv_cache_hit_ratio` | Gauge | Cache hit rate |
| `kv_cache_pods_tracked` | Gauge | Active pods |

### Logging

- Structured logging (JSON)
- Event processing logs
- Query logs

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| Event format | Protobuf/msgpack | Versioned |
| Query API | gRPC | Stable interface |
| Metrics | Prometheus | Comprehensive |
| Configuration | YAML/env | Documented |
| Documentation | README | Basic |

## Customization & Extension

### Extension Points

1. **Custom Event Sources** - Implement EventSource interface
2. **Custom Index** - Alternative block tracking
3. **Custom Scorer** - Different scoring algorithms
4. **Offload Backends** - CPU/disk tiering

### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `zmq.endpoint` | ZeroMQ listen address | `tcp://*:5555` |
| `api.port` | API server port | `8080` |
| `index.ttl` | Block TTL | `1h` |
| `index.maxSize` | Max blocks tracked | `1000000` |

### Tiered Caching (Advanced)

```yaml
# Enable tiered caching
tieredCache:
  enabled: true
  tiers:
    - name: gpu
      priority: 1
    - name: cpu
      priority: 2
      maxSize: 100GB
    - name: disk
      priority: 3
      path: /cache
```

## Dynamo Equivalent

**Closest Match**: **KVBM (KV Block Manager)**

| Aspect | llm-d KV Cache | Dynamo KVBM |
|--------|----------------|-------------|
| **Purpose** | Track KV cache locality | Manage KV cache memory |
| **Scope** | Global index across pods | Per-worker memory |
| **Events** | ZeroMQ from vLLM | NATS/ZMQ |
| **Routing** | Feed to scheduler | Feed to router |
| **Offload** | Tiered (planned) | CPU/disk offload |

**Key Differences**:
- llm-d KV Cache is a global index; Dynamo KVBM manages actual memory
- llm-d uses ZeroMQ; Dynamo uses NATS/ZMQ
- llm-d is Go; Dynamo KVBM is Rust
- llm-d is read-only tracker; Dynamo KVBM handles allocation

## Related Documentation

- [llm-d Architecture](https://github.com/llm-d/llm-d/docs/architecture.md)
- [Prefix Cache Guide](https://github.com/llm-d/llm-d/guides/precise-prefix-cache-aware/)
- [Tiered Prefix Cache](https://github.com/llm-d/llm-d/guides/tiered-prefix-cache/)

## Tests

- **Unit Tests**: Package-level tests
- **Integration Tests**: With mock vLLM
- **Benchmarks**: Index performance tests
