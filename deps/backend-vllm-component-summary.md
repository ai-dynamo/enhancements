# Backend vLLM Component Summary

## Overview

The vLLM backend wrapper integrates vLLM (v1 engine) with Dynamo's distributed runtime. It provides a Python interface for running vLLM as prefill workers, decode workers, or aggregated workers within a Dynamo deployment, supporting disaggregated inference, KV cache events, and multimodal processing.

## Location

- **Source**: `components/src/dynamo/vllm/`
- **Package**: `ai-dynamo` (part of main package)
- **Current Version**: 0.8.0

## Internal Dependencies

### Dynamo Python Modules

| Module | Usage |
|--------|-------|
| `dynamo.llm` | `ModelInput`, `ModelType`, `ModelRuntimeConfig`, `register_llm`, `fetch_llm`, `ZmqKvEventPublisher` |
| `dynamo.runtime` | `DistributedRuntime` |
| `dynamo.runtime.logging` | `configure_dynamo_logging()` |
| `dynamo.common.config_dump` | Configuration dumping |
| `dynamo.common.utils.endpoint_types` | Endpoint type parsing |
| `dynamo.common.utils.prometheus` | Prometheus metrics registration |

### External Python Dependencies

| Package | Usage |
|---------|-------|
| `vllm` | v1 AsyncLLM engine, distributed KV events |
| `uvloop` | High-performance async event loop |
| `prometheus_client` | Metrics exposure |

### External Services

| Service | Required For | Notes |
|---------|--------------|-------|
| etcd | Worker registration | Via `register_llm` |
| NATS | KV events | Optional, for router awareness |
| ZMQ | KV event publishing | vLLM native events |

## Public Interface

### CLI Entry Point

```bash
python -m dynamo.vllm [args]
```

### Key CLI Arguments

**Model Configuration:**
- `--model` - Model name or path
- `--tokenizer` - Tokenizer name or path

**Worker Mode:**
- `--is-prefill-worker` - Run as prefill-only worker
- `--is-decode-worker` - Run as decode-only worker
- (default) - Aggregated mode

**Dynamo Integration:**
- `--namespace` - Dynamo namespace
- `--component-name` - Component registration name
- `--endpoint-name` - Endpoint registration name
- `--store-kv` - etcd/file/mem
- `--request-plane` - tcp/http/nats

**KV Events:**
- `--enable-kv-events` - Enable KV cache event publishing
- `--kv-events-publisher-type` - nats/zmq

### Exposed Dynamo Endpoints

| Endpoint | Worker Type | Description |
|----------|-------------|-------------|
| `{ns}.{component}.generate` | All | Main generation endpoint |
| `{ns}.prefill.generate` | Prefill | Prefill-specific endpoint |
| `{ns}.backend.generate` | Decode | Decode-specific endpoint |

### Handler Classes

| Class | Purpose |
|-------|---------|
| `DecodeWorkerHandler` | Handles decode/generation requests |
| `PrefillWorkerHandler` | Handles prefill-only requests |
| `MultimodalDecodeWorkerHandler` | Multimodal decode handling |
| `MultimodalPDWorkerHandler` | Multimodal prefill-decode |
| `EncodeWorkerHandler` | Embedding/encode requests |
| `ProcessorHandler` | Preprocessing for multimodal |

## User/Developer Interaction

### 1. Standalone Aggregated Worker

```bash
python -m dynamo.vllm \
    --model Qwen/Qwen3-0.6B \
    --namespace my-ns
```

### 2. Disaggregated Prefill Worker

```bash
python -m dynamo.vllm \
    --model Qwen/Qwen3-0.6B \
    --is-prefill-worker \
    --namespace my-ns
```

### 3. Disaggregated Decode Worker

```bash
python -m dynamo.vllm \
    --model Qwen/Qwen3-0.6B \
    --is-decode-worker \
    --namespace my-ns
```

### 4. K8s Deployment (via DGD)

```yaml
VllmDecodeWorker:
  dynamoNamespace: my-deployment
  componentType: worker
  subComponentType: decode
  replicas: 2
  resources:
    limits:
      gpu: "1"
  extraPodSpec:
    mainContainer:
      image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1
      command: [python3, -m, dynamo.vllm]
      args:
        - --model=Qwen/Qwen3-0.6B
        - --is-decode-worker
```

## Packaging & Containers

### Container

- **Dedicated container**: `nvcr.io/nvidia/ai-dynamo/vllm-runtime`
- **Base**: NVIDIA CUDA + vLLM
- **Path**: `/workspace/components/src/dynamo/vllm/`

### GPU Requirements

- Requires NVIDIA GPU with CUDA support
- Supports tensor parallelism (`--tensor-parallel-size`)
- Supports pipeline parallelism

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format | Notes |
|-------|--------|--------|-------|
| **Requests** | Dynamo runtime | Dict with `token_ids` | Pre-tokenized by frontend |
| **CLI Arguments** | Startup | argparse | Model, worker type, etc. |
| **Model Weights** | HuggingFace/Local | Safetensors/PyTorch | Auto-downloaded |

### Outputs

| Output | Destination | Format | Notes |
|--------|-------------|--------|-------|
| **Responses** | Dynamo runtime | Streaming dict | Token IDs, text, logprobs |
| **KV Events** | NATS/ZMQ | Protobuf | Block create/delete |
| **Prometheus Metrics** | `/metrics` | Prometheus format | vLLM engine metrics |
| **Health Check** | Dynamo runtime | Pydantic model | `VllmHealthCheckPayload` |

### Request/Response Format

**Request:**
```python
{
    "token_ids": [1, 2, 3, ...],
    "sampling_options": {"temperature": 0.7, "max_tokens": 100},
    "stop_conditions": {"stop_strings": ["<|end|>"]},
    "annotations": [...],
}
```

**Response (streaming):**
```python
{
    "token_ids": [4, 5, 6],
    "text": "Hello",
    "finish_reason": None,  # or "stop", "length"
    "disaggregated_params": {...},  # For prefill workers
}
```

## Observability

### Prometheus Metrics

Exposes vLLM native metrics plus Dynamo-specific metrics:
- Engine utilization
- Request latency
- Token throughput
- KV cache utilization

### Health Check

```python
class VllmHealthCheckPayload(BaseModel):
    status: str  # "healthy", "unhealthy"
    model_name: str
    gpu_memory_used: float
    # ...
```

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| CLI args | Many vLLM passthrough | Document Dynamo-specific args |
| Request format | Implicit | JSON schema |
| Response format | Implicit | JSON schema |
| KV event format | vLLM/Dynamo mix | Standardize, version |
| Health check | `VllmHealthCheckPayload` | Standardize fields |
| Metrics | vLLM + Dynamo | Document all, prefixes |

## Customization & Extension

### Extension Points

1. **Custom Handlers** - Subclass `DecodeWorkerHandler` or `PrefillWorkerHandler`
2. **Custom Sampling** - Pass `sampling_options` in requests
3. **Model Configuration** - All vLLM args supported via passthrough

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Custom preprocessing | Frontend handles | Use custom frontend |
| Engine modifications | vLLM internal | Fork vLLM |

## Related Documentation

- [vLLM Documentation](https://docs.vllm.ai/)
- [Backend Guide](docs/development/backend-guide.md)
- [Disaggregated Serving](docs/router/kv_cache_routing.md)

## Tests

- Unit tests: `components/src/dynamo/vllm/tests/`
- E2E tests: `tests/router/test_router_e2e_with_vllm.py`
