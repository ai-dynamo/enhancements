# Dynamo Deployment Flow with DGDRs

**Status**: Draft

**Authors**: Hannah Zhang

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: Kyle Kranen

**Required Reviewers**: Kyle Kranen, Maksim Khadkevich, Ben Hamm, Anish Maddipoti

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

---

# Summary

This proposal reframes the **DynamoGraphDeploymentRequest (DGDR)**, a Kubernetes Custom Resource Definition (CRD), as the unified, declarative entry point for deploying Dynamo models to Kubernetes clusters. DGDR automates the journey from high-level user intent (model, backend, SLAs) to low-level Kubernetes resources (DynamoGraphDeployments), transparently handling profiling and configuration selection.

DGDR is the recommended on-ramp to DynamoGraphDeployments (DGDs) for all users.

---

# Motivation

Currently, Dynamo deployment flows lack a clear golden path. Users face multiple competing entry points: Dynamo Recipes (model-specific, pre-configured), AI Configurator (simulation-based, requires manual DGD deployment), and SLA-Driven Profiling (online profiling, complex setup). This fragmentation creates confusion and prevents a consistent, streamlined experience for new users.

DGDR addresses this by providing one declarative interface that works for both simple and advanced users. Simple users can set `autoApply: true` to instantly deploy. Advanced users can set `autoApply: false`, inspect profiling results, export the generated DGD, and iterate directly on that object. DGDR is the entry path, DGDs are the long-term interface.

## Goals

* Unified deployment entrypoint: single high-level CRD for all Dynamo Kubernetes deployments
* Flexible profiling paths: support both rapid and thorough profiling
* SLA-aware optimization: drive configuration selection based on workload and latency/throughput targets
* Feature optionality: enable/disable Dynamo features (Planner, KV Router, KVBM) without rigid coupling
* Progressive complexity: simple for users with minimal requirements; powerful for advanced customization
* Entry-to-iteration: DGDR bootstraps a DGD; users then iterate directly on DGDs themselves

## Non-Goals

* DGDR does **not** replace DynamoGraphDeployment (DGD). DGD remains the low-level, fully expressive interface for long-term operation and deep customization.
* Detailed scheduling/resource-request logic is delegated to Planner where applicable.
* This proposal does **not** require Planner to be implemented or available in all deployments. Planner is optional.

## Supported Deployment Patterns

The DGDR must support multiple deployment patterns:

| Pattern | Description | Deployment Mode | Key Features |
| :---- | :---- | :---- | :---- |
| **Rapid (AIC)** | Fast simulation-based profiling | `searchStrategy: rapid` | Quick setup (seconds), reasonable defaults, no GPU costs |
| **Thorough (Online)** | On-cluster profiling with real GPUs | `searchStrategy: thorough` | SLA-optimized, actual metrics, GPU costs |
| **Auto Backend** | Automatic backend selection | `backend: auto` | Compare all backends, select optimal one |
| **Exploratory** | Inspect before deploying | `autoApply: false` | GUI for comparison, manual deployment |
| **Auto-Deploy** | Instant deployment | `autoApply: true` | Fast path to production |

## Requirements

### REQ 1: Unified Declarative Interface

DGDR **MUST** provide a single, declarative, high-level interface for deploying Dynamo models. Users **MUST** be able to specify a model, backend, and workload/SLAs, and DGDR **MUST** produce a deployable DynamoGraphDeployment without requiring manual configuration of low-level parameters.

### REQ 2: SLA-Driven Profiling

DGDR **MUST** support SLA-driven profiling. When users provide SLAs (TTFT, ITL) and workload parameters (ISL, OSL), DGDR **MUST** run profiling if possible to find configurations that satisfy or approach those SLAs, and **SHOULD** select the best configuration according to throughput and latency metrics.

### REQ 3: Optional SLAs and Workload Parameters

DGDR **MUST** allow users to omit SLAs and workload parameters. When omitted, DGDR **SHOULD** use reasonable defaults (e.g. ISL=512, OSL=128, TTFT=100ms, ITL=15ms) and still produce a valid, deployable DGD.

### REQ 4: Multiple Search Strategies

DGDR **MUST** support at least two search strategies: (a) rapid (AIC-only, ~30 seconds), and (b) thorough (real on-cluster profiling deployments, minutes to hours). Users **MUST** be able to select via `searchStrategy` field.

### REQ 5: Optional Auto-Deploy

DGDR **MUST** support both exploratory and auto-deploy modes. When `autoApply: true`, DGDR **MUST** automatically create and apply the generated DGD, also storing it in the DGDR status. When `autoApply: false`, DGDR **MUST** store profiling results and recommended DGD in status, allowing users to inspect and manually trigger deployment. Optionally, users can portforward the DGDR locally and view a GUI with profiling data with the option to export and deploy DGDRs directly from the UI.

### REQ 6: Optional Planner Integration

DGDR **MUST** allow users to optionally include Planner in the generated DGD via the `features.planner` field. When `features.planner: false` (default), the generated DGD **MUST NOT** include Planner. When `features.planner: true`, the generated DGD **MAY** include Planner for autoscaling and SLA monitoring.

### REQ 7: Entry Path, Not Permanent Home

DGDR **MUST** be documented and designed as the recommended entry path to generating a first DGD. Once a DGD is generated, users **MUST** be able to export, inspect, and iterate directly on that DGD without depending on DGDR for long-term operation.

### REQ 8: Support for Model Cache

DGDR **MUST** support mounting model caches (PVCs) for large or private models. Users **MUST** be able to specify `modelCache.pvcName` and `modelCache.pvcPath`, and DGDR **MUST** use the cached model for profiling and in the generated DGD.

### REQ 9: Pareto Optimization

When profiling completes, DGDR **MUST** contain not just one recommended configuration, but also a Pareto curve (set of non-dominated configurations). Users **MUST** be able to see trade-offs between throughput, latency, and other metrics.

### REQ 10: Clear Status and Monitoring

DGDR **MUST** expose a clear status field that tracks the profiling phase (Pending, Profiling, Ready, Deploying, Deployed, Failed). Users **MUST** be able to observe progress and understand what is happening at each stage.

---

# Proposal

## High-Level Design

DGDR is a Kubernetes CRD that acts as a declarative interface to profiling and DGD generation. The user creates a DGDR resource; the DGDR controller runs profiling, selects a best configuration, generates a DGD, and optionally applies it.

### Flow Diagram

```
User creates DGDR
    ↓
Controller validates spec (model, backend, hardware)
    ↓
Hardware/GPU discovery
    ↓
[If searchStrategy: rapid]
    → AIC profiling (~30 seconds)
    → Pareto curve + recommended config
    ↓
[If searchStrategy: thorough]
    → Online profiling (minutes to hours)
    → Real on-cluster metrics
    → Pareto curve + recommended config
    ↓
[If autoApply: true]
    → Generate DGD
    → Apply DGD to cluster
    → Monitor deployment
    ↓
[If autoApply: false]
    → Store results in status/ConfigMap
    → User views GUI (portforward)
    → User selects config and deploys
    ↓
DGD running in cluster
```

### API Overview

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: my-deployment
  namespace: default
spec:
  # Required: Model identifier
  model: "Qwen/Qwen3-32B"           # HF ID or served name if modelCache used
  
  # Optional: Serving backend
  backend: auto                     # enum: [auto, sglang, trtllm, vllm] (default: auto)
  
  # Optional: Container image for profiling (will be `nvcr.io/nvidia/ai-dynamo/dynamo-frontend:<operator-version>` if omitted)
  image: "nvcr.io/nvidia/ai-dynamo/dynamo-frontend:1.0.0"
  
  # Optional: Workload parameters
  workload:
    isl: 512                        # Input sequence length (default: 512)
    osl: 128                        # Output sequence length (default: 128)
  
  # Optional: SLA targets
  sla:
    optimizationType: throughput    # What to optimize: hybrid, throughput, or latency (default: hybrid)
    ttft: 100.0                     # Time-To-First-Token (ms, default: 100)
    itl: 15.0                       # Inter-Token Latency (ms, default: 15)
  
  # Optional: Model cache (for large models on PVC)
  modelCache:
    pvcName: "model-cache"
    pvcPath: "deepseek-r1"
  
  # Optional: Kubernetes resource overrides
  overrides:
    profilingJob: {}                # Job template overrides (e.g., securityContext)
    dgd: {}                         # DGD spec overrides
  
  # Optional: Search strategy
  searchStrategy: rapid              # enum: [rapid, thorough] (default: rapid)
  
  # Optional: extra features
  features:
      planner: false           # Enable autoscaling via Planner (default: false)
      kvRouter: false          # Enable KV router (default: false)
      mocker: false            # Enable mocker deployment for testing (default: false)
  
  # Auto-deploy resulting DGD after profiling
  autoApply: true

status: # theoretical
  # Current phase in the lifecycle
  phase: Ready                       # enum: [Pending, Profiling, Ready, Deploying, Deployed, Failed]
  
  # Reference to generated DGD
  dgdName: "my-deployment-dgd-abc123"
  
  # Reference to profiling job
  profilingJobName: "my-deployment-prof-abc123"
  
  # Detailed status conditions
  conditions:
    - type: ProfilingComplete
      status: "True"
      message: "AIC profiling completed successfully"
  
  # Profiling results (example)
  profilingResults:
    pareto:
      - backend: trtllm
        parallelization: 8
        throughput: 250.5            # tokens/second
        ttft: 125.3                  # milliseconds
        itl: 18.2
      - backend: sglang
        parallelization: 4
        throughput: 220.0
        ttft: 140.0
        itl: 19.5
    selectedConfig: {...}            # Recommended configuration
    startTime: "2026-01-23T12:00:00Z"
    completionTime: "2026-01-23T12:00:35Z"
  
  # Deployment info
  deploymentInfo:
    replicas: 3
    availableReplicas: 3
```

## Field Groups & Descriptions

### Model and Backend Specification

| Field | Description | Required | Default |
| :---- | :---- | :---- | :---- |
| `model` | Target model. If `modelCache` is not used, this must be a HuggingFace ID (e.g., `Qwen/Qwen3-32B`). If `modelCache` is used, this becomes the served model name. | Yes | N/A |
| `backend` | Serving framework: `auto`, `vllm`, `sglang`, `trtllm`. `auto` means profiling will run on all backends and the user will receive the best backend out of all of them. | No | `auto` |
| `image` | Docker image containing Profiler code (DPP + AIC) | No | `nvcr.io/nvidia/ai-dynamo/dynamo-frontend:<tag>` where `<tag>` is the same tag of the currently deployed Dynamo Operator |

**Justification**: These are the minimum inputs for any deployment.

### Workload/SLAs

| Field | Description | Required |
| :---- | :---- | :---- |
| `workload` | User-set ISL and OSL. | No – reasonable defaults are used initially and can be explored via GUI later. |
| `sla` | Target SLAs | No – reasonable defaults are used initially and can be explored via GUI later. |
| `sla.optimizationType` | What to optimize when deciding a config: `hybrid`, `throughput`, or `latency` | No – hybrid is the default |

**Purpose**: Defines how the system finds an optimal configuration.

### Model Cache

| Field | Description | Required |
| :---- | :---- | :---- |
| `modelCache.pvcName` | Name of PVC containing model weights | No |
| `modelCache.pvcPath` | Subpath within PVC where model is stored | No |

**Purpose**: Allows users to attach and use PVCs for local storage of models.

### Overrides

| Field | Description | Required |
| :---- | :---- | :---- |
| `overrides.profilingJob` | Override parts of the profiling job created by the DGDR | No |
| `overrides.dgd` | Override parts of the DGDs created by the DGDR | No |

**Purpose**: Allows users to customize profiling jobs and generated DGDs with cluster-specific requirements (e.g., security contexts, secrets).

### Deployment Features

| Field | Description | Required | Default |
| :---- | :---- | :---- | :---- |
| `features.planner` | Whether to enable Planner in the final deployment | No | false |
| `features.kvRouter` | Whether to enable KV Router in the final deployment | No | false |
| `features.mocker` | Whether to have the final deployment be a mocker deployment | No | false |

**Purpose**: Allows users to specify whether certain features should be included in their final deployment. Not guaranteed to strictly have better performance.

### Search Strategy

| Field | Description | Required | Default |
| :---- | :---- | :---- | :---- |
| `searchStrategy` | Whether the user wants a rapid deployment (AIC under the hood) or is okay with a slower but more thorough search (online deployments/validations) | No | `rapid` |

**Purpose**: Allows users to specify whether they want a fast search or a more thorough but more time-consuming search.

### Auto-deployment of DGD

| Field | Description | Required | Default |
| :---- | :---- | :---- | :---- |
| `autoApply` | If `true`, DGDR automatically creates and applies the DGD after profiling completes. If `false`, DGDR deploys a GUI that users can portforward to select a DGD. | No | `true` |

**Justification**: Controls whether DGDR is exploratory or commit-to-deploy. If users want to customize the deployment after profiling, they should set `autoApply: false`, open the GUI by portforwarding, select a configuration, export it, manually edit the DGD, and deploy with `kubectl apply -f`.

## Search Strategies

### Rapid Search (Default)

| Aspect | Detail |
| ------ | ------ |
| **Speed** | ~30 seconds (AIC only) |
| **Accuracy** | Simulation-based; may differ from on-cluster reality |
| **Cost** | $0, no GPU-hours |
| **Output** | Pareto curve, recommended DGD |
| **Best for** | Quick prototyping, users with baseline SLAs, development environments |

**Implementation**: Calls AIC with user-provided workload and SLA constraints, along with auto-detected hardware information. AIC explores the configuration space (parallelization, tensor-parallel degree, KV cache quantization, aggregated vs disaggregated, etc.) and returns ranked candidates.

**When to use**:
- You want to get started quickly
- You're in a development/testing phase
- Your SLAs are not mission-critical
- You want a reasonable baseline configuration
- You're okay with simulation-based estimates

**Hardware Detection**: The controller automatically detects GPU hardware using DCGM or similar tools, then maps the detected hardware to AIC's "system" parameter (e.g., H100 SXM → `h100_sxm`). Users don't need to specify any hardware details.

### Thorough Search

| Aspect | Detail |
| ------ | ------ |
| **Speed** | Minutes to hours (depends on cluster size and config space) |
| **Accuracy** | Actual on-cluster metrics from profiling runs |
| **Cost** | GPU hours for profiling |
| **Output** | Pareto curve, recommended DGD, profiling logs |
| **Best for** | Production deployments, tight SLA requirements, mission-critical services |

**Implementation**:
1. Sweeps over prefill engines with user-provided ISL (fast)
2. Sweeps over decode engines with user-provided ISL/OSL under different KV targets (slow)
3. Sweeps over selected prefill+decode engine with different loads (slowest)

Online profiling internally caps the number of sweeps and uses heuristics (early stopping, Bayesian-style pruning) to optimize the search. The user is not exposed to the internals of sweeping logic or sweep counts; those remain implementation details.

**When to use**:
- You need production-ready configurations
- Your SLAs are tight and mission-critical
- You want actual on-cluster performance data
- You're willing to spend GPU hours for accuracy
- You need to validate performance before deploying

**Important notes**:
- Thorough search consumes GPU resources during profiling
- The profiling jobs run on the same cluster where you'll deploy
- Results are cluster-specific and account for actual hardware, network, storage characteristics
- For Phase 0 (GTC 2026), thorough search only supports disaggregated deployments

## Deployment After Profiling

### Option A: Exploratory (`autoApply: false`)

User runs profiling and manually deploys:

```
DGDR created (autoApply: false)
    ↓
Profiling runs
    ↓
User inspects Pareto curves (GUI)
    ↓
User selects a preferred DGD, edits as desired, deploys with a button
    ↓
(alternately) User exports recommended DGD, edits as desired
    ↓
kubectl apply -f edited-dgd.yaml
    ↓
DGD deployed
```

### Option B: Commit to SLAs (`autoApply: true`)

User commits to deployment recommended by Profiler:

```
DGDR created (autoApply: true, SLAs specified)
    ↓
Profiling runs
    ↓
Best configuration selected
    ↓
DGD generated and applied automatically
    ↓
User obtains deployment; can still export and edit DGD for future use
```

## Controller Logic (High-Level)

The DGDR controller follows this behavior:

1. **Validate DGDR**: Check that model exists, backend is supported, run hardware/GPU discovery
2. **Run Profiler**: Execute profiling on the hardware with the model information. Profiler returns a recommended DGD and optionally a Pareto curve of configurations.
3. **Auto-Apply or Wait**:
   - If `autoApply == true`: Generate and apply a DGD from the recommended config
   - If `autoApply == false`: Allow users to view UI for DGD comparison and deployment; user chooses one and that DGD is deployed

The controller is designed to be stateless and idempotent. Each DGDR resource is independent, and the controller doesn't maintain state across requests.

## Controller State Machine

```
[Pending] → [Profiling] → [Ready] → [Deploying] → [Deployed]
                          ↘ (error) → [Failed]
```

### Phase: Pending
- Validate DGDR spec (model exists, backend supported, image available)
- Hardware detection -- auto-discover GPU hardware
- Check prerequisites (cluster has Dynamo operator, sufficient GPU capacity)
- Transition to Profiling

### Phase: Profiling
- If `searchStrategy: rapid`: Call AIC, store recommended config in status
- If `searchStrategy: thorough`: Use DynamoPlannerProfiler logic to run sweeps utilizing real, on-cluster deployments. Store Pareto curves, diagrams, and recommended config to ConfigMaps and the recommended config to status.
- Transition to Ready

### Phase: Ready
- Generate DGD from recommended configuration
- If `autoApply: true` → transition to Deploying
- If `autoApply: false` → remain in Ready; user can manually trigger deployment via CLI or GUI

### Phase: Deploying
- Apply DGD to cluster using `kubectl apply`
- Transition to Deployed once DGD is observed by DGD controller

### Phase: Deployed
- Monitor DGD status
- Remain in this phase for the lifetime of the deployment

### Phase: Failed
- Set error conditions in status with descriptive messages
- Remain in Failed; user must fix the spec or delete/recreate the DGDR

## Key Design Decisions

### 1. Auto Backend Selection

When `backend: auto` is specified, DGDR runs profiling across all supported backends (vllm, sglang, trtllm) and compares their configurations. The system evaluates each backend against the provided SLAs and workload parameters, then selects the optimal configuration based on:
- SLA satisfaction (TTFT, ITL targets)
- Throughput optimization
- Resource efficiency

This eliminates the need for users to understand backend-specific characteristics and makes DGDR truly backend-agnostic.

### 2. Planner is Optional, Not the Ultimate Goal

Some DGDR-generated DGDs will include Planner (for autoscaling, SLA tracking), but others will deliberately omit it:

- Early aggregated deployments (before aggregated autoscaling is implemented)
- Users who prefer fixed-capacity, cost-predictable setups
- Early deployments where Planner is not yet available

This is controlled by the `features.planner` field (default: `false`).

### 3. DGDR is an Entry Path, Not a Permanent Interface

DGDR is the way to generate your first DGD. Once you have a DGD, users of all experience levels can iterate directly on it:

- Simple users rely on `autoApply: true` to instantly get a running deployment
- Advanced users run with `autoApply: false`, export the generated DGD, and iterate directly on DGDs for long-term customization

### 4. SLAs and Workload Parameters are Optional

Many users don't know exact SLAs upfront. Reasonable defaults (ISL=512, OSL=128, TTFT=100ms, ITL=15ms) allow quick deployment. GUI/CLI tools let users explore SLA trade-offs later if needed.

### 5. Search Heuristics are Internal

For `searchStrategy: thorough`, internal heuristics (early stopping, Bayesian-style pruning, sweep counts) remain internal. Users only see the `rapid`/`thorough` boolean toggle.

---

# Implementation Details

## Validation and Error Handling

The DGDR controller performs extensive validation before starting profiling:

### Model Validation
- **HuggingFace models**: Verify the model exists and is accessible
- **Private models**: Require `modelCache` to be specified
- **Model compatibility**: Check if AIC supports the model/hardware combination

### Backend Validation
- For specific backends (`vllm`, `sglang`, `trtllm`): Verify the backend supports the model
- For `auto` backend: Profile all available backends

### Configuration Validation
- **Unsupported combinations**: if the user requests `searchStrategy: rapid` for models not supported by AIC, provide clear warnings and return a working but naive configuration
- **SLA feasibility**: When SLAs cannot be met, provide clear warnings and suggest the closest achievable configuration
- **Resource availability**: Check that the cluster has sufficient GPU capacity

### Warning Conditions (Non-Blocking)

Some conditions produce warnings but **do not fail** the DGDR. The DGDR proceeds to `Ready` or `Deployed` phase with a working configuration, but includes warnings in the status to inform users that the result may not be SLA-optimized:

| Warning Type | Status Condition | What Happens | User Action |
|--------------|------------------|--------------|-------------|
| Unsupported model in rapid mode | `type: ConfigurationWarning`<br>`status: "True"`<br>`message: "Model 'foo/bar' not supported by AIC. Generated DGD uses reasonable defaults but may not be SLA-optimized. Use searchStrategy: thorough for SLA optimization"` | DGDR generates a DGD with conservative defaults based on model size and hardware. Deployment succeeds but may not meet specified SLAs. | Review deployed performance. If SLAs aren't met, switch to `searchStrategy: thorough` |
| SLA infeasible | `type: SLAWarning`<br>`status: "True"`<br>`message: "Cannot meet TTFT=50ms with this model/hardware. Closest achievable: TTFT=120ms. Deploying with best available configuration"` | DGDR generates a DGD with the closest achievable configuration. Deployment succeeds with degraded SLA targets. | Accept degraded SLAs, use larger/faster GPUs, or adjust SLA targets |

**Important**: These warnings are stored in `status.conditions[]` and the DGDR phase is `Ready` or `Deployed`, not `Failed`. Users should monitor the status conditions to understand any limitations of their deployment.

### Error Conditions (Blocking)

When critical errors occur, the DGDR transitions to `Failed` phase with descriptive error messages:

| Error Type | Message Example | Resolution |
|------------|----------------|-----------|
| Model not found | "Model 'foo/bar' not found on HuggingFace and no modelCache specified" | Specify correct model or add modelCache |
| Backend incompatibility | "Model 'foo/bar' not supported by backend 'vllm'" | Use different backend or `auto` |
| Insufficient resources | "Cluster does not have enough GPUs for profiling" | Free up resources or use rapid search |

## RBAC & Permissions

DGDR controller requires:

- **Read**: Secrets (for model credentials)
- **Read/Write**: ConfigMaps, DynamoGraphDeployments (to create/update DGDs)
- **Read/Write/Delete**: Jobs (to manage profiling jobs)
- **Read**: Nodes (to discover GPU capacity)

## Monitoring and Observability

Users can monitor DGDR progress through several mechanisms:

### Status Field

The DGDR status field provides real-time information:

```yaml
status:
  phase: Ready  # Current phase: Pending, Profiling, Ready, Deploying, Deployed, Failed
  conditions:
    - type: ProfilingComplete
      status: "True"
      message: "Profiling completed successfully"
    - type: ConfigurationWarning  # Warning (non-blocking)
      status: "True"
      message: "Model not supported by AIC. Generated DGD uses reasonable defaults but may not be SLA-optimized"
  profilingJobName: "my-deployment-prof-abc123"
  profilingResults:
    pareto: [...]  # Available after profiling completes
    selectedConfig: {...}  # Recommended configuration
  dgdName: "my-deployment-dgd-abc123"  # Generated DGD name
```

**Note**: Check `status.conditions[]` for warnings. A DGDR can be in `Ready` or `Deployed` phase with warnings present, indicating the deployment will work but may not be SLA-optimized.

### Watching Status

```bash
# Watch DGDR status
kubectl get dgdr my-deployment -w

# Get detailed status
kubectl describe dgdr my-deployment

# View profiling logs
kubectl logs <profilingJobName>
```

### GUI for Exploration

When `autoApply: false`, users can portforward to view results:

```bash
# Portforward the DGDR service
kubectl port-forward svc/my-deployment-dgdr 8080:80

# Open browser to localhost:8080
# View Pareto curves, compare configurations, deploy selected config
```

The GUI provides:
- **Pareto curves**: Visualize throughput vs latency trade-offs
- **Configuration comparison**: Compare different configurations side-by-side
- **Export DGD**: Download the generated DGD YAML
- **Deploy button**: One-click deployment of selected configuration

## Integration Points

| Component | Role | Current State | DGDR Integration |
| --------- | ---- | ------------- | ---------------- |
| **AI Configurator (AIC)** | Simulation-based configuration search | Provides Pareto curves and candidate configs | DGDR calls AIC for `searchStrategy: rapid`. AIC explores configuration space (parallelization, TP degree, KV quantization, agg/disagg) and returns ranked candidates. |
| **Dynamo Planner Profiler (DPP)** | On-cluster profiling and validation | Disagg-only, provides profiling sweeps | DGDR calls DPP when `searchStrategy: thorough`. DPP runs real deployments and measures actual performance. |
| **DynamoGraphDeployment (DGD)** | Low-level deployment specification | Managed by DGD controller | DGDR generates and optionally applies DGDs. DGD controller manages the lifecycle of deployed models. |
| **Planner** | Runtime SLA tracking and autoscaling | Available for disaggregated deployments | Included in generated DGD only if `features.planner: true`. Monitors SLAs and adjusts replicas dynamically. |
| **Recipes** | Pre-built DGD templates | Deployed directly by users | Complementary path. Users can skip DGDR if using Recipes for benchmarking or known configurations. |

### Integration Flow

```
User creates DGDR
    ↓
DGDR Controller validates and detects hardware
    ↓
[If searchStrategy: rapid]
    → DGDR → AIC → Returns config → DGDR generates DGD
    ↓
[If searchStrategy: thorough]
    → DGDR → DPP → Runs sweeps → Returns config → DGDR generates DGD
    ↓
[If features.planner: true]
    → Generated DGD includes Planner components
    ↓
DGDR applies DGD (if autoApply: true)
    ↓
DGD Controller manages deployment
    ↓
[If Planner enabled]
    → Planner monitors SLAs and autoscales
```

## Example Workflows

### Example 1: Quick Start (No SLAs, No Backend Preference)

**Scenario**: User does not know SLAs or what backend they want to use but wants a reasonable starting deployment.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: no-sla-deployment
spec:
  model: "Qwen/Qwen3-32B"
```

**What happens**:
- User provides only the model
- Internal logic picks conservative, reasonable configuration and parallelization mapping (between agg/disagg and among backends)
- With `autoApply: true` (default), DGDR immediately creates and applies a DGD
- User has something running and can measure performance to refine SLAs later

**Result**: Runs AIC on all available backends with conservative defaults, compares results, and automatically selects and applies the best configuration. Generates and applies DGD in ~30 seconds.

### Example 2: Fast Deployment with SLAs

**Scenario**: User wants quick deployment, knowing their SLAs.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: aic-deployment
spec:
  model: "Qwen/Qwen3-32B"

  workload:
    isl: 3000
    osl: 150
  sla:
    ttft: 200
    itl: 20
```

**What happens**:
- Controller runs hardware/GPU discovery to discover hardware information
- Profiler calls AIC on every backend with SLAs and hardware information
- Profiler returns a Pareto curve of configurations and a recommended DGD
- DGDR controller stores the Pareto curve in status
- DGDR controller writes and applies the recommended DGD (since `autoApply` defaults to `true`)

**Result**: AIC profiles all backends with the specified SLAs and workload constraints, compares results, and auto-deploys the best configuration in ~30 seconds.

### Example 3: Thorough Profiling

**Scenario**: User wants online profiling sweeps for more accurate results.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: online-autodeploy
spec:
  model: "Qwen/Qwen3-32B"
  searchStrategy: thorough
```

**What happens**:
- DGDR runs thorough online profiling with real on-cluster deployments
- From all completed sweeps, DGDR picks the best configuration (not necessarily the first SLA-satisfying one)
- The best configuration is written into a DGD and applied automatically
- Users can still export and inspect the DGD afterward if they want to adjust it

**Result**: Runs real on-cluster profiling sweeps (minutes to hours depending on cluster size). Generates Pareto curve with actual performance metrics and auto-deploys the optimal configuration.

### Example 4: Exploratory Mode (Inspect Before Deploying)

**Scenario**: User wants profiling but prefers to inspect the results and make their own changes before deploying.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: online-exploratory
spec:
  model: "Qwen/Qwen3-32B"
  autoApply: false
```

**What happens**:
- DGDR runs profiling (rapid by default)
- User portforwards DGDR service to access the web UI
- User looks at graphs in the web UI to decide which configuration to deploy
- User selects a configuration and deploys it with a button in the UI
- Advanced users can further export a DGD from the UI, modify it (add custom engine flags, advanced routing, extra Kubernetes annotations), and treat it as the new source of truth
- Future iterations happen directly on DGDs (with or without DGDR), using the profiling results as a reference rather than rerunning DGDR every time

**Result**: Profiling results stored in status/ConfigMaps. User can portforward and view GUI with Pareto curves, compare configurations, and manually select one to deploy.

### Example 5: MoE Model with Model Cache

**Scenario**: Large MoE model with model cache PVC and disaggregated deployment.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: sla-moe
spec:
  model: "deepseek-ai/DeepSeek-R1"
  backend: sglang

  modelCache:
    pvcName: "model-cache"
    pvcPath: "deepseek-r1"  # will override model name within DGDs as local model path
```

**What happens**:
- Controller mounts model cache PVC in profiling Job
- Profiler uses the cached model for profiling
- Profiler generates and deploys a MoE-optimized DGD once profiling finishes
- Generated DGD also uses the model cache PVC

**Result**: Model loaded from PVC instead of downloading from HuggingFace. Profiler returns optimized configuration for the large MoE model.

### Example 6: Private Model

**Scenario**: User has a private model (not on HuggingFace) that they want to use.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: private-model
spec:
  model: "myPrivateModel"
  modelCache:
    pvcName: "model-cache"
    pvcPath: "privateModel"  # will override model name within DGDs as local model path
```

**What happens**:
- The model will be fetched from the model cache to be used for deployment
- Profiler will return a DGD with reasonable defaults following the model size and other model information
- Since the model is not publicly available, AIC may not have pre-computed configurations for it
- If AIC doesn't support the model, thorough search with online profiling is required

**Result**: Model loaded from PVC. If model is supported by AIC, profiling happens normally. If not supported, DGDR falls back to basic configuration or requires `searchStrategy: thorough`.

### Example 7: Custom Overrides

**Scenario**: User has a custom model/cluster and wants some overrides for security or authentication.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: override-security
spec:
  model: "deepseek-ai/DeepSeek-R1"
  overrides:
    profilingJob:
      template:
        spec:
          securityContext:
            fsGroup: 1000
            runAsGroup: 1000
            runAsNonRoot: true
            runAsUser: 1000
    dgd:
      spec:
        services:
          decode:
            envFromSecret: hf-token-secret
```

**What happens**:
- Profiling job is triggered with the given `securityContext`
- DGDs are generated with environment variables from the secret `hf-token-secret`
- Any valid Kubernetes Job spec fields can be overridden for profiling jobs
- Any valid DGD spec fields can be overridden for generated DGDs

**Result**: Allows customization of security contexts, secrets, resource requests, tolerations, node selectors, and other Kubernetes-specific configurations.

### Example 8: Enable Planner and KV Routing

**Scenario**: User wants a deployment with Planner for autoscaling and KV Router for improved routing.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: planner-kvrouter
spec:
  model: "Qwen/Qwen3-32B"
  features:
    planner: true
    kvRouter: true
```

**What happens**:
- Controller runs hardware/GPU discovery to discover hardware information, then profiles all backends
- Profiler returns a Pareto curve of configurations and a recommended DGD that contains Planner and KV router
- DGDR controller writes and applies the recommended DGD
- The generated DGD includes:
  - Planner components for runtime autoscaling and SLA monitoring
  - KV Router for optimized KV cache routing

**Result**: Deployment with autoscaling capabilities via Planner and intelligent KV cache routing. Planner monitors SLAs at runtime and adjusts replicas as needed.

---

# Implementation Phases

## Phase 0

**Release Target**: GTC 2026, Dynamo 1.0.0

**Supported API / Behavior:**

* DGDR CRD with immutable spec
* Rapid search (AIC-only profiling)
* Exploratory mode (`autoApply: false`) with ConfigMap/status export
* Quick-start path with defaults
* SLA-driven profiling with reasonable defaults
* Web UI for result exploration
* Resulting DGD has KV Router

**Not Supported:**

* Mutable DGDR (updating DGDR to trigger re-profiling)
* Thorough search (online profiling) - may be disabled for GTC
* Aggregated profiling when the user requests planner

**Key Engineering Tasks:**

* DGDR API cleanup - Remove legacy fields such as `baseDgd`, `configMapRef`, and `deploymentOverrides`
* Proper hardware detection - Auto-discover GPU hardware using DCGM instead of requiring manual configuration
* Validation - Disallow invalid configurations (e.g., offline profiling of unsupported model/hardware combinations)
* Allow profiling job/pod overrides - Add `overrides` section for customization
* Support for `auto` backend - Profile all backends and select the best one
* Rapid workflow with agg support, no autoscaling - Thorough path will only support disagg
* AIC validation functions - Helper functions for returning DGDs even for unsupported models

## Phase 1

**Release Target**: Mid-late 2026

**Supported API / Behavior:**

* Thorough search with optimized online profiling
* Aggregated deployments support for thorough search
* Aggregated profiling support with Planner
* Full Planner integration (Planner included when `features.planner: true`)
* Aggregated Planner support - Planner can autoscale aggregated deployments
* Potential: mutable DGDR (updating spec triggers re-profiling, in-place DGD updates)
* Runtime ISL/OSL adaptation
* Online profiling optimizations - Reduce sweeps from ~22 to 5-10 with heuristics, early-stopping, parallel sweeping, Bayesian search
* Profiler refactor + AIC-first pipeline - Always call AIC first, consolidate logic between Profiler and AIC

**Key Engineering Tasks:**

* Aggregated profiling support - Extend Profiler and DGDR to recommend aggregated configurations
* Online profiling optimizations - Heuristics, early-stopping, parallel sweeping, Bayesian search to reduce sweep counts
* AIC-first pipeline - Always utilize AIC first, consolidate logic in AIC repository
* Mutable DGDRs - Treat DGDR as DGD lifecycle manager, updating DGDR triggers re-profiling
* Aggregated Planner support - Extend Planner for autoscaling aggregated deployments

---

# Integration Points

| Component | Role | Integration |
| --------- | ---- | ----------- |
| **AIC** | Simulation-based configuration search | DGDR calls AIC for rapid-search modes without Planner |
| **DynamoPlannerProfiler** | On-cluster profiling and validation, setting up data for Planner | DGDR calls DPP when `searchStrategy: thorough` or when the user requires Planner |
| **DGD** | Low-level deployment spec | DGDR generates and applies DGDs |
| **Planner** | Runtime SLA tracking and autoscaling | Included in DGD only if `features.planner: true` |
| **Recipes** | Pre-built templates | Complementary; users can skip DGDR if using Recipes |

---

# RBAC & Permissions

DGDR controller requires:

- **Read**: Secrets (for model credentials)
- **Read/Write**: ConfigMaps, DynamoGraphDeployments (to create/update DGDs)
- **Read/Write/Delete**: Jobs (to manage profiling jobs)
- **Read**: Nodes (to discover GPU capacity)

---

# Frequently Asked Questions

## When should I use DGDR vs directly creating a DGD?

- **Use DGDR** when:
  - You're deploying a new model and don't know the optimal configuration
  - You want to compare different backends or configurations
  - You have SLAs and want an optimized configuration
  - You want a quick starting point

- **Use DGD directly** when:
  - You already have a known-good configuration
  - You're using a Recipe
  - You need very specific customizations that DGDR doesn't support
  - You're iterating on an existing deployment

## What's the difference between `backend: auto` and specifying a backend?

- **`backend: auto`**: DGDR profiles all available backends (vllm, sglang, trtllm) and automatically selects the best one based on your SLAs and throughput requirements. This is recommended when you're unsure which backend to use.

- **Specific backend** (e.g., `backend: trtllm`): DGDR only profiles that specific backend. Use this when you know you want a particular backend (e.g., organizational requirements, specific features).

## What happens if my SLAs can't be met?

**DGDR will NOT fail**. Instead, it will:
1. Attempt to find the closest configuration that approaches your SLAs
2. Store a warning in `status.conditions[]` with type `SLAWarning`
3. Provide the achievable SLAs in the warning message
4. Generate and deploy a DGD with the best available configuration
5. Transition to `Ready` or `Deployed` phase (not `Failed`)

**Example status**:
```yaml
status:
  phase: Deployed
  conditions:
    - type: SLAWarning
      status: "True"
      message: "Cannot meet TTFT=50ms. Closest achievable: TTFT=120ms"
```

You can then decide whether to:
- Accept the deployed configuration and monitor actual performance
- Adjust your SLAs to match what's achievable
- Use different/larger hardware
- Try a different backend or model

## How long does profiling take?

- **Rapid (`searchStrategy: rapid`)**: ~30 seconds
  - AIC simulation-based profiling
  - No GPU resources consumed

- **Thorough (`searchStrategy: thorough`)**: Minutes to hours
  - Depends on cluster size and configuration space
  - Actual on-cluster profiling with real GPUs
  - More sweeps = more time but better results

## Can I update a DGDR after creating it?

For **Phase 0 (GTC 2026)**: No, DGDRs are immutable. To change configuration:
1. Delete the old DGDR
2. Create a new DGDR with updated specs
3. Or export the generated DGD and edit it directly

For **Phase 1 and beyond**: Mutable DGDRs may be supported, allowing you to update the spec and trigger re-profiling. This is still up for discussion.

## What's the relationship between DGDR and Planner?

- **DGDR**: Generates an initial optimized configuration (one-time operation)
- **Planner**: Monitors and adjusts the deployment at runtime (continuous operation)

DGDR can generate DGDs with or without Planner:
- `features.planner: false` (default): Static configuration, no autoscaling
- `features.planner: true`: Dynamic configuration with SLA monitoring and autoscaling

## Can I use DGDR for private models?

Yes! Use the `modelCache` field to specify a PVC containing your model:

```yaml
spec:
  model: "myPrivateModel"
  modelCache:
    pvcName: "model-cache"
    pvcPath: "privateModel"
```

**Note**: Private models may not be supported by AIC. If you use `searchStrategy: rapid` with an unsupported model, DGDR will generate a working configuration with reasonable defaults, but include a warning that it may not be SLA-optimized. For SLA optimization with unsupported models, use `searchStrategy: thorough`.

## How do I know which configuration was selected?

After profiling completes, check the DGDR status:

```bash
kubectl get dgdr my-deployment -o yaml
```

Look for:
- `status.profilingResults.selectedConfig`: The chosen configuration
- `status.profilingResults.pareto`: Alternative configurations (Pareto curve)
- `status.dgdName`: The name of the generated DGD

## What's the difference between exploratory and auto-deploy?

- **Auto-deploy (`autoApply: true`, default)**:
  - DGDR automatically creates and applies the DGD
  - Fast path to production
  - You can still inspect the generated DGD afterward

- **Exploratory (`autoApply: false`)**:
  - DGDR runs profiling but doesn't deploy
  - You portforward and view a GUI with results
  - Compare configurations and manually select one
  - Export and customize the DGD before deploying

## What's the difference between warnings and errors?

**Warnings (Non-Blocking)**:
- DGDR succeeds and generates a working DGD
- Status phase: `Ready` or `Deployed`
- Warning stored in `status.conditions[]`
- Deployment works but may not be SLA-optimized
- Examples:
  - Model not supported by AIC in rapid mode
  - SLAs cannot be met with current hardware

**Errors (Blocking)**:
- DGDR fails and does not generate a DGD
- Status phase: `Failed`
- Error stored in `status.conditions[]`
- Requires user intervention
- Examples:
  - Model not found
  - Backend incompatibility
  - Insufficient cluster resources

**How to check**:
```bash
# Check for warnings
kubectl get dgdr my-deployment -o jsonpath='{.status.conditions[?(@.type=="ConfigurationWarning")]}'
kubectl get dgdr my-deployment -o jsonpath='{.status.conditions[?(@.type=="SLAWarning")]}'

# Check phase
kubectl get dgdr my-deployment -o jsonpath='{.status.phase}'
```

---

# Complete DGDR Schema

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: string
spec:
  model: string         # HF ID or served model name
  backend: string       # enum: auto, sglang, trtllm, vllm
  image: string

  workload:
    isl: int     # Input sequence length
    osl: int     # Output sequence length
  sla:
    optimizationType: string    # enum: hybrid, latency, throughput
    ttft: float                 # Time To First Token target (milliseconds)
    itl: float                  # Inter-Token Latency target (milliseconds)

  # Optional: Model cache PVC for large models
  modelCache:
    pvcName: string    # Name of PVC containing model weights
    pvcPath: string    # Subpath within PVC where model is stored

  overrides:
    profilingJob: *corev1.JobSpec
    dgd: DynamoGraphDeployment

  features:
    planner: bool
    kvRouter: bool
    mocker: bool

  searchStrategy: string    # enum: [rapid, thorough]
  autoApply: bool

status:
  phase: string         # Pending, Profiling, Ready, Deploying, Deployed, Failed
  dgdName: string
  profilingJobName: string
  conditions: []        # e.g. profiling status/step
  profilingResults:
    pareto: []          # list of Pareto-optimal configs
    selectedConfig: {}  # recommended config
  deploymentInfo:
    replicas: int
    availableReplicas: int
```

---

# Alternate Solutions

N/A - DGDR is the unified approach that consolidates multiple existing paths (Recipes, AIC, SLA-Driven Profiling) into a single declarative interface.

---

# References

* [PRD: Dynamo Model Deployment Experience](https://docs.google.com/document/d/1EYdQhwW9JhGli6O5kG6p9agVScBnQq8GEdEk-EmbXG0/edit?tab=t.0#heading=h.hpr7nykuyzmg)
* [Initial DGDR Design: SLA-Driven Profiling](https://docs.google.com/document/d/109DXimY2LOHeLRiMyBb8O_pBB5-2sODcqA8hoaRQoWY/edit?tab=t.0#heading=h.nr7ps9oa5glj)
* [Dynamo Model Deployment Experience Design doc](https://docs.google.com/document/d/1SXH51OxNpJ5QlJSw7Xs4-02RO7l5_a8Dh5AFY6cu6p8/edit?tab=t.lconu3xj0fmr#heading=h.v3l4vg2blhac)
* [Planner SLA Quickstart](https://github.com/ai-dynamo/dynamo/blob/main/docs/planner/sla_planner_quickstart.md)
* [Dynamo Framework Support Matrix](https://github.com/ai-dynamo/dynamo?tab=readme-ov-file#framework-support-matrix)
* [SLA-Driven Profiling Support Matrix](https://github.com/ai-dynamo/dynamo/blob/main/docs/benchmarks/sla_driven_profiling.md#support-matrix)
* [Current DGDR examples](https://github.com/ai-dynamo/dynamo/tree/main/benchmarks/profiler/deploy)
* [Supporting Aggregated Serving in Dynamo Planner](https://docs.google.com/document/d/1gYb_dJtmxPKhyto_LYcy0BMCg1bW66OjpsUqVDpJHq8/edit?tab=t.0#heading=h.k2qi5cqrl5t1)

---

# Background

## Current State of Dynamo Deployments

Today, users deploying Dynamo models to Kubernetes face multiple entry points with unclear guidance:

1. **Dynamo Recipes**: Model-specific, pre-configured DGDs (e.g., "the Qwen DGD")
   - Pros: Quick to deploy, battle-tested configurations
   - Cons: Limited to specific models, no customization for workload/SLAs, manual process

2. **AI Configurator (AIC)**: Simulation-based configuration generation
   - Pros: Fast (~30 seconds), supports many models and backends
   - Cons: Users must manually deploy the resulting DGD, simulation-based results may differ from actual on-cluster performance

3. **SLA-Driven Profiling**: Online profiling with real on-cluster metrics
   - Pros: Accurate performance data, optimized for specific SLAs
   - Cons: Requires understanding many parameters, running profiling jobs directly, complex setup

This fragmentation creates confusion and prevents users from having a consistent, streamlined experience. New users don't know which path to take, and the lack of a unified interface means each approach has different workflows and requirements.

## Evolution of DGDR

The DGDR was initially conceived as a facilitator for SLA-driven profiling → Planner deployment workflow. However, the scope has been broadened to become:

- A single unified path for Dynamo deployments to Kubernetes
- A smooth, integrated launch experience that accommodates various deployment scenarios
- An SLA-aware/SLA-optimized deployment orchestrator that can leverage Profiler and Planner where appropriate, but is not tightly coupled to them

The DGDR is not just a Profiler or Planner tool—it's the primary deployment interface for Dynamo on Kubernetes, supporting the full spectrum of deployment patterns from simple to complex, with or without online profiling.

## Why DGDR Solves This

DGDR provides a single, declarative interface that abstracts away profiling complexity while preserving flexibility:

1. **Unified Entry Point**: One CRD for all deployment scenarios
2. **Progressive Complexity**: Start simple (just specify model), add complexity as needed (SLAs, features, overrides)
3. **Backend Agnostic**: `backend: auto` automatically profiles and selects the best backend
4. **Flexible Profiling**: Choose between rapid (AIC, ~30s) or thorough (on-cluster, minutes-hours)
5. **Exploratory or Auto-Deploy**: `autoApply: false` for exploration with GUI, `autoApply: true` for instant deployment

The name "Request" emphasizes that this is a request for a deployment, not the deployment spec itself.

## Relationship to DynamoGraphDeployments (DGDs)

A key design principle: **DGDR is the entry path, not the permanent home**.

- **DGDR**: High-level, SLA-aware interface for generating an initial configuration
- **DGD**: Low-level, fully expressive interface for long-term operation and customization

Once a DGD is generated, users of all experience levels can iterate directly on it:

- **Simple users**: Use DGDR with `autoApply: true` to instantly get a running deployment, then rely on DGD for the deployment lifecycle
- **Advanced users**: Use DGDR with `autoApply: false` to bootstrap a DGD, export it, customize it (add custom engine flags, advanced routing, extra annotations), and iterate directly on the DGD for fine-grained control

This design allows DGDR to provide a golden path to a first deployment while DGDs remain the interface for long-term customization and advanced workflows.

## Planner is Optional

Planner is a powerful feature for autoscaling and SLA monitoring, but it's not required for all deployments:

- **Early aggregated deployments** (before aggregated autoscaling is implemented in Planner)
- **Users who prefer fixed-capacity, cost-predictable setups**
- **Early deployments where Planner is not yet available**

`features.planner: false` (default) reflects this optionality. Some DGDR-generated DGDs will include Planner (for autoscaling, SLA tracking, etc.), but others will deliberately omit it.

---

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **AIC** | AI Configurator; a simulation-based tool for finding optimal configurations without on-cluster profiling |
| **Autoscaling** | Dynamic adjustment of deployment replicas or resources based on load and SLAs (typically via Planner) |
| **DGD** | DynamoGraphDeployment; a low-level, fully expressive Kubernetes CRD for deploying Dynamo models |
| **DGDR** | DynamoGraphDeploymentRequest; a high-level CRD that automates profiling and DGD generation |
| **Entry Path** | The recommended starting point for new users; in this case, DGDR is the entry path to DGDs/Dynamo Kubernetes Deployments |
| **ITL** | Inter-Token Latency; latency between consecutive tokens in generation (milliseconds) |
| **ISL** | Input Sequence Length; the length of the input prompt (number of tokens) |
| **KV Router** | A Dynamo component for routing KV cache requests |
| **Mocker** | A deployment mode for testing that simulates model inference without actual computation |
| **Pareto Curve** | Set of non-dominated configurations representing trade-offs (e.g., throughput vs. latency) |
| **Planner** | A Dynamo component that manages autoscaling and SLA monitoring at runtime |
| **Profiling** | Process of running a model under various configurations to measure performance |
| **SLA** | Service Level Agreement; target latency or throughput requirements (TTFT, ITL) |
| **TTFT** | Time-To-First-Token; latency until the first token is generated (milliseconds) |

---

## Acronyms & Abbreviations

**AIC:** AI Configurator

**DGD:** DynamoGraphDeployment

**DGDR:** DynamoGraphDeploymentRequest

**SLA:** Service Level Agreement

**TTFT:** Time-To-First-Token

**ITL:** Inter-Token Latency

**ISL:** Input Sequence Length

**OSL:** Output Sequence Length

**PVC:** Persistent Volume Claim

**AIC:** AI Configurator

**DPP:** Dynamo Planner Profiler
