# Backend SGLang Component Summary

## Overview

The SGLang backend wrapper integrates SGLang with Dynamo's distributed runtime. It provides a Python interface for running SGLang as prefill workers, decode workers, or aggregated workers, supporting disaggregated inference, multi-node deployments, embeddings, and multimodal processing.

## Location

- **Source**: `components/src/dynamo/sglang/`
- **Package**: `ai-dynamo` (part of main package)
- **Current Version**: 0.8.0

## Internal Dependencies

### Dynamo Python Modules

| Module | Usage |
|--------|-------|
| `dynamo.llm` | `ModelInput`, `ModelType`, `register_llm` |
| `dynamo.runtime` | `DistributedRuntime` |
| `dynamo.runtime.logging` | `configure_dynamo_logging()` |
| `dynamo.common.config_dump` | Configuration dumping |
| `dynamo.common.utils.endpoint_types` | Endpoint type parsing |

### External Python Dependencies

| Package | Usage |
|---------|-------|
| `sglang` | SGLang engine (`sgl.Engine`) |
| `uvloop` | High-performance async event loop |

### External Services

| Service | Required For | Notes |
|---------|--------------|-------|
| etcd | Worker registration | Via `register_llm` |
| NATS | KV events | Optional |

## Public Interface

### CLI Entry Point

```bash
python -m dynamo.sglang [args]
```

### Key CLI Arguments

**Model Configuration:**
- `--model-path` - Model name or path
- `--tokenizer-path` - Tokenizer path

**Worker Mode (DisaggregationMode):**
- `--disagg-mode prefill` - Prefill-only worker
- `--disagg-mode decode` - Decode-only worker
- (default) - Aggregated mode

**Dynamo Integration:**
- `--namespace` - Dynamo namespace
- `--component-name` - Component registration name
- `--store-kv` - etcd/file/mem
- `--request-plane` - tcp/http/nats

**Multi-node:**
- `--node-rank` - Node rank for multi-node deployments
- `--tp-size` - Tensor parallelism size
- `--dp-size` - Data parallelism size

### Exposed Dynamo Endpoints

| Endpoint | Worker Type | Description |
|----------|-------------|-------------|
| `{ns}.{component}.generate` | All | Main generation endpoint |
| `{ns}.prefill.generate` | Prefill | Prefill-specific endpoint |
| `{ns}.backend.generate` | Decode | Decode-specific endpoint |
| `{ns}.{component}.encode` | Embedding | Embedding endpoint |

### Handler Classes

| Class | Purpose |
|-------|---------|
| `DecodeWorkerHandler` | Handles decode/generation requests |
| `PrefillWorkerHandler` | Handles prefill-only requests |
| `EmbeddingWorkerHandler` | Handles embedding requests |
| `MultimodalWorkerHandler` | Multimodal generation |
| `MultimodalPrefillWorkerHandler` | Multimodal prefill |
| `MultimodalEncodeWorkerHandler` | Multimodal embedding |
| `MultimodalProcessorHandler` | Multimodal preprocessing |

## User/Developer Interaction

### 1. Standalone Aggregated Worker

```bash
python -m dynamo.sglang \
    --model-path Qwen/Qwen3-0.6B \
    --namespace my-ns
```

### 2. Disaggregated Prefill Worker

```bash
python -m dynamo.sglang \
    --model-path Qwen/Qwen3-0.6B \
    --disagg-mode prefill \
    --namespace my-ns
```

### 3. Multi-Node Deployment

```bash
# Node 0 (leader)
python -m dynamo.sglang --model-path MODEL --node-rank 0 --tp-size 8

# Node 1
python -m dynamo.sglang --model-path MODEL --node-rank 1 --tp-size 8
```

### 4. K8s Deployment (via DGD)

```yaml
SglangWorker:
  dynamoNamespace: my-deployment
  componentType: worker
  replicas: 2
  resources:
    limits:
      gpu: "1"
  extraPodSpec:
    mainContainer:
      image: nvcr.io/nvidia/ai-dynamo/sglang-runtime:0.6.1
      command: [python3, -m, dynamo.sglang]
      args:
        - --model-path=Qwen/Qwen3-0.6B
```

## Packaging & Containers

### Container

- **Dedicated container**: `nvcr.io/nvidia/ai-dynamo/sglang-runtime`
- **Base**: NVIDIA CUDA + SGLang
- **Path**: `/workspace/components/src/dynamo/sglang/`

### GPU Requirements

- Requires NVIDIA GPU with CUDA support
- Supports tensor parallelism
- Supports data parallelism
- Multi-node support

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format | Notes |
|-------|--------|--------|-------|
| **Requests** | Dynamo runtime | Dict with `token_ids` | Pre-tokenized |
| **CLI Arguments** | Startup | argparse | Model, mode, etc. |
| **Model Weights** | HuggingFace/Local | Safetensors/PyTorch | Auto-downloaded |

### Outputs

| Output | Destination | Format | Notes |
|--------|-------------|--------|-------|
| **Responses** | Dynamo runtime | Streaming dict | Token IDs, text |
| **Prometheus Metrics** | `/metrics` | Prometheus format | SGLang shared memory metrics |
| **Health Check** | Dynamo runtime | Pydantic model | `SglangHealthCheckPayload` |

### Health Check

```python
class SglangHealthCheckPayload(BaseModel):
    status: str
    model_name: str
    # ...

class SglangPrefillHealthCheckPayload(BaseModel):
    status: str
    model_name: str
    # ...
```

## Observability

### Prometheus Metrics

SGLang metrics exposed via shared memory:
- Request throughput
- Token latency
- Cache utilization

### Non-Leader Node Handling

For multi-node deployments, non-leader nodes (node_rank >= 1):
- Run scheduler processes only
- Don't handle requests directly
- Still expose metrics via Prometheus

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| CLI args | Many SGLang passthrough | Document Dynamo-specific args |
| Request format | Implicit | JSON schema |
| Response format | Implicit | JSON schema |
| Health check | `SglangHealthCheckPayload` | Standardize fields |
| Multi-node config | Manual | Document setup |

## Customization & Extension

### Extension Points

1. **Custom Handlers** - Subclass handler classes
2. **Disaggregation Modes** - `DisaggregationMode` enum
3. **Model Configuration** - SGLang args supported via passthrough

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| KV events | Different from vLLM | Use standalone router |
| Prefill routing | Requires manual router | See disagg_router.sh example |

## Related Documentation

- [SGLang Documentation](https://sgl-project.github.io/)
- [Backend Guide](docs/development/backend-guide.md)
- Example: `examples/backends/sglang/launch/disagg_router.sh`

## Tests

- Unit tests: `components/src/dynamo/sglang/tests/`
- E2E tests: `tests/router/test_router_e2e_with_sglang.py`
