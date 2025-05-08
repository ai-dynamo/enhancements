# Fault Tolerance Definitions

**Status**: Draft 

**Authors**: nnshah1

**Category**: Architecture 

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: nnshah1

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: [Link to Pull Request of the Proposal itself]

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

1. Service continuity during worker failures
2. Graceful degradation during infrastructure outages
3. Data consistency for KV cache routing
4. Recovery from planner miscalculations
5. Network partition resilience

Without formal fault tolerance definitions, users cannot reliably predict system behavior during failures.

## Goals

* Define different types of faults as seen from dynamo users and deployers
* Define the required behavior of dynamo components w.r.t to faults and failures.
* Define test cases and implementation changes needed to address faults.
* Define failure modes specific to Dynamo's components
* Establish recovery requirements for each failure class
* Create verifiable test cases for fault scenarios
* Preserve SLA guarantees during failure recovery

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
   - **MUST** continue processing existing requests for ≥5 minutes  
   - **SHOULD** buffer component registration changes in memory  
   - **MUST** attempt re-registration with exponential backoff (max 30s interval)  

2. **NATS Outage**:  
   - **MUST** persist undelivered KV events locally for ≥15 minutes  
   - **SHOULD** queue prefill requests in memory with disk spillover  
   - **MUST** replay queued messages in-order after reconnection  

**Recovery Requirements**:  
- **MUST** reconcile etcd state within 1 adjustment interval after restoration  
- **MUST** maintain KV cache consistency through hash-chain validation post-recovery  
- **SHOULD** prioritize replay of prefills over decodes during catch-up  


### REQ 2 Component Process Failure
**Dynamo MUST** automatically detect and recover from worker process failures within 2 adjustment intervals. Failed workers **SHOULD** be removed from routing tables within 1 metric collection interval.

### REQ 3 LLM Model Instance Failure

### REQ 4 GPU Failure 

### REQ 5 LLM Model xP / Shard Failure

### REQ 6 KV Router Failure

### REQ 7 Inter component request failure

### REQ 8 LLM Client Failure

### REQ 9 Remote Prefill Failure

### REQ 10 Decode Failure

### REQ 11 Planner Failure 

### REQ 13 Planner Misconfiguration 
**Dynamo SHOULD** automatically roll back scaling decisions that cause KV cache utilization >95% or prefill queue depth >100 for 3 consecutive intervals.


# Proposal

**\[Required\]**

Describe the high level design / proposal. Use sub sections as needed, but start with an overview and then dig into the details. Try to provide images and diagrams to facilitate understanding.

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

