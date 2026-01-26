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

This proposal reframes the **DynamoGraphDeploymentRequest (DGDR)**, a Kubernetes Custom Resource Definition (CRD), as the unified, declarative entry point for deploying Dynamo models to Kubernetes clusters. DGDR automates the journey from high-level user intent (model, backend, SLAs) to low-level Kubernetes resources (DynamoGraphDeployments), transparently handling profiling and configuration selection. DGDR is the recommended on-ramp to DynamoGraphDeployments (DGDs) for all users.

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

## Requirements

### REQ 1: Unified Declarative Interface

DGDR **MUST** provide a single, declarative, high-level interface for deploying Dynamo models. Users **MUST** be able to specify a model, backend (`any` is okay if users want the Dynamo platform to choose a backend for the user), and optional workload/SLAs, and DGDR **MUST** produce a deployable DynamoGraphDeployment without requiring manual configuration of low-level parameters.

### REQ 2: SLA-Driven Profiling

DGDR **MUST** support SLA-driven profiling. When users provide SLAs (TTFT, ITL) and workload parameters (ISL, OSL), DGDR **MUST** run profiling if possible to find configurations that satisfy or approach those SLAs, and **SHOULD** select the best configuration according to throughput and latency metrics.

### REQ 3: Optional SLAs and Workload Parameters

DGDR **MUST** allow users to omit SLAs and workload parameters. When omitted, DGDR **SHOULD** use reasonable defaults (e.g. ISL=512, OSL=128, TTFT=100ms, ITL=15ms) and still produce a valid, deployable DGD.

### REQ 4: Multiple Search Strategies

DGDR **MUST** support at least two search strategies: (a) rapid (AIC-only, ~30 seconds), and (b) thorough (real on-cluster profiling deployments, minutes to hours). Users **MUST** be able to select via `searchStrategy` field.

### REQ 5: Optional Auto-Deploy

DGDR **MUST** support both exploratory and auto-deploy modes. When `autoApply: true`, DGDR **MUST** automatically create and apply the generated DGD, also storing it in the DGDR status. When `autoApply: false`, DGDR **MUST** store profiling results and recommended DGD in status, allowing users to inspect and manually trigger deployment. Optionally, users can portforward the DGDR locally and view a GUI with profiling data with the option to export and deploy DGDRs directly from the UI.

### REQ 6: Optional Planner Integration

DGDR **MUST** allow users to optionally include Planner in the generated DGD via the `enableAutoScaling` field. When `enableAutoScaling: false` (default), the generated DGD **MUST NOT** include Planner. When `enableAutoScaling: true`, the generated DGD **MAY** include Planner for autoscaling and SLA monitoring.

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
  
  # Required: Serving backend
  backend: any                      # enum: [any, sglang, trtllm, vllm]
  
  # Required: Container image for profiling and runtime
  image: "nvcr.io/nvidia/ai-dynamo/profiler:1.0.0"
  
  # Optional: Workload parameters
  workload:
    isl: 512                        # Input sequence length (default: 512)
    osl: 128                        # Output sequence length (default: 128)
  
  # Optional: SLA targets
  sla:
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
  
  # Optional: Enable autoscaling via Planner
  enableAutoScaling: false           # default: false
  
  # Auto-deploy resulting DGD after profiling
  autoApply: true

status:
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
  
  # Profiling results
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

## Search Strategies

### Rapid Search (Default)

| Aspect | Detail |
| ------ | ------ |
| **Speed** | ~30 seconds (AIC only) |
| **Accuracy** | Simulation-based; may differ from on-cluster reality |
| **Cost** | $0, no GPU-hours |
| **Output** | Pareto curve, recommended DGD |
| **Best for** | Quick prototyping, users with baseline SLAs |

Implementation: Calls AIC with user-provided workload and SLA constraints. AIC explores the configuration space (parallelization, tensor-parallel degree, KV cache quantization, etc.) and returns ranked candidates.

### Thorough Search

| Aspect | Detail |
| ------ | ------ |
| **Speed** | Minutes to hours (depends on cluster size and config space) |
| **Accuracy** | Actual on-cluster metrics from profiling runs |
| **Cost** | GPU hours for profiling |
| **Output** | Pareto curve, recommended DGD, profiling logs |
| **Best for** | Production deployments, tight SLA requirements |

Implementation:
1. Sweeps over prefill engines with user-provided ISL (fast)
2. Sweeps over decode engines with user-provided ISL/OSL under different KV targets (slow)
3. Sweeps over selected prefill+decode engine with different loads (slowest)

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

### 1. Planner is Optional, Not the Ultimate Goal

Some DGDR-generated DGDs will include Planner (for autoscaling, SLA tracking), but others will deliberately omit it:

- Early aggregated deployments (before aggregated autoscaling is implemented)
- Users who prefer fixed-capacity, cost-predictable setups
- Early deployments where Planner is not yet available

This is controlled by the `enableAutoScaling` field (default: `false`).

### 2. DGDR is an Entry Path, Not a Permanent Interface

DGDR is the way to generate your first DGD. Once you have a DGD, users of all experience levels can iterate directly on it:

- Simple users rely on `autoApply: true` to instantly get a running deployment
- Advanced users run with `autoApply: false`, export the generated DGD, and iterate directly on DGDs for long-term customization

### 3. SLAs and Workload Parameters are Optional

Many users don't know exact SLAs upfront. Reasonable defaults (ISL=512, OSL=128, TTFT=100ms, ITL=15ms) allow quick deployment. GUI/CLI tools let users explore SLA trade-offs later if needed.

### 4. Backend Selection is Optional

`backend: any` enables DGDR to compare configurations across multiple backends and recommend the best one, rather than forcing users to pick a backend upfront.

### 5. Search Heuristics are Internal

For `searchStrategy: thorough`, internal heuristics (early stopping, Bayesian-style pruning, sweep counts) remain internal. Users only see the `rapid`/`thorough` boolean toggle.

---

# Implementation Details

## RBAC & Permissions

DGDR controller requires:

- **Read**: Secrets (for model credentials)
- **Read/Write**: ConfigMaps, DynamoGraphDeployments (to create/update DGDs)
- **Read/Write/Delete**: Jobs (to manage profiling jobs)
- **Read**: Nodes (to discover GPU capacity)

## Integration Points

| Component | Role | Integration |
| --------- | ---- | ----------- |
| **AIC** | Simulation-based configuration search | DGDR calls AIC for rapid-search modes without Planner |
| **DynamoPlannerProfiler** | On-cluster profiling and validation, setting up data for Planner | DGDR calls DPP when `searchStrategy: thorough` or when the user requires Planner |
| **DGD** | Low-level deployment spec | DGDR generates and applies DGDs |
| **Planner** | Runtime SLA tracking and autoscaling | Included in DGD only if `enableAutoScaling: true` |
| **Recipes** | Pre-built templates | Complementary; users can skip DGDR if using Recipes |

## Example Workflows

### Example 1: Quick Start (No SLAs, No Backend Preference)

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: quick-start
spec:
  model: "Qwen/Qwen3-32B"
  backend: any
  image: "nvcr.io/nvidia/ai-dynamo/profiler:1.0.0"
  autoApply: true
```

**Result**: Runs AIC with conservative defaults, generates and applies DGD in ~30 seconds. User has a running deployment and can measure performance to refine SLAs later.

### Example 2: SLA-Optimized, Auto-Deploy

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: sla-optimized
spec:
  model: "Qwen/Qwen3-32B"
  backend: trtllm
  image: "nvcr.io/nvidia/ai-dynamo/profiler:1.0.0"
  workload:
    isl: 2048
    osl: 512
  sla:
    ttft: 150
    itl: 20
  autoApply: true
```

**Result**: Calls AIC with exact SLAs and constraints, auto-deploys the best configuration.

### Example 3: Exploratory (Inspect Before Deploying)

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: exploratory
spec:
  model: "Qwen/Qwen3-32B"
  backend: trtllm
  image: "nvcr.io/nvidia/ai-dynamo/profiler:1.0.0"
  workload:
    isl: 2048
    osl: 512
  sla:
    ttft: 150
    itl: 20
  autoApply: false
```

**Result**: Runs profiling, stores Pareto curve and recommended config in status. User inspects via CLI/GUI, selects a configuration, and manually applies the DGD.

### Example 4: With Autoscaling (Planner Enabled)

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeploymentRequest
metadata:
  name: autoscaling
spec:
  model: "Qwen/Qwen3-32B"
  backend: trtllm
  image: "nvcr.io/nvidia/ai-dynamo/profiler:1.0.0"
  workload:
    isl: 2048
    osl: 512
  sla:
    ttft: 150
    itl: 20
  enableAutoScaling: true
  autoApply: true
```

**Result**: Generated DGD includes Planner components for runtime autoscaling and SLA monitoring.

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

**Not Supported:**

* Mutable DGDR (updating DGDR to trigger re-profiling)
* Thorough search (online profiling)

## Phase 1

**Release Target**: Mid-late 2026

**Supported API / Behavior:**

* Thorough search with optimized online profiling
* Aggregated deployments support for thorough search
* Planner integration (Planner included when `enableAutoScaling: true`)
* Potential: mutable DGDR (updating spec triggers re-profiling, in-place DGD updates)
* Runtime ISL/OSL adaptation

---

# References

* [Dynamo Model Deployment Experience Design doc](https://docs.google.com/document/d/1SXH51OxNpJ5QlJSw7Xs4-02RO7l5_a8Dh5AFY6cu6p8/edit?tab=t.lconu3xj0fmr#heading=h.v3l4vg2blhac)
* [SLA-Driven Profiling Design](https://docs.google.com/document/d/109DXimY2LOHeLRiMyBb8O_pBB5-2sODcqA8hoaRQoWY/edit?tab=t.0#heading=h.nr7ps9oa5glj)
* [Dynamo Planner SLA Quickstart](https://github.com/ai-dynamo/dynamo/blob/main/docs/planner/sla_planner_quickstart.md)

---

# Background

## Current State of Dynamo Deployments

Today, users deploying Dynamo models to Kubernetes face multiple entry points with unclear guidance:

1. **Dynamo Recipes**: Model-specific, pre-configured DGDs (e.g., "the Qwen DGD")
2. **AI Configurator (AIC)**: Simulation-based configuration generation, but users must manually deploy the resulting DGD
3. **SLA-Driven Profiling**: Online profiling with optimization, but requires understanding many parameters and running profiling jobs directly

This lack of a clear golden path creates friction and prevents consistent user experience.

## Why DGDR Solves This

DGDR provides a single, declarative interface that abstracts away profiling complexity. Users provide high-level intent (model, backend, SLAs), and DGDR handles the rest. The name "Request" emphasizes that this is a request for a deployment, not the deployment spec itself.

## DGDR as Entry Path, Not Permanent Home

A key insight is that DGDR is the entry path, not the permanent home. Once a DGD is generated, users can export and iterate on it directly. This allows:

- Simple users to use DGDR with `autoApply: true` and forget about profiling
- Advanced users to bootstrap with DGDR and then customize DGDs

## Planner is Optional

Planner is a powerful feature for autoscaling and SLA monitoring, but it's not required for all deployments. Early deployments may use disaggregated-only (before agg autoscaling). Cost-sensitive users may prefer fixed capacity. `enableAutoScaling: false` (default) reflects this optionality.

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
