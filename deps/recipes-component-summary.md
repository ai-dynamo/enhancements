# Recipes Component Summary

## Overview

The Recipes component provides production-ready Kubernetes deployment templates for various LLM models across different inference backends. Each recipe includes complete DynamoGraphDeployment manifests, model download configurations, and optional benchmark setups, allowing users to deploy optimized inference pipelines with minimal configuration.

## Location

- **Source**: `recipes/`
- **Documentation**: `recipes/README.md`
- **Contributing Guide**: `recipes/CONTRIBUTING.md`

## Available Recipes

| Model | Frameworks | Deployment Modes | GPU Requirements |
|-------|------------|------------------|------------------|
| **Llama-3-70B** | vLLM | Aggregated, Disagg Single-Node, Disagg Multi-Node | 4-16x H100/H200 |
| **Qwen3-32B-FP8** | TensorRT-LLM | Aggregated, Disaggregated | 4-8x GPU |
| **Qwen3-32B** | vLLM | Aggregated (round-robin), Disagg (KV-router) | 16x H200 |
| **GPT-OSS-120B** | TensorRT-LLM | Aggregated | 4x GB200 |
| **Qwen3-235B-A22B-FP8** | TensorRT-LLM | Aggregated, Disaggregated | 4-8x GPU |
| **DeepSeek-R1** | SGLang, TensorRT-LLM | Disaggregated (8/16/32+ GPUs) | 8-36+ H200/GB200 |

## Internal Dependencies

### Dynamo Components

| Component | Usage |
|-----------|-------|
| Kubernetes Operator | DGD CRD reconciliation |
| Frontend | HTTP API endpoint |
| Backend Workers | vLLM, SGLang, TRT-LLM runtimes |
| NATS | Inter-component messaging |
| etcd | Service discovery |

### External Dependencies

| Dependency | Purpose |
|------------|---------|
| Kubernetes StorageClass | PVC provisioning |
| HuggingFace Hub | Model downloads |
| Container registries | Runtime image pulls |
| GPU drivers | NVIDIA GPU access |

### Optional Dependencies

| Dependency | Purpose |
|------------|---------|
| GAIE (Inference Gateway) | Advanced routing and load balancing |
| AIPerf | Performance benchmarking |

## Recipe Structure

### Standard Layout

```
<model-name>/
├── README.md                    # Model-specific notes
├── model-cache/
│   ├── model-cache.yaml         # PersistentVolumeClaim
│   └── model-download.yaml      # Model download Job
└── <framework>/                 # vllm, sglang, trtllm
    └── <deployment-mode>/       # agg, disagg, disagg-kv-router
        ├── deploy.yaml          # DynamoGraphDeployment
        ├── perf.yaml            # Benchmark job (optional)
        └── gaie/                # Inference Gateway (optional)
            └── k8s-manifests/
                ├── model/       # InferenceModel, InferencePool
                ├── rbac/        # Service accounts, roles
                └── epp/         # Endpoint Protection Policy
```

### Key File Types

| File | Purpose |
|------|---------|
| `deploy.yaml` | DynamoGraphDeployment CRD manifest |
| `model-cache.yaml` | PVC for model storage |
| `model-download.yaml` | K8s Job for HuggingFace download |
| `perf.yaml` | AIPerf benchmark configuration |

## Public Interface

### deploy.yaml Structure

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: <deployment-name>
spec:
  backendFramework: vllm|sglang|trtllm
  pvcs:
    - name: model-cache
      create: false
  services:
    Frontend:
      componentType: frontend
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/<framework>-runtime:tag
    VllmWorker:  # or SglangWorker, TrtllmWorker
      componentType: worker
      resources:
        limits:
          gpu: "N"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/<framework>-runtime:tag
          args:
            - python3 -m dynamo.<framework> --model <model>
```

### Customization Points

| Parameter | Location | Description |
|-----------|----------|-------------|
| Model path | `MODEL_PATH` env or CLI arg | Model identifier or path |
| Tensor parallelism | `--tensor-parallel-size N` | GPU parallelism |
| Data parallelism | `--data-parallel-size N` | Replica parallelism |
| GPU memory | `--gpu-memory-utilization 0.90` | KV cache allocation |
| Block size | `--block-size 64` | KV cache granularity |
| Router mode | `--router-mode kv|round-robin` | Request routing |
| Replicas | `replicas: N` | Worker count |

## User/Developer Interaction

### 1. Standard Deployment Workflow

```bash
# 1. Create namespace and secrets
kubectl create namespace ${NAMESPACE}
kubectl create secret generic hf-token-secret \
  --from-literal=HF_TOKEN="your-token" \
  -n ${NAMESPACE}

# 2. Configure storage (edit storageClassName)
vi <model>/model-cache/model-cache.yaml

# 3. Create PVC and download model
kubectl apply -f <model>/model-cache/
kubectl wait --for=condition=Complete job/model-download -n ${NAMESPACE}

# 4. Deploy model service
kubectl apply -f <model>/<framework>/<mode>/deploy.yaml -n ${NAMESPACE}
kubectl wait --for=condition=ready pod -l nvidia.com/dynamo-graph-deployment-name=<name>

# 5. Test deployment
kubectl port-forward svc/<name>-frontend 8000:8000 -n ${NAMESPACE}
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"<model>","messages":[{"role":"user","content":"Hello"}]}'
```

### 2. Run Benchmarks

```bash
# Apply benchmark job
kubectl apply -f <model>/<framework>/<mode>/perf.yaml -n ${NAMESPACE}

# Monitor progress
kubectl logs -f job/<benchmark-job-name> -n ${NAMESPACE}
```

### 3. GAIE Integration

```bash
# Deploy Inference Gateway components
kubectl apply -R -f <model>/<framework>/<mode>/gaie/k8s-manifests -n ${NAMESPACE}
```

## Packaging & Containers

### Container Images Used

| Framework | Image |
|-----------|-------|
| vLLM | `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1+` |
| SGLang | `nvcr.io/nvidia/ai-dynamo/sglang-runtime:0.6.1+` |
| TensorRT-LLM | `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:0.6.1+` |

### Storage Requirements

| PVC | Typical Size | Purpose |
|-----|--------------|---------|
| Model cache | 100Gi | Model weights |
| Compilation cache | 50Gi | vLLM compiled graphs |
| Performance results | 50Gi | Benchmark outputs |

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| Model identifier | HuggingFace Hub | Model name or path |
| HF token | K8s Secret | String |
| Storage class | Cluster config | StorageClass name |

### Outputs

| Output | Destination | Format |
|--------|-------------|--------|
| HTTP endpoint | Frontend Service | OpenAI-compatible API |
| Benchmark results | PVC | JSON/CSV metrics |

### Environment Variables

| Variable | Purpose | Example |
|----------|---------|---------|
| `HF_TOKEN` | HuggingFace authentication | From secret |
| `MODEL_PATH` | Model location | `/mnt/models/Qwen/Qwen3-32B` |
| `SERVED_MODEL_NAME` | API model name | `Qwen/Qwen3-32B` |

## Observability

### Health Endpoints

| Endpoint | Purpose |
|----------|---------|
| `/health` | Liveness check |
| `/v1/models` | Model availability |

### Debugging

```bash
# Check deployment status
kubectl describe dgd <name> -n ${NAMESPACE}

# View worker logs
kubectl logs -l app.kubernetes.io/component=worker -n ${NAMESPACE}

# Check events
kubectl get events -n ${NAMESPACE} --sort-by='.lastTimestamp'
```

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| Recipe validation | CONTRIBUTING.md | Automated CI validation |
| Version compatibility | Implicit | Document version matrix |
| Storage requirements | Per-recipe | Standardize sizing guide |
| GPU requirements | README only | Structured metadata |
| Testing | Manual | Automated recipe testing |
| Documentation | Per-recipe README | Consistent format |

## Customization & Extension

### Contributing New Recipes

From `CONTRIBUTING.md`:

1. Create model directory: `recipes/<model>/`
2. Add framework directory: `recipes/<model>/<framework>/`
3. Add deployment mode: `recipes/<model>/<framework>/<deployment>/`
4. **Required**: `deploy.yaml` must exist
5. **Recommended**: `perf.yaml` for benchmarking
6. **Required**: Model cache setup files
7. Verify on target hardware

### Validation Requirements

```bash
# All recipes must have:
- Model directory in recipes/<model>/
- Framework: vllm, sglang, or trtllm
- deploy.yaml in deployment directory
- Model cache configuration
```

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Multi-tenant | Single namespace | Deploy per namespace |
| Storage class | User-configured | Document requirements |
| GPU types | Hardware-specific | Use appropriate recipe |

## Related Documentation

- [Kubernetes Installation](docs/kubernetes/README.md)
- [Create Deployment](docs/kubernetes/deployment/create_deployment.md)
- [vLLM Backend](docs/backends/vllm/README.md)
- [TensorRT-LLM Backend](docs/backends/trtllm/README.md)
- [Inference Gateway](deploy/inference-gateway/README.md)

## Tests

- Recipe validation: `recipes/CONTRIBUTING.md` guidelines
- Integration tests: Deploy and verify each recipe
- Benchmark tests: `perf.yaml` configurations
