# Backend TensorRT-LLM Component Summary

## Overview

The TensorRT-LLM (TRT-LLM) backend wrapper integrates NVIDIA TensorRT-LLM with Dynamo's distributed runtime. It provides a Python interface for running optimized TensorRT engines as workers, supporting disaggregated inference, KV cache events, multimodal processing, and NIXL-based KV transfer.

## Location

- **Source**: `components/src/dynamo/trtllm/`
- **Package**: `ai-dynamo` (part of main package)
- **Current Version**: 0.8.0

## Internal Dependencies

### Dynamo Python Modules

| Module | Usage |
|--------|-------|
| `dynamo.llm` | `ModelInput`, `ModelType`, `ModelRuntimeConfig`, `register_llm`, `ZmqKvEventPublisher` |
| `dynamo.runtime` | `DistributedRuntime` |
| `dynamo.runtime.logging` | `configure_dynamo_logging()`, `map_dyn_log_to_tllm_level()` |
| `dynamo.nixl_connect` | NIXL KV transfer integration |
| `dynamo.common.config_dump` | Configuration dumping |
| `dynamo.common.utils.prometheus` | Prometheus metrics registration |

### External Python Dependencies

| Package | Usage |
|---------|-------|
| `tensorrt_llm` | TRT-LLM LLMAPI engine |
| `transformers` | Model config, tokenizer |
| `torch` | CUDA device management |
| `uvloop` | High-performance async event loop |
| `prometheus_client` | Metrics exposure |

### External Services

| Service | Required For | Notes |
|---------|--------------|-------|
| etcd | Worker registration | Via `register_llm` |
| NATS | KV events | Optional |
| ZMQ | KV event publishing | Default |

## Public Interface

### CLI Entry Point

```bash
python -m dynamo.trtllm [args]
```

### Key CLI Arguments

**Model Configuration:**
- `--model` - Model name or path
- `--tokenizer` - Tokenizer name or path
- `--engine-dir` - Pre-built TRT engine directory

**Worker Mode (DisaggregationMode):**
- `--disagg-mode prefill` - Prefill-only worker
- `--disagg-mode decode` - Decode-only worker
- (default) - Aggregated mode

**Dynamo Integration:**
- `--namespace` - Dynamo namespace
- `--component-name` - Component registration name
- `--store-kv` - etcd/file/mem
- `--request-plane` - tcp/http/nats

**TRT-LLM Specific:**
- `--max-batch-size` - Maximum batch size
- `--max-num-tokens` - Maximum tokens per batch
- `--kv-cache-free-gpu-mem-fraction` - KV cache memory allocation

### Exposed Dynamo Endpoints

| Endpoint | Worker Type | Description |
|----------|-------------|-------------|
| `{ns}.{component}.generate` | All | Main generation endpoint |
| `{ns}.prefill.generate` | Prefill | Prefill-specific endpoint |
| `{ns}.backend.generate` | Decode | Decode-specific endpoint |

### Handler Classes (via Factory)

```python
class RequestHandlerFactory:
    @staticmethod
    def create_handler(config: RequestHandlerConfig) -> RequestHandler
```

| Handler Type | Purpose |
|--------------|---------|
| Aggregated | Combined prefill+decode |
| Prefill | Prefill-only processing |
| Decode | Decode-only generation |
| Multimodal | Vision-language models |

### Engine Classes

| Class | Purpose |
|-------|---------|
| `TensorRTLLMEngine` | Wrapper around TRT-LLM LLMAPI |
| `Backend` | Backend type enum |

## User/Developer Interaction

### 1. Standalone Aggregated Worker

```bash
python -m dynamo.trtllm \
    --model Qwen/Qwen3-0.6B \
    --namespace my-ns
```

### 2. Pre-built Engine

```bash
python -m dynamo.trtllm \
    --engine-dir /path/to/trt_engine \
    --tokenizer Qwen/Qwen3-0.6B \
    --namespace my-ns
```

### 3. Disaggregated Prefill Worker

```bash
python -m dynamo.trtllm \
    --model Qwen/Qwen3-0.6B \
    --disagg-mode prefill \
    --namespace my-ns
```

### 4. K8s Deployment (via DGD)

```yaml
TrtllmWorker:
  dynamoNamespace: my-deployment
  componentType: worker
  replicas: 2
  resources:
    limits:
      gpu: "1"
  extraPodSpec:
    mainContainer:
      image: nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime:0.6.1
      command: [python3, -m, dynamo.trtllm]
      args:
        - --model=Qwen/Qwen3-0.6B
```

## Packaging & Containers

### Container

- **Dedicated container**: `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime`
- **Base**: NVIDIA TensorRT-LLM
- **Path**: `/workspace/components/src/dynamo/trtllm/`

### GPU Requirements

- Requires NVIDIA GPU with TensorRT support
- Supports tensor parallelism
- Supports pipeline parallelism
- Optimized for NVIDIA hardware (H100, A100, etc.)

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format | Notes |
|-------|--------|--------|-------|
| **Requests** | Dynamo runtime | Dict with `token_ids` | Pre-tokenized |
| **CLI Arguments** | Startup | argparse | Model, mode, etc. |
| **Model/Engine** | Local/HuggingFace | TRT engine or weights | Auto-built if needed |

### Outputs

| Output | Destination | Format | Notes |
|--------|-------------|--------|-------|
| **Responses** | Dynamo runtime | Streaming dict | Token IDs, text |
| **KV Events** | NATS/ZMQ | Protobuf | Block create/delete |
| **Prometheus Metrics** | Registry | Prometheus format | TRT-LLM metrics |
| **Health Check** | Dynamo runtime | Pydantic model | `TrtllmHealthCheckPayload` |

### Logging Integration

TRT-LLM log level automatically mapped from `DYN_LOG`:
```python
# DYN_LOG=info -> TLLM_LOG_LEVEL=INFO
# Set DYN_SKIP_TRTLLM_LOG_FORMATTING=1 to disable
```

## Observability

### Prometheus Metrics

TRT-LLM MetricsCollector integration:
- Engine statistics
- KV cache utilization
- Request latency
- Token throughput

### Health Check

```python
class TrtllmHealthCheckPayload(BaseModel):
    status: str
    model_name: str
    # ...
```

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| CLI args | Many TRT-LLM passthrough | Document Dynamo-specific args |
| Engine building | Implicit | Document build process |
| Request format | Implicit | JSON schema |
| Response format | Implicit | JSON schema |
| KV event format | ZMQ default | Standardize, version |
| Health check | `TrtllmHealthCheckPayload` | Standardize fields |

## Customization & Extension

### Extension Points

1. **Custom Request Handlers** - Use `RequestHandlerFactory`
2. **Engine Configuration** - TRT-LLM LLMAPI options
3. **Logits Processing** - Custom logits processors in `logits_processing/`

### Configuration Classes

```python
class Config:
    # Dynamo args
    namespace: str
    component_name: str
    # TRT-LLM args
    model: str
    engine_dir: str
    max_batch_size: int
    # ...
```

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Engine building | Requires pre-build or auto-build time | Pre-build engines |
| Custom ops | TRT-LLM limitations | Use TRT-LLM plugins |

## Related Documentation

- [TensorRT-LLM Documentation](https://nvidia.github.io/TensorRT-LLM/)
- [Backend Guide](docs/development/backend-guide.md)
- Examples: `examples/tensorrt_llm/`, `examples/backends/trtllm/`

## Tests

- Unit tests: `components/src/dynamo/trtllm/tests/`
- E2E tests: `tests/router/test_router_e2e_with_trtllm.py`
