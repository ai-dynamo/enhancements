# Deployment Component Summary

## Overview

The Deployment component provides Kubernetes-native infrastructure for deploying Dynamo's distributed LLM inference platform. It includes a Go-based Kubernetes operator that manages Custom Resource Definitions (CRDs) for declarative deployment of inference graphs, automatic scaling, and multi-backend support.

## Location

- **Operator Source**: `deploy/cloud/operator/`
- **Helm Charts**: `deploy/cloud/helm/`
- **CRD Definitions**: `deploy/cloud/operator/config/crd/bases/`
- **Language**: Go 1.24+
- **Current Version**: 0.8.0

## Internal Dependencies

### Go Dependencies

| Package | Purpose |
|---------|---------|
| `k8s.io/api`, `k8s.io/client-go` (v0.33.3) | Kubernetes API |
| `sigs.k8s.io/controller-runtime` (v0.21.0) | Controller framework |
| `github.com/NVIDIA/grove` | Multi-GPU pod orchestration |
| `sigs.k8s.io/lws` | LeaderWorkerSet for multinode |
| `volcano.sh/apis` | Volcano scheduler integration |
| `istio.io/client-go` | Istio service mesh support |
| `go.etcd.io/etcd/client` | etcd backend |
| `github.com/prometheus-operator` | Prometheus integration |

### External Services

| Service | Purpose | Notes |
|---------|---------|-------|
| etcd | State storage | Via Helm dependency |
| NATS | Message broker | Via Helm dependency |
| Grove | Multi-GPU orchestration | Optional |
| Kai-Scheduler | GPU-aware scheduling | Optional |

## Module Structure

### API Types (`api/v1alpha1/`)

| CRD | Short Name | Purpose |
|-----|-----------|---------|
| `DynamoGraphDeployment` | `dgd` | Complete inference pipeline definition |
| `DynamoComponentDeployment` | `dcd` | Individual component (created by DGD) |
| `DynamoGraphDeploymentRequest` | `dgdr` | SLA-driven profiling and config generation |
| `DynamoGraphDeploymentScalingAdapter` | `dgdsa` | Autoscaler integration |
| `DynamoModel` | `dmodel` | Model discovery and endpoints |

### Controllers (`internal/controller/`)

| Controller | File | Purpose |
|------------|------|---------|
| `DynamoGraphDeploymentReconciler` | `dynamographdeployment_controller.go` | Reconciles DGD resources |
| `DynamoComponentDeploymentReconciler` | `dynamocomponentdeployment_controller.go` | Manages individual components |
| `DynamoGraphDeploymentRequestReconciler` | `dynamographdeploymentrequest_controller.go` | Handles SLA profiling |
| `DynamoGraphDeploymentScalingAdapterReconciler` | `dynamographdeploymentscalingadapter_controller.go` | Bridges autoscalers |

### Backend Implementations (`internal/dynamo/`)

| File | Purpose |
|------|---------|
| `backend_vllm.go` | vLLM-specific configuration |
| `backend_sglang.go` | SGLang-specific configuration |
| `backend_trtllm.go` | TensorRT-LLM configuration |
| `grove.go` | Grove orchestrator integration |
| `graph.go` | Deployment graph management |

## Public Interface

### Core CRD Types

```go
// DynamoGraphDeployment - Main deployment resource
type DynamoGraphDeploymentSpec struct {
    BackendFramework string                                    // sglang|vllm|trtllm
    PVCs             []PVC                                     // Persistent volumes
    Envs             []corev1.EnvVar                           // Global environment
    Services         map[string]*DynamoComponentDeploymentSharedSpec
}

// DynamoComponentDeploymentSharedSpec - Per-service config
type DynamoComponentDeploymentSharedSpec struct {
    ServiceName      string            // Component name
    ComponentType    string            // "frontend", "worker", "planner"
    SubComponentType string            // "prefill", "decode"
    Replicas         *int32
    Resources        *Resources        // GPU, CPU, memory
    ExtraPodSpec     *ExtraPodSpec     // Full K8s pod overrides
    // ...
}

// Resources - Resource specification
type Resources struct {
    Requests *ResourceItem
    Limits   *ResourceItem
    Claims   []ResourceClaim
}
```

### Key kubectl Commands

```bash
# View deployments
kubectl get dgd -n <namespace>
kubectl get dcd -n <namespace>
kubectl describe dgd <name> -n <namespace>

# Scale deployment
kubectl scale dgd <name> --replicas=N -n <namespace>

# Check status
kubectl get dgd -o wide
```

## User/Developer Interaction

### 1. Helm Installation

```bash
# Install CRDs
helm install dynamo-crds dynamo-crds-0.8.0.tgz

# Install Platform
helm install dynamo-platform dynamo-platform-0.8.0.tgz \
  --namespace dynamo-system \
  --create-namespace \
  --set dynamo-operator.enabled=true \
  --set nats.enabled=true \
  --set etcd.enabled=true
```

### 2. Simple Deployment

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: llama3-70b-agg
spec:
  backendFramework: vllm
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1
    VllmWorker:
      componentType: worker
      replicas: 1
      resources:
        limits:
          gpu: "4"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1
          args:
            - python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B
```

### 3. SLA-Driven Deployment (DGDR)

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: auto-config-llm
spec:
  model: Qwen/Qwen3-0.6B
  backend: vllm
  autoApply: true
  profilingConfig:
    profilerImage: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1
```

## Packaging & Containers

### Container Images

| Image | Purpose |
|-------|---------|
| `nvcr.io/nvidia/ai-dynamo/kubernetes-operator:0.8.0` | Operator manager |
| `nvcr.io/nvidia/ai-dynamo/vllm-runtime:*` | vLLM workers |
| `nvcr.io/nvidia/ai-dynamo/sglang-runtime:*` | SGLang workers |
| `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:*` | TRT-LLM workers |

### Helm Chart Structure

```
deploy/cloud/helm/
├── platform/                    # Main Dynamo Platform chart
│   ├── Chart.yaml (v0.8.0)
│   ├── values.yaml
│   └── components/operator/     # Operator subchart
├── crds/                        # CRD-only chart
└── README.md
```

### Helm Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `dynamo-operator` | 0.8.0 | Kubernetes operator |
| `nats` | 1.3.2 | Message broker |
| `etcd` | 12.0.18 | State storage |
| `kai-scheduler` | v0.9.4 | Scheduling optimization |
| `grove` | v0.1.0-alpha.3 | Multi-GPU orchestration |

## Service Interface & I/O Contract

### CRD API

- **API Group**: `nvidia.com`
- **API Version**: `v1alpha1`
- **Scope**: Namespaced

### Status Fields

```go
type DynamoGraphDeploymentStatus struct {
    Phase              string
    Conditions         []metav1.Condition
    ComponentStatuses  map[string]ComponentStatus
    ObservedGeneration int64
}
```

### Events

The operator emits Kubernetes events for:
- Deployment creation/updates
- Component scaling
- Error conditions
- Profiling completion (DGDR)

## Observability

### Prometheus Metrics

Operator exposes metrics via controller-runtime:
- Reconciliation latency
- Error counts
- Resource counts per CRD type

### Logging

Structured logging via `klog`:
- Log levels configurable via Helm values
- JSON output support

### Health Endpoints

| Endpoint | Port | Purpose |
|----------|------|---------|
| `/healthz` | 8081 | Liveness probe |
| `/readyz` | 8081 | Readiness probe |
| `/metrics` | 8080 | Prometheus metrics |

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| CRD versioning | `v1alpha1` | Graduate to `v1beta1` or `v1` |
| API stability | Evolving | Define deprecation policy |
| Helm values | Various | Document all configurable values |
| Upgrade path | Implicit | Document upgrade procedures |
| Multi-tenancy | Namespace-scoped | Document isolation guarantees |
| Resource quotas | Basic | Enhanced quota management |

## Customization & Extension

### Extension Points

1. **Custom Backends** - Add to `internal/dynamo/backend_*.go`
2. **Custom Schedulers** - Integrate via annotations
3. **Custom Metrics** - Add to operator metrics registry
4. **Webhooks** - Validation and mutation webhooks

### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `namespaceRestriction.enabled` | Limit to specific namespace | `false` |
| `webhook.enabled` | Enable admission webhooks | `true` |
| `metrics.enabled` | Enable Prometheus metrics | `true` |
| `grove.enabled` | Enable Grove integration | `false` |

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Cross-namespace | DGD is namespace-scoped | Deploy per namespace |
| CRD upgrades | Manual migration | Use Helm upgrade |
| Multi-cluster | Single cluster only | Use separate installations |

## Related Documentation

- [Kubernetes Installation Guide](docs/kubernetes/installation_guide.md)
- [API Reference](docs/kubernetes/api_reference.md)
- [Create Deployment](docs/kubernetes/deployment/create_deployment.md)
- [Helm Chart README](deploy/cloud/helm/README.md)

## Tests

- Unit tests: `deploy/cloud/operator/*_test.go`
- Integration tests: `tests/k8s/`
- E2E tests: `tests/fault_tolerance/deploy/`
