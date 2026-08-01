# Hierchical Planner for Multi-deployment 

**Status**: Draft 

**Authors**: [daiyaanarfeen](https://github.com/daiyaanarfeen)

**Category**: Architecture

**Replaces**: [Link of previous proposal if applicable] 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

Current DynamoGraphDeployment (DGD) design assumes one worker configuration scaled by one planner with requests routed to replicas by one router, but different request streams (ISL, OSL, TTFT, ITL, etc) can have different ideal worker configurations. This proposal aims to extend DGD to support workers of different configurations. 

# Motivation

Existing design for DGD, which encompasses the deployment of one model, assumes only one configuration shared across all worker replicas, and important components (planner, router) assume worker replicas are homogeneous. However, different request streams (ISL, OSL, TTFT, ITL, etc) to the same model can have different ideal worker configurations (parallelism, GPU model, power configuration, etc). In a resource-constrained setting with dynamic request patterns, DGD must be able to optimally scale different worker configs without violating constraints and route requests across heterogeneoeus workers to maximize SLO-attainment. 

## Goals

* Tie multiple worker configs for the same model into one DGD

* Coordinate scaling of all the configs with a global planner that coordinates with local planners

* Route requests across the heterogeneous workers with a global router that routes requests to local routers

* Separate prefill and decode worker configs for more efficient scaling according to workload characteristics

# Proposal

Currently the design contains the following:

1. DGD will have multiple local planners for each P/D worker config and one global planner to coordinate resources across the local planners; planner metrics will be collected by workers instead of frontends

2. DGD will have one frontend that receives all requests to the model; requests will have SLO info in the request body

3. DGD will have one router pod containing the logic for global router and local kv-routers; router will use request SLO to route request to specific P/D workers

# Alternate Solutions

## Multi-DGD 

**Pros:**

* Simplifies implementation, component/service discovery in particular

**Cons:**

* DGDs for the same model with different worker configs do not share lifecycle 

* Complicates routing across different P/D worker pools (would be in different DGDs)

## Planner/Router logic encapuslation 

**Pros:**

* Single planner pod: simpler discovery, fewer pods in one DGD 

* Multiple router pods: simpler implementation 

**Cons:**

* Single planner pod: more complicated implementation (merge global and local planner logic into one pod)

* Multiple router pods: potentially higher latency (additional network hops before request gets routed)