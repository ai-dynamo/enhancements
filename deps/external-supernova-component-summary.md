# Supernova Component Summary (nvidia-lpu/Groq)

## Overview

Supernova is a set of Kubernetes controllers that manage model deployments on Groq infrastructure. It handles allocation and repair within a cluster, while Nebula manages all Supernovas globally. Together they provide automated model instance lifecycle management, failure recovery, and multi-cluster coordination.

## Location

- **Repository**: `https://github.com/nvidia-lpu/supernova` (private)
- **Primary Source**: `internal/`, `cmd/`
- **API Definitions**: `api/`
- **Tokenizers**: `tokenizers/`
- **Language**: Go (2.6M LOC)

## Internal Dependencies

### Languages

| Language | Size | Purpose |
|----------|------|---------|
| Go | 2,618,004 bytes | Controllers, CLI |
| Jinja | 46,355 bytes | Templates |
| Shell | 29,981 bytes | Scripts |
| Makefile | 8,386 bytes | Build automation |

### Go Dependencies

| Category | Dependencies |
|----------|--------------|
| **Kubernetes** | k8s.io/api, k8s.io/client-go |
| **Controller** | sigs.k8s.io/controller-runtime |
| **CLI** | cobra, pflag |
| **Testing** | ginkgo, gomega |

### Architecture

```
┌─────────────────────────────────────────────────┐
│                   Nebula                         │
│         (Global Multi-Cluster Manager)           │
└─────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│ Supernova   │      │ Supernova   │      │ Supernova   │
│ (Cluster 1) │      │ (Cluster 2) │      │ (Cluster 3) │
└─────────────┘      └─────────────┘      └─────────────┘
```

## Module Structure

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `api/` | CRD type definitions |
| `cmd/` | CLI entry points |
| `internal/` | Controller implementations |
| `config/` | Kustomize configurations |
| `tokenizers/` | HuggingFace tokenizer definitions |
| `hack/` | Development scripts |
| `kustomize/` | K8s manifests |
| `test/` | Integration tests |
| `docs/` | Documentation |

### CLI Commands (`cmd/`)

| Command | Purpose |
|---------|---------|
| `supernova` | Main controller |
| `nebula` | Global coordinator |
| `kubectl-fit` | Topology visualizer plugin |
| `alloc` | Allocation utility |

## Public Interface

### Custom Resource Definitions

| CRD | Short Name | Purpose |
|-----|-----------|---------|
| `InferenceEngineInstance` | `iei` | Individual model instance |
| `InferenceEngineDeployment` | `ied` | Deployment specification |
| `Release` | `gr` | Release management |
| `Deployment` | `gd` | Global deployment |

### kubectl Commands

```bash
# List inference engine instances
kubectl get iei --all-namespaces

# Get deployments (global)
kubectl get gd -n inference-engine --context msp2-prod1

# Get releases
kubectl get gr -n inference-engine --context msp2-prod1

# Get agent pods
kubectl get pods -l supernova.groq.io/model-instance=<name>,supernova.groq.io/role=agent

# Get conductor pod
kubectl get pods -l supernova.groq.io/model-instance=<name>,supernova.groq.io/role=conductor

# Topology visualizer
kubectl fit --context dal1-prod1
```

### InferenceEngineInstance Status

```yaml
apiVersion: inference-engine.groq.io/v1
kind: InferenceEngineInstance
metadata:
  name: example-llama-32-90b-vision-2
spec:
  model: llama32-90b
  targetAgents: 108
status:
  readyAgents: 108
  pendingAgents: 0
```

### Labels

| Label | Purpose |
|-------|---------|
| `supernova.groq.io/model-instance` | Instance identifier |
| `supernova.groq.io/role` | `agent` or `conductor` |

## User/Developer Interaction

### 1. View Instances

```bash
kubectl get iei --all-namespaces
# Output:
# NAMESPACE      NAME                    MODEL         TARGETAGENTS   READYAGENTS   PENDINGAGENTS
# default        example-llama-32-90b    llama32-90b   108            108           0
```

### 2. Inspect Instance

```bash
kubectl describe iei example-llama-32-90b-vision-2

# Check events
kubectl get events \
  --field-selector involvedObject.kind=InferenceEngineInstance \
  --field-selector involvedObject.name=example-llama-32-90b-vision-2
```

### 3. Topology Visualizer (kubectl plugin)

```bash
# Install
GOPRIVATE=github.com/groq go install github.com/groq/supernova/cmd/kubectl-fit@latest

# Run
kubectl fit --context dal1-prod1
```

### 4. Development

```bash
# Run tests
make test

# Download build data for tests
export GROQ_REGISTRY_SECRET=$(groq auth token)
go run ./cmd/alloc --build <build_id> --save-build-output-path ./internal/cluster/testdata/builds/
```

## Packaging & Containers

### Build

```bash
make build
make test
```

### Deployment

- **Supernova**: Per-cluster controller
- **Nebula**: Global coordinator (one leader elected)
- **Kustomize**: K8s manifest generation

### Tokenizers

Tokenizers deployed to NFS: `/nfs/supernova/tokenizers`

Supported formats:
- Canonical HF path: `<org>/<model>`
- Normalized: `<org>_<model>` (lowercase)

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| InferenceEngineDeployment | K8s API | CRD |
| Cluster topology | K8s nodes | Node labels |
| Build data | Groq registry | Binary |

### Outputs

| Output | Destination | Format |
|--------|-------------|--------|
| Agent Pods | K8s API | Pod specs |
| Conductor Pods | K8s API | Pod specs |
| Status updates | CRD status | JSON |
| Events | K8s events | Event objects |

### Event Types

| Event | Description |
|-------|-------------|
| `AgentPodsRecreated` | Recreated N agent pods |
| `AllocationFailed` | Could not allocate resources |
| `InstanceRelocated` | Moved to different cluster |

## Observability

### Instance Monitoring

```bash
# Check pending agents
kubectl get iei -o wide

# View pod status
kubectl get pods -l supernova.groq.io/model-instance=<name>
```

### Events

```bash
kubectl get events \
  --field-selector involvedObject.kind=InferenceEngineInstance \
  --sort-by='.lastTimestamp'
```

### Metrics

- Agent ready/pending counts
- Allocation success/failure rates
- Repair operation counts

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| CRDs | Versioned (`v1`) | Well-defined schemas |
| Labels | Standardized | `supernova.groq.io/*` |
| Events | K8s native | Standard event types |
| Tokenizers | NFS-based | HuggingFace compatible |
| Multi-cluster | Nebula coordination | Leader election |

## Customization & Extension

### Extension Points

1. **Tokenizers** - Add to `tokenizers/` directory
2. **Allocation** - Custom allocation strategies
3. **Repair** - Custom repair policies
4. **kubectl Plugins** - Extend `kubectl-fit`

### Tokenizer Configuration

```yaml
runtimes:
  default:
    hf_model_path: CohereForAI/c4ai-command-a-reasoning-08-2025-fp16
```

Adding new tokenizer:
1. Create directory with normalized model ID
2. Add `tokenizer_cache_info.json`
3. Add test case to `internal/tokenizer/tokenizer_test.go`

## Incident Recovery

### Loss of Nebula Cluster

Multi-regional leader election elects new Nebula. Update `--global-leader-elect-endpoint` if storage region lost.

### Loss of Supernova Cluster

Nebula assumes last observed replicas. Won't block other cluster deployments.

### Loss of Model Instance

Supernova attempts repair, then relocation within cluster. If impossible, Nebula reallocates to different cluster.

### Fast Rollbacks

Use dedicated mask targets in gitops repository. See [Notion Runbook](https://www.notion.so/groq/Fast-Supernova-Nebula-Rollbacks).

## Dynamo Equivalent

**Closest Match**: **Deployment (K8s Operator)**

| Aspect | Supernova | Dynamo Deployment |
|--------|-----------|-------------------|
| **Purpose** | Manage model instances | Manage DGD deployments |
| **CRDs** | `InferenceEngineInstance` | `DynamoGraphDeployment` |
| **Scope** | Per-cluster + global | Per-namespace |
| **Repair** | Auto-repair, relocate | K8s self-healing |
| **Multi-cluster** | Nebula coordination | Not supported |

**Key Differences**:
- Supernova has Nebula for global coordination; Dynamo is single-cluster
- Supernova manages agent/conductor pods; Dynamo manages frontend/worker pods
- Supernova has custom allocation logic; Dynamo uses K8s scheduling
- Supernova has built-in tokenizer management; Dynamo relies on backends

## Related Documentation

- [Gitops Repository](https://github.com/groq/gitops)
- [Inference Engine Instances](https://github.com/groq/inference-engine-instances)
- [Notion Runbook](https://www.notion.so/groq/Fast-Supernova-Nebula-Rollbacks)

## Tests

- **Unit Tests**: `internal/` test files
- **Integration Tests**: `test/` directory
- **Tokenizer Tests**: `internal/tokenizer/tokenizer_test.go`
- **Build Data**: `internal/cluster/testdata/builds/`
