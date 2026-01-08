# llm-d Inference Simulator Component Summary

## Overview

The llm-d Inference Simulator is a lightweight vLLM simulator for testing inference scheduling without requiring actual GPU accelerators. It mimics vLLM's behavior including request queuing, token generation timing, and metrics exposition, enabling development and testing of the llm-d platform without expensive hardware.

## Location

- **Repository**: `https://github.com/llm-d/llm-d-inference-sim` (public)
- **Language**: Go
- **Stars**: 77+
- **Description**: Lightweight vLLM simulator for testing

## Internal Dependencies

### Go Dependencies

| Category | Dependencies |
|----------|--------------|
| **HTTP Server** | net/http, gorilla/mux |
| **Metrics** | Prometheus client |
| **Configuration** | viper, cobra |
| **Logging** | zap, logrus |
| **Testing** | Ginkgo, Gomega |

### Simulated Behavior

| Aspect | Simulation |
|--------|------------|
| Request queuing | Configurable queue depth |
| Token generation | Configurable latency |
| TTFT | Time-to-first-token simulation |
| ITL | Inter-token latency simulation |
| Metrics | vLLM-compatible metrics |

## Module Structure

### Key Components

| Component | Purpose |
|-----------|---------|
| **HTTP Server** | OpenAI-compatible API |
| **Request Queue** | Simulates vLLM queue |
| **Token Generator** | Simulates token streaming |
| **Metrics Server** | Prometheus metrics |
| **Config Manager** | Runtime configuration |

### Simulated Endpoints

| Endpoint | Description |
|----------|-------------|
| `/v1/chat/completions` | Chat API (streaming) |
| `/v1/completions` | Completion API |
| `/health` | Health check |
| `/metrics` | Prometheus metrics |

## Public Interface

### Configuration

```yaml
# Simulator configuration
server:
  port: 8000
  metricsPort: 9090

simulation:
  # Time-to-first-token (ms)
  ttft:
    mean: 50
    stddev: 10

  # Inter-token latency (ms)
  itl:
    mean: 10
    stddev: 2

  # Request queue
  queue:
    maxDepth: 100
    processingRate: 50  # requests/second

  # Token generation
  tokens:
    minLength: 10
    maxLength: 500

model:
  name: "llama-70b-simulated"
  maxContextLength: 8192
```

### CLI Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `--port` | HTTP server port | 8000 |
| `--metrics-port` | Prometheus port | 9090 |
| `--config` | Config file path | `config.yaml` |
| `--ttft-mean` | Mean TTFT (ms) | 50 |
| `--itl-mean` | Mean ITL (ms) | 10 |
| `--queue-depth` | Max queue depth | 100 |

### API Response Format

```json
{
  "id": "chatcmpl-sim-123",
  "object": "chat.completion.chunk",
  "created": 1234567890,
  "model": "llama-70b-simulated",
  "choices": [
    {
      "index": 0,
      "delta": {
        "content": "Hello"
      },
      "finish_reason": null
    }
  ]
}
```

## User/Developer Interaction

### 1. Run Simulator

```bash
# Build
go build -o inference-sim ./cmd/simulator

# Run with defaults
./inference-sim

# Run with custom config
./inference-sim --config my-config.yaml

# Run with CLI overrides
./inference-sim --ttft-mean 100 --itl-mean 20
```

### 2. Docker

```bash
# Build image
docker build -t llm-d-inference-sim .

# Run container
docker run -p 8000:8000 -p 9090:9090 llm-d-inference-sim
```

### 3. Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inference-sim
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: simulator
          image: llm-d-inference-sim:latest
          ports:
            - containerPort: 8000
            - containerPort: 9090
          env:
            - name: TTFT_MEAN
              value: "50"
            - name: ITL_MEAN
              value: "10"
```

### 4. Test with curl

```bash
# Chat completion
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-70b-simulated",
    "messages": [{"role": "user", "content": "Hello"}],
    "stream": true
  }'

# Health check
curl http://localhost:8000/health

# Metrics
curl http://localhost:9090/metrics
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

### Container Image

```dockerfile
FROM golang:1.21 AS builder
WORKDIR /app
COPY . .
RUN go build -o inference-sim ./cmd/simulator

FROM gcr.io/distroless/static:nonroot
COPY --from=builder /app/inference-sim /
ENTRYPOINT ["/inference-sim"]
```

### Deployment Options

- Standalone binary
- Docker container
- Kubernetes Deployment
- Part of llm-d test environment

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| HTTP requests | Clients | OpenAI API |
| Configuration | File/env | YAML/env vars |

### Outputs

| Output | Destination | Format |
|--------|-------------|--------|
| Completions | HTTP response | SSE/JSON |
| Metrics | Prometheus | Exposition |
| Logs | stdout | Structured |

### Simulated Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `vllm_request_queue_length` | Gauge | Current queue depth |
| `vllm_request_latency_seconds` | Histogram | Request latency |
| `vllm_tokens_generated_total` | Counter | Total tokens |
| `vllm_active_requests` | Gauge | In-flight requests |

## Observability

### Prometheus Metrics

Compatible with vLLM metrics format:
- Request latency histograms
- Queue depth gauges
- Token throughput counters
- Error rates

### Logging

```bash
# Set log level
./inference-sim --log-level debug

# JSON logging
./inference-sim --log-format json
```

### Health Endpoint

```bash
curl http://localhost:8000/health
# {"status": "healthy", "queue_depth": 5, "uptime": "1h30m"}
```

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| API | OpenAI-compatible | Matches vLLM |
| Metrics | vLLM-compatible | Prometheus format |
| Configuration | YAML/CLI | Flexible |
| Deployment | Container-ready | K8s manifests |
| Documentation | README | Basic |

## Customization & Extension

### Extension Points

1. **Custom Latency Models** - Statistical distributions
2. **Custom Metrics** - Additional Prometheus metrics
3. **Request Patterns** - Load simulation
4. **Error Injection** - Fault testing

### Configuration Profiles

```yaml
# profiles/high-latency.yaml
simulation:
  ttft:
    mean: 200
    stddev: 50
  itl:
    mean: 50
    stddev: 10

# profiles/low-latency.yaml
simulation:
  ttft:
    mean: 20
    stddev: 5
  itl:
    mean: 5
    stddev: 1
```

### Error Injection

```yaml
# Enable error injection
errorInjection:
  enabled: true
  rate: 0.05  # 5% error rate
  types:
    - timeout
    - serverError
    - rateLimited
```

## Dynamo Equivalent

**Closest Match**: **Mocker Backend**

| Aspect | llm-d Inference Sim | Dynamo Mocker |
|--------|---------------------|---------------|
| **Purpose** | Test without GPUs | Test without GPUs |
| **API** | OpenAI-compatible | OpenAI-compatible |
| **Metrics** | vLLM-compatible | Dynamo metrics |
| **Configuration** | YAML/CLI | CLI args |
| **Language** | Go | Python |

**Key Differences**:
- llm-d Sim focuses on vLLM simulation; Dynamo Mocker is simpler
- llm-d Sim has configurable latency distributions; Dynamo Mocker is basic
- llm-d Sim is standalone; Dynamo Mocker integrates with runtime
- llm-d Sim is Go; Dynamo Mocker is Python

## Related Documentation

- [llm-d Testing Guide](https://github.com/llm-d/llm-d/docs/testing.md)
- [vLLM Documentation](https://docs.vllm.ai/)
- [Development Guide](DEVELOPMENT.md)

## Tests

- **Unit Tests**: Package-level tests
- **Integration Tests**: With scheduler
- **Load Tests**: Concurrent request handling
- **Latency Tests**: Distribution validation
