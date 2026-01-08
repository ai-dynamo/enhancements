# Planner Component Summary

## Overview

The Planner monitors system state and automatically scales prefill/decode workers to meet performance targets. It supports SLA-based scaling (predictive, using TTFT/ITL targets) and load-based scaling (reactive, using KV cache and queue metrics).

## Location

- **Source**: `components/src/dynamo/planner/`
- **Package**: `ai-dynamo` (part of main package)
- **Current Version**: 0.8.0

## Internal Dependencies

### Dynamo Python Modules

| Module | Usage |
|--------|-------|
| `dynamo.runtime` | `DistributedRuntime`, `dynamo_worker` decorator |
| `dynamo.runtime.logging` | `configure_dynamo_logging()` - unified logging |
| `dynamo._core` | `VirtualConnectorCoordinator` - Rust binding for etcd coordination |
| `dynamo.prometheus_names` | Metric name constants for Prometheus queries |

### Dynamo Rust Crates (via Python Bindings)

| Crate | Location | Usage |
|-------|----------|-------|
| `dynamo_runtime` | `lib/runtime/` | Core runtime, etcd transport |
| Python bindings | `lib/bindings/python/rust/planner.rs` | `VirtualConnectorCoordinator`, `VirtualConnectorClient`, `PlannerDecision` |

The `VirtualConnectorCoordinator` (Rust) provides:
- etcd-based state coordination for local/virtual scaling
- Atomic decision tracking (`decision_id`, `scaled_decision_id`)
- Worker count management via etcd KV store

### External Python Dependencies

| Package | Usage |
|---------|-------|
| `kubernetes` | K8s API client for `KubernetesConnector` |
| `prometheus_api_client` | Query Prometheus for metrics |
| `prometheus_client` | Expose planner's own metrics |
| `pydantic` | Data validation (`TargetReplica`, config models) |
| `prophet` | Load prediction (SLA planner) |
| `pmdarima` | ARIMA-based load prediction |
| `numpy`, `pandas` | Data processing for metrics/predictions |
| `matplotlib` | Visualization (dry-run plotting) |

### External Services

| Service | Required For | Notes |
|---------|--------------|-------|
| etcd | VirtualConnector | State coordination for local testing |
| Prometheus | SLA Planner | Metric collection and querying |
| K8s API | KubernetesConnector | DGD scaling operations |

## Public Interface

### Exported Classes (`__all__`)

| Class | Type | Description |
|-------|------|-------------|
| `PlannerConnector` | ABC | Abstract base class for planner connectors |
| `KubernetesConnector` | Class | K8s implementation - scales via DGD API |
| `VirtualConnector` | Class | Local/testing implementation via DistributedRuntime |
| `LoadPlannerDefaults` | Config | Load-based planner configuration |
| `SLAPlannerDefaults` | Config | SLA-based planner configuration |
| `TargetReplica` | Pydantic Model | Scaling target specification |
| `SubComponentType` | Enum | PREFILL, DECODE |

### PlannerConnector (ABC) Methods

```python
async def add_component(component_name) -> None
async def remove_component(component_name) -> None
```

### KubernetesConnector Methods

```python
async def add_component(sub_component_type: SubComponentType, blocking: bool = True) -> None
async def remove_component(sub_component_type: SubComponentType, blocking: bool = True) -> None
async def set_component_replicas(target_replicas: list[TargetReplica], blocking: bool = True) -> None
async def validate_deployment(prefill_component_name: str = None, decode_component_name: str = None) -> None
def get_model_name(deployment: dict = None) -> str
async def wait_for_deployment_ready() -> None
```

### Configuration Classes

**BasePlannerDefaults** (common):
- `namespace`, `environment`, `backend`
- `adjustment_interval` (default: 180s)
- `max_gpu_budget`, `min_endpoint`
- `decode_engine_num_gpu`, `prefill_engine_num_gpu`
- `metric_reporting_prometheus_port`

**LoadPlannerDefaults** (load-based scaling):
- `metric_pulling_interval` (default: 10s)
- `decode_kv_scale_up_threshold` (default: 0.9)
- `decode_kv_scale_down_threshold` (default: 0.5)
- `prefill_queue_scale_up_threshold` (default: 5.0)
- `prefill_queue_scale_down_threshold` (default: 0.2)

**SLAPlannerDefaults** (SLA-based scaling):
- `metric_pulling_prometheus_endpoint`
- `profile_results_dir`
- `isl`, `osl` (input/output sequence length targets)
- `ttft`, `itl` (latency targets in ms)
- `load_predictor` (constant, arima, prophet)
- `load_prediction_window_size`

## User/Developer Interaction

### 1. Deployment (Kubernetes) - Primary Use Case

Users don't call planner APIs directly. The planner runs as an autonomous component in a DynamoGraphDeployment:

**DGD Spec Example:**
```yaml
Planner:
  dynamoNamespace: my-deployment
  componentType: planner
  replicas: 1
  extraPodSpec:
    mainContainer:
      image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.1
      workingDir: /workspace/components/src/dynamo/planner
      command: [python3, -m, planner_sla]
      args:
        - --environment=kubernetes
        - --backend=vllm
        - --adjustment-interval=60
        - --profile-results-dir=/workspace/profiling_results
```

**Configuration via:**
- Environment variables (`DYN_NAMESPACE`, `DYN_PARENT_DGD_K8S_NAME`, `PROMETHEUS_ENDPOINT`)
- CLI arguments for planner mode selection
- DGD YAML spec defining services with `subComponentType: prefill/decode`

### 2. SLA-based Workflow (Recommended)

1. **Pre-deployment**: Run profiling (2-4 hours on real silicon or minutes with AI Configurator simulator)
2. **Deploy**: Planner pulls metrics from Prometheus, predicts load, scales workers to meet TTFT/ITL targets
3. **Runtime**: Continuous adjustment based on observed metrics and SLA targets

### 3. Programmatic Usage (Advanced/Testing)

```python
from dynamo.planner import KubernetesConnector, TargetReplica, SubComponentType

connector = KubernetesConnector(dynamo_namespace="my-ns")
await connector.set_component_replicas([
    TargetReplica(sub_component_type=SubComponentType.PREFILL, desired_replicas=2),
    TargetReplica(sub_component_type=SubComponentType.DECODE, desired_replicas=4),
])
```

### 4. VirtualConnector (Local Testing)

For local testing without K8s - simulates scaling via `DistributedRuntime`.

## Packaging & Containers

### Container

- **NOT a separate container** - included in main runtime images
- **Registry**: `nvcr.io/nvidia/ai-dynamo/`
- **Images**: `vllm-runtime`, `tensorrtllm-runtime`, `sglang-runtime`
- **Path in container**: `/workspace/components/src/dynamo/planner/`

### K8s Operator Auto-Injection

The Dynamo operator automatically configures planner components:
- `serviceAccountName: planner-serviceaccount` (for K8s API access)
- `PLANNER_PROMETHEUS_PORT: 9085` environment variable
- Metrics port 9085 exposed (named `metrics`)

### Required K8s Permissions

Planner requires K8s API access to:
- GET/PATCH `DynamoGraphDeployment` resources
- Read deployment status and replica counts

## Observability

### Prometheus Metrics (prefix: `planner:`)

| Metric | Type | Description |
|--------|------|-------------|
| `planner:num_p_workers` | Gauge | Number of prefill workers |
| `planner:num_d_workers` | Gauge | Number of decode workers |
| `planner:observed_ttft` | Gauge | Observed time to first token (ms) |
| `planner:observed_itl` | Gauge | Observed inter-token latency (ms) |
| `planner:observed_request_rate` | Gauge | Observed request rate (req/s) |
| `planner:observed_request_duration` | Gauge | Observed request duration (s) |
| `planner:observed_isl` | Gauge | Observed input sequence length |
| `planner:observed_osl` | Gauge | Observed output sequence length |
| `planner:p_correction_factor` | Gauge | Prefill correction factor |
| `planner:d_correction_factor` | Gauge | Decode correction factor |

### Grafana Dashboard

Pre-built dashboard available: `deploy/observability/k8s/grafana-planner-dashboard-configmap.yaml`

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| Python exports | `__all__` defined | Document public vs internal APIs |
| CLI entry point | `python -m planner_sla` | Standardize, document args |
| Config schema | CLI args + env vars | Versioned config schema, validation |
| Prometheus metrics | `planner:*` prefix | Document all metrics, types, labels |
| K8s RBAC | `planner-serviceaccount` | Document required permissions |
| DGD spec | `componentType: planner` | Document required/optional fields |
| Error handling | Custom exceptions in `utils/exceptions.py` | Standardize error codes/messages |
| Logging | Uses `dynamo.runtime.logging` | Consistent log levels/formats |
| Versioning | Part of `ai-dynamo` package | Independent versioning? |

## Related Documentation

- [Planner Introduction](docs/planner/planner_intro.rst)
- [SLA Planner Quick Start](docs/planner/sla_planner_quickstart.md)
- [SLA-Driven Profiling](docs/benchmarks/sla_driven_profiling.md)

## Tests

- Unit tests: `tests/planner/unit/`
- E2E tests: `tests/planner/test_scaling_e2e.py`
- Replica calculation tests: `tests/planner/test_replica_calculation.py`
