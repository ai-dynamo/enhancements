# Examples Component Summary

## Overview

The Examples component provides comprehensive tutorials and reference implementations demonstrating Dynamo's capabilities. Examples range from basic quickstart guides to advanced deployment patterns, covering all supported backends (vLLM, SGLang, TensorRT-LLM), deployment modes (aggregated, disaggregated), and cloud platforms (EKS, AKS, ECS).

## Location

- **Primary**: `examples/`
- **Python Bindings Examples**: `lib/bindings/python/examples/`
- **Runtime Examples**: `lib/runtime/examples/`
- **Documentation Link**: `docs/examples/` (symlink)

## Example Categories

### Basics & Tutorials (`examples/basics/`)

| Example | Description | Requirements |
|---------|-------------|--------------|
| `quickstart/` | Simplest aggregated serving | 1 GPU |
| `disaggregated_serving/` | Prefill/decode separation | 2+ GPUs |
| `multinode/` | Distributed with KV routing | 2+ replicas, 2 GPUs each |

### Backend Examples (`examples/backends/`)

| Backend | Modes | Features |
|---------|-------|----------|
| `vllm/` | agg, agg_router, disagg, disagg_router | LoRA, KV routing, planner |
| `sglang/` | agg, agg_router, disagg | SLURM support |
| `trtllm/` | Multiple templates | Multi-node, UCX, NIXL |
| `mocker/` | agg, disagg | Testing mock engine |

### Deployment Examples (`examples/deployments/`)

| Platform | Status | Description |
|----------|--------|-------------|
| `EKS/` | Complete | AWS Kubernetes |
| `AKS/` | Complete | Azure Kubernetes |
| `ECS/` | Complete | AWS Container Service |
| `GKE/` | Coming Soon | Google Kubernetes |
| `router_standalone/` | Complete | Standalone router patterns |

### Custom Backend Examples (`examples/custom_backend/`)

| Example | Purpose |
|---------|---------|
| `hello_world/` | Minimal Dynamo service |
| `cancellation/` | Request cancellation handling |
| `nim_backend_mock/` | Metrics collection interface |

### Specialized Examples

| Example | Purpose |
|---------|---------|
| `multimodal/` | Vision and audio processing |
| `llm/` | Advanced LLM patterns |

## Internal Dependencies

### Infrastructure Requirements

| Component | Purpose | Setup |
|-----------|---------|-------|
| etcd | Service discovery | `docker compose up` |
| NATS | Message broker | `docker compose up` |
| CUDA GPU | LLM inference | Hardware |

### Software Requirements

| Requirement | Version | Notes |
|-------------|---------|-------|
| Python | 3.9+ | Client scripts |
| Docker | Latest | Containerized services |
| Kubernetes | 1.25+ | Cloud deployments |
| CUDA | 11.8+ | GPU support |

### Installation

```bash
# Basic
pip install ai-dynamo

# With framework
pip install ai-dynamo[sglang]
pip install ai-dynamo[vllm]
pip install ai-dynamo[tensorrt-llm]
```

## Example Structure

### Standard Layout

```
example_name/
├── README.md              # Setup and usage
├── server.py              # Backend/worker implementation
├── client.py              # Client script (if applicable)
├── deploy/                # Kubernetes files
│   ├── docker-compose.yml # Infrastructure
│   ├── agg.yaml           # Aggregated deployment
│   ├── disagg.yaml        # Disaggregated deployment
│   └── *.yaml             # Alternatives
└── components/            # Custom implementations
```

### Execution Flow

```bash
# 1. Start infrastructure
docker compose -f deploy/docker-compose.yml up -d

# 2. Launch backend worker
python -m dynamo.<framework> --model <model> [options]

# 3. Launch frontend
python -m dynamo.frontend --http-port 8000 [--router-mode kv]

# 4. Send requests
curl -X POST http://localhost:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model": "...", "messages": [...]}'
```

## Public Interface

### Worker Decorator

```python
from dynamo.runtime import DistributedRuntime, dynamo_worker

@dynamo_worker()
async def worker(runtime: DistributedRuntime):
    component = runtime.namespace("my-ns").component("my-component")
    endpoint = component.endpoint("generate")

    async def handler(request):
        yield {"result": "response"}

    await endpoint.serve_endpoint(handler)
```

### Endpoint Decorator

```python
from dynamo.runtime import dynamo_endpoint

@dynamo_endpoint(Endpoint)
async def generate(self, request):
    async for token in self.model.generate(request):
        yield token
```

### Cancellation Handling

```python
async def handler(request, context):
    for i in range(100):
        if context.is_stopped():
            break
        yield {"token": i}
        await asyncio.sleep(0.1)
```

## User/Developer Interaction

### 1. Quickstart Example

```bash
# Start infrastructure
cd examples/basics/quickstart
docker compose -f deploy/docker-compose.yml up -d

# Start vLLM backend
python -m dynamo.vllm \
  --model Qwen/Qwen3-0.6B \
  --namespace example

# Start frontend (another terminal)
python -m dynamo.frontend \
  --http-port 8000 \
  --namespace example

# Test
curl http://localhost:8000/v1/chat/completions \
  -d '{"model":"Qwen/Qwen3-0.6B","messages":[{"role":"user","content":"Hi"}]}'
```

### 2. Disaggregated Serving

```bash
# Requires 2+ GPUs
cd examples/basics/disaggregated_serving

# Start prefill worker
CUDA_VISIBLE_DEVICES=0 python -m dynamo.vllm \
  --model Qwen/Qwen3-0.6B \
  --disagg-mode prefill

# Start decode worker
CUDA_VISIBLE_DEVICES=1 python -m dynamo.vllm \
  --model Qwen/Qwen3-0.6B \
  --disagg-mode decode

# Start frontend
python -m dynamo.frontend --http-port 8000
```

### 3. Custom Backend (Hello World)

```python
# server.py
from dynamo.runtime import DistributedRuntime, dynamo_worker, dynamo_endpoint

@dynamo_worker()
async def worker(runtime: DistributedRuntime):
    endpoint = runtime.namespace("hello").component("world").endpoint("greet")

    @dynamo_endpoint(endpoint)
    async def greet(request):
        yield {"message": f"Hello, {request.get('name', 'World')}!"}

    await endpoint.serve_endpoint(greet)

if __name__ == "__main__":
    import asyncio
    asyncio.run(worker())
```

### 4. Kubernetes Deployment

```yaml
# deploy/agg.yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: quickstart
spec:
  backendFramework: vllm
  services:
    Frontend:
      componentType: frontend
    VllmWorker:
      componentType: worker
      resources:
        limits:
          gpu: "1"
```

## Packaging & Containers

### Container Images

All examples can run in:
| Image | Use Case |
|-------|----------|
| `nvcr.io/nvidia/ai-dynamo/vllm-runtime` | vLLM examples |
| `nvcr.io/nvidia/ai-dynamo/sglang-runtime` | SGLang examples |
| `nvcr.io/nvidia/ai-dynamo/tensorrtllm-runtime` | TRT-LLM examples |

### Docker Compose Services

```yaml
# deploy/docker-compose.yml
services:
  nats:
    image: nats:latest
    ports:
      - "4222:4222"
  etcd:
    image: quay.io/coreos/etcd:v3.5.0
    ports:
      - "2379:2379"
```

## Service Interface & I/O Contract

### HTTP Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Chat completions (OpenAI compatible) |
| `/v1/completions` | POST | Text completions |
| `/v1/models` | GET | List available models |
| `/health` | GET | Health check |
| `/metrics` | GET | Prometheus metrics |

### Request Format

```json
{
  "model": "Qwen/Qwen3-0.6B",
  "messages": [
    {"role": "user", "content": "Hello"}
  ],
  "stream": true,
  "max_tokens": 256
}
```

### Response Format (Streaming)

```json
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion.chunk",
  "choices": [
    {
      "delta": {"content": "Hi"},
      "index": 0
    }
  ]
}
```

## Observability

### Logging

```bash
# Set log level
export DYN_LOG=debug

# Run with verbose logging
python -m dynamo.vllm --model ... 2>&1 | tee worker.log
```

### Health Checks

```bash
# Check frontend health
curl http://localhost:8000/health

# Check metrics
curl http://localhost:8000/metrics
```

### Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `DYN_LOG` | Log level | `info` |
| `NATS_SERVER` | NATS connection | `nats://localhost:4222` |
| `ETCD_ENDPOINTS` | etcd connection | `localhost:2379` |
| `CUDA_VISIBLE_DEVICES` | GPU selection | All GPUs |

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| README format | Various | Standardize template |
| Environment setup | Per-example | Unified setup script |
| Error handling | Implicit | Document common errors |
| Testing | Manual | Automated example testing |
| Dependencies | requirements.txt | Unified dependency management |
| Versioning | Implicit | Version compatibility matrix |

## Customization & Extension

### Extension Points

1. **Custom Handlers** - Implement request handlers
2. **Custom Backends** - Add new inference engines
3. **Custom Routing** - Implement routing strategies
4. **Custom Metrics** - Add application metrics

### Adding New Examples

1. Create directory: `examples/<category>/<example>/`
2. Add `README.md` with setup instructions
3. Add implementation files
4. Add `deploy/` directory with K8s/compose files
5. Test locally and in containers
6. Update main `examples/README.md`

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Windows | Linux containers required | Use WSL2 or VM |
| CPU-only | Most require GPU | Use mocker backend |
| Model size | Memory constraints | Use smaller models |

## Related Documentation

- [Quickstart Guide](docs/quickstart.md)
- [Backend Guide](docs/development/backend-guide.md)
- [Runtime Guide](docs/development/runtime-guide.md)
- [Kubernetes Deployment](docs/kubernetes/README.md)

## Tests

- Example validation: README-based verification
- Integration tests: `tests/integration/`
- E2E tests: `tests/e2e/`
- CI: Example smoke tests in GitHub Actions
