# Benchmarks Component Summary

## Overview

The Benchmarks component provides comprehensive performance evaluation tools for Dynamo deployments. It includes client-side and server-side benchmarking, SLA-driven profiling for automatic configuration optimization, router performance testing, and synthetic data generation for controlled workload simulation.

## Location

- **Primary**: `benchmarks/`
- **Documentation**: `docs/benchmarks/benchmarking.md`
- **Profiler**: `benchmarks/profiler/`
- **Utils**: `benchmarks/utils/`

## Internal Dependencies

### Python Dependencies

```python
# From pyproject.toml
aiconfigurator[webapp]  # AI configurator with WebUI
networkx                # Graph algorithms
pandas                  # Data analysis
pydantic>=2             # Configuration validation
tabulate                # Table formatting
transformers>=4.56.0    # Tokenization
matplotlib              # Plotting
```

### External Dependencies

| Dependency | Purpose |
|------------|---------|
| AIPerf | Benchmark execution |
| NVIDIA GPUs | Inference testing |
| etcd + NATS | Router benchmarking |
| Kubernetes | In-cluster benchmarks |

## Module Structure

### Core Utilities (`benchmarks/utils/`)

| File | Lines | Purpose |
|------|-------|---------|
| `benchmark.py` | 102 | Main CLI entry point |
| `workflow.py` | 96 | Benchmark orchestration |
| `aiperf.py` | 113 | AIPerf wrapper |
| `plot.py` | 464 | Visualization and reporting |

### Profiler (`benchmarks/profiler/`)

| Module | Purpose |
|--------|---------|
| `profile_sla.py` | Main SLA profiling script |
| `profile_endpoint.py` | Endpoint profiling |
| `webui/` | Interactive configuration UI |
| `deploy/` | DGDR configurations |
| `utils/` | Profiler utilities |

### Backend Modifiers (`benchmarks/profiler/utils/config_modifiers/`)

| File | Purpose |
|------|---------|
| `vllm.py` | vLLM configuration |
| `sglang.py` | SGLang configuration |
| `trtllm.py` | TensorRT-LLM configuration |

### Router Benchmarks (`benchmarks/router/`)

| File | Purpose |
|------|---------|
| `prefix_ratio_benchmark.py` | Prefix caching sweeps |
| `real_data_benchmark.py` | Trace-based benchmarking |
| `run_engines.sh` | Worker launcher |

### Data Generation

| Directory | Purpose |
|-----------|---------|
| `prefix_data_generator/` | Trace synthesis and analysis |
| `sin_load_generator/` | Sinusoidal load patterns |
| `burstgpt_loadgen/` | BurstGPT trace conversion |

## Public Interface

### CLI Entry Point

```bash
python3 -m benchmarks.utils.benchmark \
  --benchmark-name <name> \
  --endpoint-url <url> \
  --model <model>
```

### CLI Arguments

| Argument | Description | Default |
|----------|-------------|---------|
| `--benchmark-name` | Unique identifier (alphanumeric, -, _) | Required |
| `--endpoint-url` | Target HTTP endpoint | Required |
| `--model` | Model name (must match deployment) | Required |
| `--isl` | Input sequence length | 2000 |
| `--osl` | Output sequence length | 256 |
| `--concurrencies` | Concurrency levels | [1,2,5,10,50,100,250] |

### Plot Generation

```bash
python3 -m benchmarks.utils.plot --data-dir ./benchmarks/results
```

### Data Generator CLI

```bash
# Analyze traces
datagen analyze --input trace.jsonl

# Synthesize traces
datagen synthesize --output synthetic.jsonl --prefix-ratio 0.5
```

## User/Developer Interaction

### 1. Client-Side Benchmarking

```bash
# Port-forward deployment
kubectl port-forward -n <ns> svc/<service> 8000:8000 &

# Run benchmark
python3 -m benchmarks.utils.benchmark \
  --benchmark-name deployment-a \
  --endpoint-url http://localhost:8000 \
  --model "Qwen/Qwen3-0.6B"

# Generate plots
python3 -m benchmarks.utils.plot --data-dir ./benchmarks/results
```

### 2. Server-Side Benchmarking (In-Cluster)

```bash
# Deploy benchmark job
kubectl apply -f benchmarks/incluster/benchmark_job.yaml -n $NAMESPACE

# Monitor
kubectl logs -f job/dynamo-benchmark -n $NAMESPACE

# Retrieve results
kubectl cp $NAMESPACE/pvc-access-pod:/data/results ./results

# Plot
python3 -m benchmarks.utils.plot --data-dir ./results
```

### 3. SLA-Driven Profiling

```bash
# Via DGDR CRD
kubectl apply -f benchmarks/profiler/deploy/profile_sla_dgdr.yaml

# Direct execution
python -m benchmarks.profiler.profile_sla \
  --backend trtllm \
  --model Qwen/Qwen3-32B \
  --ttft 200 \
  --itl 15
```

### 4. Router Benchmarking

```bash
# Start workers
./benchmarks/router/run_engines.sh --num-workers 8 \
  --model-path deepseek-ai/DeepSeek-R1-Distill-Llama-8B

# Start router
python -m dynamo.frontend --router-mode kv --http-port 8000

# Run prefix ratio sweep
python benchmarks/router/prefix_ratio_benchmark.py \
  --prefix-ratios 0.1 0.5 0.9
```

## Packaging & Containers

### In-Cluster Job Configuration

```yaml
# benchmarks/incluster/benchmark_job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: dynamo-benchmark
spec:
  template:
    spec:
      containers:
      - name: benchmark
        image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1
        command:
          - python3
          - -m
          - benchmarks.utils.benchmark
        volumeMounts:
          - name: results
            mountPath: /data/results
```

### Storage Requirements

| PVC | Size | Purpose |
|-----|------|---------|
| Results | 50Gi | Benchmark outputs |
| Model cache | 100Gi | Model weights (if needed) |

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| Endpoint URL | CLI argument | HTTP URL |
| Model name | CLI argument | String |
| Trace data | File | Mooncake format (JSONL) |
| Configuration | CLI/DGDR | Various |

### Outputs

| Output | Location | Format |
|--------|----------|--------|
| Raw results | `results/<name>/c*/` | JSON/CSV |
| Plots | `results/plots/` | PNG images |
| Summary | `results/plots/SUMMARY.txt` | Text |

### Result Directory Structure

```
benchmarks/results/
├── plots/
│   ├── SUMMARY.txt
│   ├── p50_inter_token_latency_vs_concurrency.png
│   ├── avg_inter_token_latency_vs_concurrency.png
│   ├── request_throughput_vs_concurrency.png
│   ├── efficiency_tok_s_gpu_vs_user.png
│   └── avg_time_to_first_token_vs_concurrency.png
├── <benchmark-name-1>/
│   ├── c1/
│   │   ├── profile_export_aiperf.json
│   │   └── profile_export_aiperf.csv
│   ├── c2/
│   └── ...
└── <benchmark-name-N>/
```

## Metrics Measured

### Primary Metrics

| Metric | Unit | Description |
|--------|------|-------------|
| TTFT | ms | Time to first token |
| ITL | ms | Inter-token latency |
| Request throughput | req/s | Requests per second |
| Token throughput | tok/s | Tokens per second |
| P50/P90/P99 latency | ms | Percentile latencies |

### Efficiency Metrics

| Metric | Description |
|--------|-------------|
| Tokens/sec/GPU | Per-GPU efficiency |
| Tokens/sec/User | User-perceived throughput |
| Cache hit rate | Prefix cache effectiveness |

### SLA Profiling Metrics

| Metric | Description |
|--------|-------------|
| TTFT vs ISL | Prefill performance curve |
| ITL vs KV usage | Decode performance |
| Pareto-optimal configs | Latency-throughput trade-offs |

## Observability

### Generated Plots

| Plot | Description |
|------|-------------|
| `p50_inter_token_latency_vs_concurrency.png` | P50 ITL across concurrency |
| `avg_inter_token_latency_vs_concurrency.png` | Average ITL |
| `request_throughput_vs_concurrency.png` | Throughput scaling |
| `efficiency_tok_s_gpu_vs_user.png` | GPU efficiency scatter |
| `avg_time_to_first_token_vs_concurrency.png` | TTFT scaling |

### Summary Report

```
# SUMMARY.txt
Benchmark Results Summary
=========================
benchmark-a:
  Max throughput: 150 req/s
  P50 TTFT: 45ms
  P50 ITL: 12ms

benchmark-b:
  Max throughput: 200 req/s
  P50 TTFT: 38ms
  P50 ITL: 10ms
```

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| CLI interface | Functional | Document all flags |
| Output format | JSON/CSV | Schema versioning |
| Metric names | Various | Standardize naming |
| Result structure | Implicit | Document schema |
| Profiler configs | Multiple | Unified config format |
| Trace format | Mooncake | Document specification |

## Customization & Extension

### Extension Points

1. **Custom Workloads** - Add data generators
2. **Custom Metrics** - Extend AIPerf wrapper
3. **Custom Plots** - Add to `plot.py`
4. **Backend Configs** - Add config modifiers

### Adding Custom Benchmarks

```python
# benchmarks/utils/custom_benchmark.py
from benchmarks.utils.workflow import BenchmarkWorkflow

class CustomBenchmark(BenchmarkWorkflow):
    def setup(self):
        # Custom setup
        pass

    def run(self):
        # Custom execution
        pass

    def analyze(self):
        # Custom analysis
        pass
```

### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `CONCURRENCIES` | Concurrency levels | `[1,2,5,10,50,100,250]` |
| `DEFAULT_ISL` | Input sequence length | 2000 |
| `DEFAULT_OSL` | Output sequence length | 256 |
| `MAX_BENCHMARKS` | Max comparisons | 12 |

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Multi-cluster | Single cluster only | Run separately |
| Real-time | Post-hoc analysis | Use streaming metrics |
| Windows | Linux containers | Use WSL2 |

## Related Documentation

- [Benchmarking Guide](docs/benchmarks/benchmarking.md)
- [SLA Profiler Guide](benchmarks/profiler/README.md)
- [Router Benchmarks](benchmarks/router/README.md)
- [Data Generator](benchmarks/prefix_data_generator/README.md)

## Tests

- Unit tests: `benchmarks/prefix_data_generator/tests/`
- Integration tests: In-cluster job validation
- CI tests: Automated benchmark smoke tests
