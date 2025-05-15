# Fault Tolerance Definitions and Test Cases

**Status**: Draft 

**Authors**: nnshah1, vikram, biswa, harrison

**Category**: Architecture 

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: nnshah1

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: [pull/4](https://github.com/ai-dynamo/enhancements/pull/4)

**Implementation PR / Tracking Issue**: TBD

# Summary

Outline and define fault tolerance definitions, goals and test cases
for dynamo.

This DEP defines fault tolerance requirements and behaviors for the
Dynamo inference serving framework. It establishes how Dynamo should
handle component failures while maintaining service continuity, and
outlines test cases to validate fault recovery mechanisms.

# Motivation

Fault tolerance is a broad area and we want to establish a common set
of terms for describing and discussing different types of fault
tolerance, establish the required behavior of dynamo in the face of
faults, and develop a plan for fault tolerant features and tests.

Having a well defined set of behaviors in the face of different types
of faults is a critical requirement for production deployment of a
distributed inference serving framework. 

Dynamo's distributed architecture with auto-scaling components,
disaggregated prefill/decode components, and stateful KV cache
management introduces multiple potential failure points. Production
deployments require clear guarantees about:

1. Service continuity during individual component failures
2. Service continuity during runtime infrastructure failures
3. Service continuity during GPU HW failure
4. Data consistency for globals state (ex: KV cache prefix tree)

Without formal fault tolerance definitions, users cannot reliably predict system behavior during failures.

## Goals

* Define different types of faults as seen from dynamo users and deployers
* Define the required behavior of dynamo components w.r.t to faults and failures.
* Define test cases and implementation changes needed to address faults.
* Define failure modes specific to Dynamo's components
* Establish resilency and recovery requirements for each failure class
* Create verifiable test cases for fault scenarios
* Preserve SLA guarantees during failure recovery
* Give guidance on how to reduce downtime and speed recovery

### Non Goals

* Define attack vectors or behavior in the face of malicious activity.
* Malicious actor resistance (DDOS protection)
* Data center disaster recovery

## Requirements

### REQ 1 Distributed Runtime Service Failure

For either a local or cloud deployment if a global distributed runtime
service such as nats.io or etcd fails:

* Dynamo components *MUST* detect the scenario, provide proper error
codes

* Dynamo deployment orchestrator *MUST* detect the scenario and
  restart the affected services

* Requests in flight that do not required global services *SHOULD*
  continue

* Components that require global state as persisted in the discovery
  and configuration plane (etcd) *MUST* rebuild the global state in
  the event of a discovery and config plane failure.
    
**Dynamo MUST** detect and withstand temporary failures of etcd or NATS services while maintaining:  
1. **Etcd Unavailability**:  
   - **MUST** continue processing existing requests for ≥X minutes  
   - **SHOULD** buffer component registration changes in memory  
   - **MUST** attempt re-registration with exponential backoff (max Xs interval)  

2. **NATS Outage**:  
   - **MUST** persist undelivered events (Kv and other) locally for ≥X minutes  
   - **SHOULD** retry requests with exponential backoff 
   - **MUST** replay queued messages in-order after reconnection  

**Recovery Requirements**:  
- **MUST** reconcile etcd state fter restoration  
- **MUST** maintain KV cache consistency post-recovery  
-

### REQ 2 Component Process Failure

**Dynamo MUST** automatically detect and recover from worker process
failures. 

Failed workers **MUST** be removed from routing tables.

**Dynamo MUST** restart workers on healthy nodes according to resource
constraints.

**Dynamo MUST** remove nodes from consideration if workers fail to
restart.

### REQ 3 LLM Model Instance Failure

**Dynamo MUST** automatically detect and recover from LLM worker process
failures. 

**Dynamo MUST** automatically transition queued and inflight requests
to healthy instances for a fixed duration before failing. 

**Dynamo SHOULD** transition partial state of inflight requests (kv
cache, partial outputs) to healthy instances during request transition.

**Dynamo MUST** automatically update routing tables, kv cache tree,
auto scaling state after recovery.

### REQ 4 GPU Failure 

**Dynamo MUST** detect and recover from GPU failure by restarting
model instances with remaining healthy GPUs.

**LLM Workers SHOULD** checkpoint kv cache periodically to system
memory and storage to speed up recovery. 

**Multi-GPU LLM Workers SHOULD** recover as quickly as possible
ideally without having to re-iniatialize all GPUs within an instance.

1. NVLINK failures -> nvl72 gpu 

2. HW GPU detection of failure 

	be able to detect which node is failing and redeploy as quickly as possible ... for sharded nodes
	

### REQ 4 Node Failure 

**Dynamo MUST** detect and recover from Node failure by restarting
model instances with remaining healthy nodes.

### REQ 5 LLM Model xP / Shard Failure

**Multi-GPU LLM Workers SHOULD** recover as quickly as possible
ideally without having to re-iniatialize all GPUs within an instance.

### REQ 6 KV Router Failure

**Dynamo MUST** fall back to random, round-robin, or pull / capacity
based routing in case of KV Router Failure.

**Dynamo SHOULD** support multiple redundant KV Router's in the
system.

### REQ 7 Frontend Failure

**Dynamo MUST** support multiple frontends behind a cluster
orchestrator such as K8s. 

### REQ 7 Inter component request failure

**Dynamo MUST** identify and retry requests that fail due to component
failure (service failure as seperate from client failures).

**Dynamo MUST** report errors that are client side vs server side in a
way that requests can be retried or failed.

### REQ 8 LLM Client Request Failure

**Dynamo SHOULD** retry requests with partial state for a specified
number of attempts with backoff.

### REQ 9 Remote Prefill Failure

**Dynamo SHOULD** retry requests with partial state for a specified
number of attempts with backoff.

**Dynamo SHOULD** fall back to local prefill after retry attempts have
been exhausted.

**Dynamo MUST** continue to handle incoming and inflight decode
requests during failure of a remote prefill request.

### REQ 10 Decode Failure

**Dynamo SHOULD** retry requests with partial state for a specified
number of attempts with backoff.

### REQ 11 Planner / Auto Scaler Behavior

**Dynamo SHOULD** automatically roll back scaling decisions that cause
KV cache utilization >95% or prefill queue depth >100 for 3
consecutive intervals.

**Dynamo SHOULD** automatically recognize failures and re-establish
planner set scaling requirements.

**Dynamo MUST** Have zero downtime as workers are scaled up and
down. In flight requests should complete successfully.

### REQ 12 Rolling Upgrades with Zero Down Time

**Dynamo MUST** support rolling upgrades of models and graphs with zero downtime.


### REQ 13 Fault Tolerance for Expert Parallelism

**LLM Workers SHOULD** offer redundancy and fault tolerance through extra experts.

**LLM Workers SHOULD** offer quick instance reinitialization /
reconfiguration by adding / removing redundant experts.


# Proposal

We'll use the following component diagram to illustrate a typical
dynamo deployment with request flow dependencies and where faults in
the system can be.

```mermaid
graph LR
    Client["Client"]
    Frontend["Frontend"]
    Router["Router"]
	Processor["Processor"]
	PrefillQueue["Remote Prefill Queue"]


    Client --> Frontend
	Frontend --> Processor
	Processor <--> Router
	
	
    %% Multiple Prefill Workers (each with 2 GPUs)
   

    subgraph Prefill1["Prefill Worker 1"]
        direction TB
        P1GPU0["GPU 0"]
        P1GPU1["GPU 1"]
    end
    subgraph Prefill2["Prefill Worker 2"]
        direction TB
        P2GPU0["GPU 0"]
        P2GPU1["GPU 1"]
    end
    subgraph Prefill3["Prefill Worker 3"]
        direction TB
        P3GPU0["GPU 0"]
        P3GPU1["GPU 1"]
    end

    %% Multiple Decode Workers (each with 4 GPUs)
    Processor --> Decode1
    Processor --> Decode2
    Processor --> Decode3

    subgraph Decode1["Decode Worker 1"]
        direction TB
        D1GPU0["GPU 0"]
        D1GPU1["GPU 1"]
        D1GPU2["GPU 2"]
        D1GPU3["GPU 3"]
    end
    subgraph Decode2["Decode Worker 2"]
        direction TB
        D2GPU0["GPU 0"]
        D2GPU1["GPU 1"]
        D2GPU2["GPU 2"]
        D2GPU3["GPU 3"]
    end
    subgraph Decode3["Decode Worker 3"]
        direction TB
        D3GPU0["GPU 0"]
        D3GPU1["GPU 1"]
        D3GPU2["GPU 2"]
        D3GPU3["GPU 3"]
    end

	Decode1 --> PrefillQueue
    Decode2 --> PrefillQueue
    Decode3 --> PrefillQueue

    %% Prefill and Decode workers can communicate (dashed lines)
    %% Prefill1 -.-> Decode1
    %% Prefill2 -.-> Decode2
    %% Prefill3 -.-> Decode3

    %% Optional: Style blocks for emphasis
    style Prefill1 stroke:#0066cc,stroke-width:2px
    style Prefill2 stroke:#0066cc,stroke-width:2px
    style Prefill3 stroke:#0066cc,stroke-width:2px
    style Decode1 stroke:#000,stroke-width:2px
    style Decode2 stroke:#000,stroke-width:2px
    style Decode3 stroke:#000,stroke-width:2px

```



Focus primarily in K8s environment and limited support in local or
slurm environments.

Focus in two main areas:

1. Dynamo Software System Resiliency. 

  Enable request transition with partial state recovery. 
  
  Enable worker restart using K8s.
  

1. LLM Worker Resiliency in face of GPU failures 

  Leverage technologies such as NVRX and see at which layer they can
  be applied most effectively.


# Implementation Details

TBD

# Implementation Phases

## Phase 1 Software System Resilience and Recovery

**Release Target**: 0.3

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to github issues\>

**Supported API / Behavior:**

* \<name and concise description of the API / behavior\>

**Not Supported:**

* \<name and concise description of the API / behavior\>


## Phase 2 LLM Worker GPU Resilience Basic

**Release Target**: 0.4

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to github issues\>

**Supported API / Behavior:**

* \<name and concise description of the API / behavior\>

**Not Supported:**

* \<name and concise description of the API / behavior\>

## Phase 3 LLM Worker GPU Resilience Improvements

**Release Target**: 0.5

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to github issues\>

**Supported API / Behavior:**

* \<name and concise description of the API / behavior\>

**Not Supported:**

* \<name and concise description of the API / behavior\>



# Alternate Solutions

**\[Required, if not applicable write N/A\]**

List out solutions that were considered but ultimately rejected. Consider free form \- but a possible format shown below.

## Alt \<\#\> \<Title\>

**Pros:**

\<bulleted list or pros describing the positive aspects of this solution\>

**Cons:**

\<bulleted list or pros describing the negative aspects of this solution\>

**Reason Rejected:**

\<bulleted list or pros describing why this option was not used\>

**Notes:**

\<optional: additional comments about this solution\>

# Additional Notes 

## Recovery Strategies
1. **Worker Health Checks**: Etcd lease-based liveness monitoring
2. **State Management**: KV cache versioning with hash chain validation
3. **Retry Queues**: Prefill queue persistence through NATS JetStream
4. **Plan Rollbacks**: TensorBoard-integrated planner decision audit trail

## Test Cases
1. **TC-1**: SIGKILL decode worker during high load
   - Verify: Planner scales up replacement within 60s
   - Verify: In-flight requests complete or restart

2. **TC-2**: Simulate network partition between prefill/decode
   - Verify: NIXL transfers buffer to local SSD
   - Verify: Queued prefills process post-recovery

3. **TC-3**: Force planner over-scale (-30% workers)
   - Verify: Auto-rollback triggers within 3 intervals
   - Verify: Metrics show stable KV cache utilization

# Alternate Solutions

## Alt 1 Immediate Shutdown on Failure
**Pros:** Simplifies failure modes  
**Cons:** Violates SLA requirements during recovery  
**Rejected:** Conflicts with REQ 2 continuity requirements

## Alt 2 External Orchestration Only  
**Pros:** Leverages Kubernetes health checks  
**Cons:** Lacks Dynamo-specific KV cache awareness  
**Rejected:** Fails REQ 3 for stateful recovery
