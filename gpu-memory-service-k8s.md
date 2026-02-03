# GPU Memory Service Kubernetes Integration

## Summary

**Goal:** Enable resiliency against software failures for large multi-node model setups like DeepSeek R1 using shadow engines.

We propose exposing an API on the DynamoGraphDeployment (DGD) CRD that allows multiple parallel engine deployments to be co-located on the same set of GPUs. When a software failure is detected on the active instance, a "shadow" engine can wake up and take over serving requests.

At a high level this is enabled by:
- **GPU Memory Service (GMS)** - Zero-copy weight sharing between engines
- **Dynamic Resource Allocation (DRA)** - Multiple pods claiming the same GPU devices
- **Shadow Mode** - Engines enter standby without allocating KV cache

```
┌──────────────────────────────────────────────────────────────────┐
│                    2-Node Multinode Deployment                   │
│                        (TP=4, 2 GPUs/node)                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│    Node 0                           Node 1                       │
│   ┌──────────────────┐            ┌──────────────────┐           │
│   │   GPU 0   GPU 1  │            │   GPU 0   GPU 1  │           │
│   │    ▲        ▲    │            │    ▲        ▲    │           │
│   │    │        │    │            │    │        │    │           │
│   │  ┌─┴────────┴─┐  │            │  ┌─┴────────┴─┐  │           │
│   │  │    GMS     │  │            │  │    GMS     │  │           │
│   │  │   Server   │  │            │  │   Server   │  │           │
│   │  └─────┬──────┘  │            │  └─────┬──────┘  │           │
│   │        │         │            │        │         │           │
│   │   ┌────┴────┐    │            │   ┌────┴────┐    │           │
│   │   │         │    │            │   │         │    │           │
│   │   ▼         ▼    │            │   ▼         ▼    │           │
│   │ ┌───┐    ┌───┐   │            │ ┌───┐    ┌───┐   │           │
│   │ │ A │    │ B │   │            │ │ A │    │ B │   │           │
│   │ └───┘    └───┘   │            │ └───┘    └───┘   │           │
│   └──────────────────┘            └──────────────────┘           │
│                                                                  │
│       ════════════════════════════════════════════════           │
│       ║  A = Primary (active, serving)     rank 0   rank 1  ║   │
│       ║  B = Shadow  (sleeping, standby)   rank 0   rank 1  ║   │
│       ════════════════════════════════════════════════════════   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Terminology Note

This document uses two sets of terms that can be confusing:

| Term | Context | Meaning |
|------|---------|---------|
| **Leader / Worker** | Multinode models | Rank 0 is the "leader" that coordinates distributed execution (e.g., Ray head, MPI rank 0). Workers are ranks 1-N that participate in model parallelism. |
| **Primary / Shadow** | Failover | The "primary" is the active engine serving requests. The "shadow" is a standby engine ready to take over on failure. |

These are orthogonal concepts: each rank (leader or worker) has both a primary and a shadow instance.

---

## Proposed API

Enable shadow engines via a new field on the DGD service spec:

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: deepseek-r1
spec:
  services:
    worker:
      componentType: worker
      replicas: 1
      multinode:
        nodeCount: 8
      resources:
        limits:
          gpu: "8"
      
      # Enable shadow engine for failover
      enableFailover: true

      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/dynamo:vllm
          command: ["python3", "-m", "dynamo.vllm"]
          args: ["--model", "deepseek-ai/DeepSeek-R1", "--tensor-parallel-size", "64"]
```

---

## Background

### GPU Memory Service (GMS) Server

The GMS server is a per-device process that manages GPU memory allocations using CUDA's Virtual Memory Management (VMM) APIs. It enables multiple processes to share GPU memory without copying.

```
                    ┌─────────────────────────┐
                    │  Physical GPU Memory    │
                    │  (Model Weights)        │
                    └───────────┬─────────────┘
                                │
                         ┌──────┴──────┐
                         │ GMS Server  │
                         └──────┬──────┘
                                │
                    ┌───────────┴───────────┐
                    │                       │
                    ▼                       ▼
              ┌───────────┐           ┌───────────┐
              │  Primary  │           │  Shadow   │
              │  Engine   │           │  Engine   │
              └───────────┘           └───────────┘
```

Relevant capabilities:

1. **Out-of-process memory management** - The GMS server allocates physical GPU memory (`cuMemCreate`) without creating a CUDA context, allowing it to manage memory on behalf of multiple client processes.

2. **Shareable memory handles** - Physical allocations are exported as OS-level file descriptors (`cuMemExportToShareableHandle`). Clients import these handles to map the same physical memory into their own virtual address space.

3. **VA-stable sleep/wake** - Clients can unmap physical memory while preserving their virtual address reservations. On wake, they remap to the same virtual addresses, allowing CUDA graphs and pointers to remain valid.

For this design, GMS enables a shadow engine to wake up and start serving requests in seconds by remapping to the existing physical memory for weights on the device.

### Dynamic Resource Allocation (DRA)

DRA (Kubernetes 1.31+) allows multiple pods to claim the same physical devices. This is the foundation for GPU sharing between primary and shadow engines. This is required for both inter-pod and intra-pod GPU sharing.

```yaml
# ResourceClaim that can be shared across pods
apiVersion: resource.k8s.io/v1beta1
kind: ResourceClaim
metadata:
  name: shared-gpu-rank-0
spec:
  devices:
    requests:
      - name: gpu
        deviceClassName: gpu.nvidia.com
        count: 2  # 2 GPUs for this rank

---
# Both primary and shadow reference the SAME claim
apiVersion: v1
kind: Pod
metadata:
  name: rank-0-primary
spec:
  resourceClaims:
    - name: gpu
      resourceClaimName: shared-gpu-rank-0  # ← Same claim
  containers:
    - name: vllm
      resources:
        claims:
          - name: gpu

---
apiVersion: v1
kind: Pod
metadata:
  name: rank-0-shadow
spec:
  resourceClaims:
    - name: gpu
      resourceClaimName: shared-gpu-rank-0  # ← Same claim (shared!)
  containers:
    - name: vllm
      env:
        - name: GMS_MODE
          value: "shadow"
```

### GMS Shadow Mode

Shadow engines initialize differently from primary engines. To prevent OOMs while there is another engine serving inference on the same device, KV cache allocation for the shadow engine is deferred until the shadow becomes active.

| Aspect | Primary | Shadow |
|--------|---------|--------|
| **Weight Loading** | Loads from disk, writes to GMS | Imports from GMS (no disk I/O) |
| **KV Cache** | Allocates at startup | Skips allocation (deferred to wake) |
| **CUDA Graphs** | Compiled at startup | Compiled at startup |
| **Initial State** | Active, serving requests | Sleeping, waiting for wake signal |

---

## Architecture

### Components

**GMS Server** - One per rank, manages GPU memory allocations for weights across primary and shadow for devices on the rank. Primary loads weights and commits to GMS; shadow imports from GMS. Communicates with engines via Unix socket.

**Pod Unit** - A primary/shadow pair for a single rank, sharing the same GPUs via DRA. The primary serves requests while the shadow sleeps with weights ready.

**Coordinator** - DGD-wide service that manages the "serving lock" and orchestrates failover. Detects primary failures and signals shadows to wake.

### Resource Layout

```
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                                     DGD: "deepseek-r1"                                      │
├─────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                             │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────┐  ┌─────────────────┐  │
│  │           Node 0                │  │           Node 1            │  │   Coordinator   │  │
│  │                                 │  │                             │  │                 │  │
│  │  ┌───────────┐   ┌───────────┐  │  │  ┌───────────┐ ┌───────────┐│  │  • Serving lock │  │
│  │  │ Primary   │   │ Shadow    │  │  │  │ Primary   │ │ Shadow    ││  │  • Health check │  │
│  │  │ (rank 0)  │   │ (rank 0)  │  │  │  │ (rank 1)  │ │ (rank 1)  ││  │  • Wake signal  │  │
│  │  │ • Active  │   │ • Asleep  │  │  │  │ • Active  │ │ • Asleep  ││  │                 │  │
│  │  └─────┬─────┘   └─────┬─────┘  │  │  └─────┬─────┘ └─────┬─────┘│  └────────┬────────┘  │
│  │        │               │        │  │        │             │      │           │          │
│  │        └───────┬───────┘        │  │        └──────┬──────┘      │           │          │
│  │                │                │  │               │             │           │          │
│  │                ▼                │  │               ▼             │           │          │
│  │       ┌────────────────┐        │  │      ┌────────────────┐     │           │          │
│  │       │   GMS Server   │        │  │      │   GMS Server   │     │           │          │
│  │       └────────┬───────┘        │  │      └────────┬───────┘     │           │          │
│  │                │                │  │               │             │           │          │
│  │                ▼                │  │               ▼             │           │          │
│  │       ┌────────────────┐        │  │      ┌────────────────┐     │           │          │
│  │       │  GPU 0  GPU 1  │        │  │      │  GPU 0  GPU 1  │     │           │          │
│  │       │   (via DRA)    │        │  │      │   (via DRA)    │     │           │          │
│  │       └────────────────┘        │  │      └────────────────┘     │           │          │
│  │                                 │  │                             │           │          │
│  └─────────────────────────────────┘  └─────────────────────────────┘           │          │
│                                                                                 │          │
│         ◄──────────────────────── health checks / wake signals ─────────────────┘          │
│                                                                                             │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementation Details

### Weight Loading Phase

Weight loading happens once per DGD, performed by the primary replica across ranks. Each primary rank acquires an exclusive RW lock on its node's GMS, loads weights from disk/network, and commits them to GMS-managed memory.

Once all ranks have committed weights, a deployment-wide signal (e.g., a ConfigMap) indicates weight loading is complete. Shadow pods use an init container that waits on this signal before starting their main container. When shadows start, they import the already-loaded weights from GMS (no disk I/O), compile CUDA graphs, and enter sleep state.

### Global Serving Mutex

A global mutex determines which replica set (primary or shadow) is allowed to serve. One implementation: both sets attempt to write to a configmap. The winner is considered to be the "serving set" is allowed to wake up and allocate KV cache across all ranks.

The shadow engines sleep until they acquire this mutex. On failover, the mutex is released (by the coordinator) and the shadow set can claim it.

### Failure Detection

A canary endpoint can serve as the liveness/readiness probe. The probe behavior should differ based on replica mode:
- **Primary (serving):** Probe checks inference health by sending a minimal request and waiting for a response. A canary set on the leader should indicate health of all ranks.
- **Shadow (sleeping):** Probe returns healthy as long as process is alive and weights are mapped

The Kubernetes controller uses these probes to manage pod lifecycle. The Coordinator watches pods and EndpointSlices to detect when any set might have failed.

### Coordinator Responsibilities

The coordinator is a single instance in the DGD that is responsible for detecting failures in any of the engines and initiating failover (see the section below).

### Failover Process

When the Coordinator detects a failure in ANY rank of the serving set:

1. **Dump KV cache** - Send RPC to all GMS servers to release KV cache memory. This must complete before another engine can claim the mutex (shadow needs that GPU memory for its KV cache).

2. **Release mutex** - Delete the ConfigMap (or equivalent) held by the failed engine, allowing the shadow set to claim it.

3. **Coordinated restart** - Force restart across all ranks of the failed deployment. The entire replica set must restart in lockstep upon multinode failures.

---

## Alternative: Separate Pods vs Sidecar containers

We consider two patterns for GPU sharing across primary, shadow, and the GMS server. We note that both require DRA:

### Option A: Separate Pods

This is the pattern documented in the diagrams so far. The primary, shadow, and GMS server are all separate pods.


```
┌─────────────────┐     ┌─────────────────┐
│   Pod: rank-0-a │     │   Pod: rank-0-b │
│   (Primary)     │     │   (Shadow)      │
│                 │     │                 │
│ ┌─────────────┐ │     │ ┌─────────────┐ │
│ │  Container  │ │     │ │  Container  │ │
│ │   vLLM      │ │     │ │   vLLM      │ │
│ └─────────────┘ │     │ └─────────────┘ │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     │
              ResourceClaim
              (shared GPUs)
```

Pros:
- Allows for independent lifecycle between primary and shadow engines
- Pod failures are isolated
- Conducive to expanding this pattern to hardware failure scenarios

Cons:
- Engine connection to GMS happens via Unix sockets which might require a hostPath or similar WAR
- Running into issues with Grove hierarchies: https://github.com/ai-dynamo/grove/issues/390

### Option B: Sidecar Pattern

```
┌─────────────────────────────────────────────────────────┐
│                      Pod: rank-0                        │
│                                                         │
│ ┌─────────────┐   ┌─────────────┐   ┌─────────────┐     │
│ │  Container  │   │  Container  │   │  Container  │     │
│ │  Primary    │   │  Shadow     │   │    GMS      │     │
│ │   vLLM      │   │   vLLM      │   │   Server    │     │
│ └─────────────┘   └─────────────┘   └─────────────┘     │
│                                                         │
└─────────────────────────────────────────────────────────┘
                          │
                   ResourceClaim
```

**Pros:**
- Side step Grove, kai-scheduler concerns
- UDS is simplified using an emptyDir volume from the engine to the GMS

**Cons:**
- Pod failures are not isolated
- Number of containers is fixed (can't be arbitrarily scaled)

---

## Open Questions / Issues

0. Should we design with inter-pod or intra-pod GPU sharing?
1. Separate pod design: Kai-scheduler might not be fully compatible with DRA. This gap appears to be closing in the next few weeks.
2. Should the coordination just be performed by the Dynamo Kubernetes controller? Or would this be more suitable in a separate component?
3. This design does not reflect errors in the GMS server. It might be best to restart from scratch when such an event happens.
