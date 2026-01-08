# llm-d Benchmark Component Summary

## Overview

The llm-d Benchmark component provides an automated benchmarking framework for evaluating inference performance on the llm-d platform. It includes load generators, metrics collection, analysis tools, and Jupyter notebooks for visualization, enabling comprehensive performance testing of LLM serving deployments.

## Location

- **Repository**: `https://github.com/llm-d/llm-d-benchmark` (public)
- **Language**: Python, Jupyter Notebooks
- **Stars**: 41+
- **Description**: Automated benchmarking framework with load generators

## Internal Dependencies

### Python Dependencies

| Category | Dependencies |
|----------|--------------|
| **HTTP Client** | aiohttp, requests |
| **Load Generation** | locust, vegeta |
| **Data Analysis** | pandas, numpy |
| **Visualization** | matplotlib, plotly |
| **Notebooks** | jupyter, jupyterlab |
| **Configuration** | pydantic, yaml |
| **Metrics** | prometheus-client |

### External Services

| Service | Purpose |
|---------|---------|
| llm-d Platform | Target for benchmarks |
| Prometheus | Metrics collection |
| Grafana | Visualization (optional) |

## Module Structure

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `benchmarks/` | Benchmark definitions |
| `load_generators/` | Load generation tools |
| `collectors/` | Metrics collectors |
| `analysis/` | Data analysis scripts |
| `notebooks/` | Jupyter analysis notebooks |
| `configs/` | Benchmark configurations |
| `scripts/` | Automation scripts |
| `results/` | Benchmark output |

### Benchmark Types

| Type | Description |
|------|-------------|
| **Throughput** | Maximum requests/second |
| **Latency** | TTFT, ITL distributions |
| **Scalability** | Performance vs replicas |
| **Cache** | Prefix cache effectiveness |
| **Disagg** | P/D disaggregation impact |

## Public Interface

### Configuration

```yaml
# benchmark-config.yaml
benchmark:
  name: "throughput-test"
  duration: 300  # seconds
  warmup: 30

target:
  endpoint: "http://gateway/v1/chat/completions"
  model: "llama-70b"

load:
  type: "constant"  # constant, ramp, burst
  concurrency: 100
  rate: 50  # requests/second

prompts:
  type: "synthetic"  # synthetic, file, shareGPT
  inputLength:
    mean: 500
    stddev: 100
  outputLength:
    mean: 200
    stddev: 50

metrics:
  collect:
    - ttft
    - itl
    - throughput
    - errors
  prometheus:
    endpoint: "http://prometheus:9090"
```

### CLI Interface

```bash
# Run benchmark
llm-d-bench run --config benchmark-config.yaml

# Generate report
llm-d-bench report --input results/ --output report.html

# Compare runs
llm-d-bench compare --baseline run1/ --candidate run2/
```

### Programmatic API

```python
from llmd_benchmark import Benchmark, LoadGenerator, MetricsCollector

# Create benchmark
benchmark = Benchmark(
    name="my-test",
    duration=300,
    target="http://gateway/v1/chat/completions"
)

# Configure load
benchmark.set_load(
    generator=LoadGenerator.CONSTANT,
    concurrency=100,
    rate=50
)

# Run and collect
results = await benchmark.run()

# Analyze
print(f"P50 TTFT: {results.ttft.p50}ms")
print(f"P99 ITL: {results.itl.p99}ms")
print(f"Throughput: {results.throughput}req/s")
```

## User/Developer Interaction

### 1. Quick Start

```bash
# Clone repository
git clone https://github.com/llm-d/llm-d-benchmark.git
cd llm-d-benchmark

# Install dependencies
pip install -r requirements.txt

# Run basic benchmark
python -m llmd_benchmark.run \
  --endpoint http://gateway/v1/chat/completions \
  --model llama-70b \
  --concurrency 50 \
  --duration 60
```

### 2. Custom Benchmark

```yaml
# my-benchmark.yaml
benchmark:
  name: "cache-effectiveness"
  duration: 600

load:
  type: "shareGPT"
  dataset: "ShareGPT_V3_unfiltered.json"
  prefixRatio: 0.5

metrics:
  collect:
    - cache_hit_ratio
    - ttft
    - throughput
```

```bash
llm-d-bench run --config my-benchmark.yaml
```

### 3. Jupyter Analysis

```bash
# Start Jupyter
jupyter lab

# Open notebooks/analysis.ipynb
```

### 4. Generate Report

```bash
# HTML report
llm-d-bench report \
  --input results/my-test/ \
  --output report.html \
  --format html

# PDF report
llm-d-bench report \
  --input results/my-test/ \
  --output report.pdf \
  --format pdf
```

## Packaging & Containers

### Installation

```bash
# From PyPI
pip install llm-d-benchmark

# From source
pip install -e .

# With extras
pip install llm-d-benchmark[notebooks,locust]
```

### Docker

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install -e .[all]
ENTRYPOINT ["llm-d-bench"]
```

### Kubernetes Job

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: benchmark-run
spec:
  template:
    spec:
      containers:
        - name: benchmark
          image: llm-d-benchmark:latest
          args:
            - run
            - --config=/config/benchmark.yaml
          volumeMounts:
            - name: config
              mountPath: /config
            - name: results
              mountPath: /results
```

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| Configuration | YAML file | Structured config |
| Prompts | File/synthetic | Text/JSON |
| Target endpoint | CLI/config | URL |

### Outputs

| Output | Location | Format |
|--------|----------|--------|
| Raw metrics | `results/<run>/metrics.json` | JSON |
| Summary | `results/<run>/summary.json` | JSON |
| Plots | `results/<run>/plots/` | PNG |
| Report | `results/<run>/report.html` | HTML |

### Result Structure

```
results/
└── my-benchmark-2024-01-15/
    ├── config.yaml           # Run configuration
    ├── metrics.json          # Raw metrics
    ├── summary.json          # Aggregated results
    ├── plots/
    │   ├── ttft_distribution.png
    │   ├── itl_distribution.png
    │   ├── throughput_over_time.png
    │   └── latency_percentiles.png
    └── report.html           # Full report
```

## Observability

### Metrics Collected

| Metric | Description |
|--------|-------------|
| `ttft` | Time to first token |
| `itl` | Inter-token latency |
| `e2e_latency` | End-to-end latency |
| `throughput` | Requests/second |
| `tokens_per_second` | Token throughput |
| `error_rate` | Error percentage |
| `queue_depth` | Request queue depth |
| `cache_hit_ratio` | Prefix cache hits |

### Statistical Analysis

- Mean, median, P50, P90, P95, P99
- Standard deviation
- Distribution histograms
- Time series plots

### Prometheus Integration

```yaml
# Collect from Prometheus during benchmark
metrics:
  prometheus:
    endpoint: "http://prometheus:9090"
    queries:
      - name: gpu_utilization
        query: 'nvidia_gpu_duty_cycle'
      - name: memory_used
        query: 'nvidia_gpu_memory_used_bytes'
```

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| Configuration | YAML | Well-structured |
| Output format | JSON + HTML | Documented |
| Metrics | Comprehensive | Standard names |
| Visualization | Matplotlib/Plotly | Interactive |
| Documentation | README | Basic guides |

## Customization & Extension

### Custom Load Generators

```python
from llmd_benchmark.load import BaseLoadGenerator

class CustomLoadGenerator(BaseLoadGenerator):
    async def generate_request(self):
        # Custom request generation
        return {
            "model": self.model,
            "messages": self.generate_messages(),
            "stream": True
        }
```

### Custom Metrics Collectors

```python
from llmd_benchmark.collectors import BaseCollector

class CustomCollector(BaseCollector):
    async def collect(self, response):
        # Custom metrics extraction
        return {
            "custom_metric": self.extract_custom(response)
        }
```

### Custom Analysis

```python
from llmd_benchmark.analysis import BaseAnalyzer

class CustomAnalyzer(BaseAnalyzer):
    def analyze(self, metrics):
        # Custom analysis logic
        return {
            "custom_insight": self.compute_insight(metrics)
        }
```

## Dynamo Equivalent

**Closest Match**: **Benchmarks**

| Aspect | llm-d Benchmark | Dynamo Benchmarks |
|--------|-----------------|-------------------|
| **Purpose** | Performance evaluation | Performance evaluation |
| **Load Gen** | Locust, custom | AIPerf |
| **Metrics** | TTFT, ITL, throughput | TTFT, ITL, throughput |
| **Visualization** | Jupyter, Plotly | Matplotlib |
| **Profiling** | Basic | SLA-driven profiler |

**Key Differences**:
- llm-d uses Locust/custom generators; Dynamo uses AIPerf
- llm-d has Jupyter-first analysis; Dynamo uses CLI tools
- llm-d is standalone; Dynamo integrates with DGDR for auto-config
- llm-d is Python; Dynamo benchmarks are mixed Python/Rust

## Related Documentation

- [llm-d Performance Guide](https://github.com/llm-d/llm-d/docs/performance.md)
- [Benchmark Examples](notebooks/)
- [Configuration Reference](docs/config.md)

## Tests

- **Unit Tests**: `tests/` directory
- **Integration Tests**: With llm-d platform
- **Validation Tests**: Known workload baselines
- **Regression Tests**: Performance comparisons
