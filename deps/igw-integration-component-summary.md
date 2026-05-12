# IGW (Inference Gateway) Integration Component Summary

## Overview

The IGW (Inference Gateway) integration enables Dynamo to work with the Kubernetes Gateway API Inference Extension (GAIE) and kGateway. It provides a custom EPP (Endpoint Picker Processor) that exposes Dynamo's KV-aware routing through the standard gateway infrastructure, allowing Dynamo deployments to be accessed via Kubernetes Gateway API with intelligent token-aware load balancing.

## Location

- **Integration Root**: `deploy/inference-gateway/`
- **Helm Chart**: `deploy/inference-gateway/helm/dynamo-gaie/`
- **C FFI Bindings**: `lib/bindings/c/src/lib.rs`
- **EPP Dockerfile**: `container/Dockerfile.epp`
- **Build Script**: `deploy/inference-gateway/build-epp-dynamo.sh`

## Internal Dependencies

### Dynamo Components

| Component | Usage |
|-----------|-------|
| **KV Router** | Core routing logic exposed via C FFI |
| **C Bindings** | FFI layer for GAIE EPP integration |
| **Frontend** | Target of InferencePool selector |

### External Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| Gateway API CRDs | v1.3.0 | Kubernetes Gateway API |
| GAIE CRDs | v0.5.1 | InferenceModel, InferencePool |
| kGateway | v2.0.3 | Gateway implementation |
| Envoy | - | Proxy with ext-proc support |

### CRDs Used

| CRD | Source | Purpose |
|-----|--------|---------|
| `Gateway` | Gateway API | Gateway instance |
| `HTTPRoute` | Gateway API | Route to InferencePool |
| `InferencePool` | GAIE | Pool of inference endpoints |
| `InferenceModel` | GAIE | Model definition |
| `EndpointPickerConfig` | GAIE | EPP plugin configuration |

## Module Structure

### Helm Chart Templates

| Template | Purpose |
|----------|---------|
| `dynamo-epp.yaml` | Custom EPP deployment with Dynamo config |
| `epp-configmap.yaml` | EndpointPickerConfig with plugin chain |
| `inference-pool.yaml` | InferencePool CRD targeting Frontend |
| `inference-model.yaml` | InferenceModel CRD for model |
| `http-router.yaml` | HTTPRoute configuration |
| `service.yaml` | EPP service |

### EPP Plugin Chain

```yaml
plugins:
  - name: single-profile-handler   # Required profile handler
  - name: dynamo-inject-workerid   # Inject worker IDs into requests
  - name: dyn-kv                   # KV-aware scorer (Dynamo router)
  - name: picker                   # Max-score picker
  - name: dynamo-cleanup           # Request lifecycle cleanup
```

### Key Files

| File | Purpose |
|------|---------|
| `deploy/inference-gateway/README.md` | Integration guide |
| `deploy/inference-gateway/install_gaie_crd_kgateway.sh` | Install CRDs and kGateway |
| `deploy/inference-gateway/build-epp-dynamo.sh` | Build custom EPP image |
| `deploy/inference-gateway/epp-patches/v0.8.0/gaie.patch` | GAIE patches for Dynamo |
| `lib/bindings/c/src/lib.rs` | C FFI for KV router |

## Public Interface

### Integration Modes

| Mode | EPP Image | Routing | Use Case |
|------|-----------|---------|----------|
| **Dynamo Custom EPP** | Custom build | KV-aware | Production (recommended) |
| **Standard GAIE EPP** | gaie/epp | Round-robin | Simple setup |

### Helm Values

```yaml
epp:
  useDynamo: true                    # Enable Dynamo mode
  dynamo:
    kvBlockSize: "16"                # MANDATORY: KV block size
    namespace: "dynamo"              # Dynamo namespace
    routerReplicaSync: true          # Sync router instances
    useKvRouting: true               # Enable KV-aware routing
    busyThreshold: 0.9               # Worker load threshold
    overlapScoreWeight: 1.0          # Token overlap weight
    routerTemperature: 0.0           # Worker selection temperature
```

### InferencePool Selector

```yaml
apiVersion: inference.gateway.networking.x-k8s.io/v1alpha1
kind: InferencePool
metadata:
  name: dynamo-pool
spec:
  targetModel: llama3-70b
  selector:
    nvidia.com/dynamo-component: Frontend
  extensionRef:
    name: dynamo-epp
    failureMode: FailOpen
```

### HTTPRoute Configuration

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: dynamo-route
spec:
  parentRefs:
    - name: inference-gateway
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /v1
      backendRefs:
        - group: inference.gateway.networking.x-k8s.io
          kind: InferencePool
          name: dynamo-pool
```

## User/Developer Interaction

### 1. Install Gateway Infrastructure

```bash
# Install CRDs and kGateway
./deploy/inference-gateway/install_gaie_crd_kgateway.sh

# Verify installation
./recipes/gaie_checks.sh
```

### 2. Build Custom EPP (Mode 1)

```bash
# Build Dynamo EPP image
./deploy/inference-gateway/build-epp-dynamo.sh

# Push to registry
docker push <registry>/dynamo-epp:latest
```

### 3. Deploy with Helm

```bash
# Install Dynamo GAIE integration
helm install dynamo-gaie deploy/inference-gateway/helm/dynamo-gaie/ \
  --namespace dynamo \
  --set epp.useDynamo=true \
  --set epp.dynamo.kvBlockSize="16" \
  --set inferenceModel.modelName="llama3-70b"
```

### 4. Verify Integration

```bash
# Check InferencePool
kubectl get inferencepool -n dynamo

# Check EPP logs
kubectl logs -l app=dynamo-epp -n dynamo

# Test endpoint
curl http://<gateway-ip>/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3-70b", "messages": [{"role": "user", "content": "Hello"}]}'
```

## Packaging & Containers

### EPP Container Build

```dockerfile
# container/Dockerfile.epp
FROM golang:1.22 AS builder
# Enable CGO for Dynamo static library
ENV CGO_ENABLED=1
# Build with Dynamo KV routing plugin
RUN go build -o epp ./cmd/epp
```

### Build Process

```bash
# 1. Build Dynamo static library
cargo build --release -p dynamo-llm-capi

# 2. Generate C headers
cbindgen --config cbindgen.toml -o dynamo_llm.h

# 3. Apply GAIE patches
patch -p1 < epp-patches/v0.8.0/gaie.patch

# 4. Build EPP image
docker build -f Dockerfile.epp -t dynamo-epp .
```

### Container Images

| Image | Purpose |
|-------|---------|
| `dynamo-epp` | Custom EPP with Dynamo KV routing |
| `gaie/epp` | Standard GAIE EPP (fallback) |
| `envoyproxy/envoy` | Gateway proxy |

## Service Interface & I/O Contract

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
│                                                                  │
│   Client                                                         │
│     │                                                            │
│     ▼                                                            │
│   ┌─────────────────┐                                           │
│   │   kGateway      │  (Envoy with ext-proc)                    │
│   │   (Gateway)     │                                           │
│   └────────┬────────┘                                           │
│            │                                                     │
│            ▼  ext-proc callback                                  │
│   ┌─────────────────┐                                           │
│   │   Dynamo EPP    │  (Custom EPP with KV routing)             │
│   │   - dyn-kv      │  ← Calls Dynamo KV router via FFI         │
│   │   - picker      │                                           │
│   └────────┬────────┘                                           │
│            │                                                     │
│            ▼  routing decision                                   │
│   ┌─────────────────┐                                           │
│   │  InferencePool  │  (Frontend pods)                          │
│   │  ┌───┐ ┌───┐    │                                           │
│   │  │ F │ │ F │    │  ← nvidia.com/dynamo-component: Frontend  │
│   │  └───┘ └───┘    │                                           │
│   └─────────────────┘                                           │
│            │                                                     │
│            ▼                                                     │
│   ┌─────────────────┐                                           │
│   │  Worker Pods    │  (vLLM, TRT-LLM, SGLang)                  │
│   └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

### C FFI Interface

```c
// lib/bindings/c/src/lib.rs exposed as C API
int dynamo_llm_init(const char* config);
float dynamo_kv_score(const char* request, const char* worker_id);
const char* dynamo_select_worker(const char* request);
void dynamo_cleanup(const char* request_id);
```

### NVIDIA Extension Fields

Request extensions for disaggregated serving (`nvext`):

| Field | Type | Purpose |
|-------|------|---------|
| `prefill_worker_id` | string | Target prefill worker |
| `decode_worker_id` | string | Target decode worker |
| `token_data` | bytes | Pre-tokenized prompt |
| `enable_local_updates` | bool | Router bookkeeping (GAIE sidecar mode) |
| `backend_instance_id` | string | Target specific backend instance |

## Observability

### Metrics

EPP exposes Prometheus metrics:
- Routing decision latency
- Worker selection distribution
- KV cache hit rates
- Error counts

### Logging

```bash
# EPP logs
kubectl logs -l app=dynamo-epp -n dynamo

# Gateway logs
kubectl logs -l app=kgateway -n gateway-system
```

### Health Checks

| Endpoint | Service | Purpose |
|----------|---------|---------|
| `/healthz` | EPP | Liveness |
| `/readyz` | EPP | Readiness |
| `/health` | Gateway | Gateway health |

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| CRD versions | GAIE v0.5.1/v0.8.0 | Track upstream GAIE |
| Helm chart | v0.2.0 | Versioned releases |
| C FFI | Basic | Stable ABI, versioning |
| Configuration | Helm values | Document all options |
| Error handling | Basic | Standardize error codes |

## Customization & Extension

### Extension Points

1. **Custom Scorers** - Add EPP plugins via GAIE plugin system
2. **Configuration** - Helm values for routing parameters
3. **FFI Extensions** - Extend C bindings for new features

### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `epp.useDynamo` | Enable Dynamo mode | `true` |
| `epp.dynamo.kvBlockSize` | KV block size | Required |
| `epp.dynamo.busyThreshold` | Worker load threshold | `0.9` |
| `epp.dynamo.overlapScoreWeight` | Token overlap weight | `1.0` |
| `epp.dynamo.routerTemperature` | Selection temperature | `0.0` |

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| GAIE version | Patches required | Use supported versions |
| Custom EPP build | Manual process | Use build script |
| Multi-model | One InferencePool per model | Deploy multiple pools |

## Relationship to Router

The IGW integration **uses** the KV Router component:

| Aspect | Direct Router | IGW Integration |
|--------|---------------|-----------------|
| **Entry Point** | Frontend embedded | kGateway/Envoy |
| **Routing Logic** | Rust (native) | C FFI → Rust |
| **Configuration** | CLI args | Helm values |
| **Protocol** | HTTP/NATS | Gateway API + ext-proc |

The C FFI bindings expose the same KV routing logic, allowing the gateway to make identical routing decisions as the embedded router.

## Dynamo Equivalent (for External Comparison)

**Comparison to llm-d IGW Integration:**

| Aspect | Dynamo IGW | llm-d IGW |
|--------|------------|-----------|
| **Gateway** | kGateway | kGateway/Istio |
| **EPP** | Custom (C FFI) | Go plugins |
| **Routing** | KV-aware via FFI | Native Go plugins |
| **Integration** | Patch-based | Native GAIE |

Dynamo's approach uses C FFI to reuse the Rust KV router, while llm-d implements routing natively in Go.

## Related Documentation

- [Integration Guide](deploy/inference-gateway/README.md)
- [GAIE Documentation](https://github.com/kubernetes-sigs/gateway-api-inference-extension)
- [kGateway](https://kgateway.io/)
- [Gateway API](https://gateway-api.sigs.k8s.io/)

## Tests

- **Verification Script**: `recipes/gaie_checks.sh`
- **Integration Tests**: Via recipes with GAIE manifests
- **Example**: `recipes/llama-3-70b/vllm/agg/gaie/`
