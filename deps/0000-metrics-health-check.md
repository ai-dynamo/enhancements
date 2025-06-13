# Metrics and Health Check Probe for Dynamo Component

**Status**: Draft

**Authors**: [Zicheng Ma] 

**Category**: Architecture 

**Sponsor**: [Neelay Shah, Hongkuan Zhou, Zicheng Ma]

**Required Reviewers**: [Neelay Shah, Hongkuan Zhou, Ishan Dhanani, Kyle Kranen, Maksim Khadkevich, Alec Flowers, Biswa Ranjan Panda]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

This proposal introduces a unified HTTP endpoint infrastructure for Dynamo components, enabling comprehensive observability and monitoring capabilities at the component level. The core design centers around embedding an HTTP server within each Dynamo component to expose standardized endpoints for both metrics collection and health monitoring. This approach migrates the existing metrics monitoring system from Hongkaun's [implementation](https://github.com/ai-dynamo/dynamo/pull/1315), while simultaneously introducing robust health check mechanisms including liveness, readiness, and custom health probes. Currently, the metrics collection is implemented in the Rust frontend with Prometheus integration, but lacks a unified approach across all components so we need to migrate to a component-level HTTP endpoint approach.

The unified endpoint design provides a consistent interface across all Dynamo components, allowing external monitoring systems, 
container orchestrators (such as Kubernetes), and operational tools to interact with each component through standard HTTP 
protocols. By consolidating metrics exposure and health check functionality into a single HTTP server per component, this 
solution simplifies deployment, reduces infrastructure complexity.

# Motivation

Currently, the Dynamo runtime does not provide direct support for comprehensive metrics collection 
or standardized health check mechanisms at the component level. While some metrics reporting to 
Prometheus exists in the Rust frontend, there is no unified design for aggregating, querying, and managing metrics 
across all Dynamo components. Additionally, there is no standardized way to check the health, liveness, and readiness of individual Dynamo components, making it difficult to monitor system health and implement proper load balancing and failover mechanisms.

This lack of observability infrastructure creates several operational challenges:
- No standardized health check mechanism for container orchestration systems (e.g., Kubernetes)
- Limited visibility into component-level metrics and resource utilization
- No centralized way to query and aggregate metrics/health across the distributed system

## Goals

* Implement a unified HTTP endpoint infrastructure for Dynamo components to expose metrics and health check endpoints
* Enable customizable health checks through Python bindings while maintaining core health checks in Rust
* Support standard observability patterns compatible with container orchestration systems


## Requirements

### REQ 1 Unified HTTP Endpoint Infrastructure

The system **MUST** include a unified HTTP endpoint infrastructure for Dynamo components to expose metrics and health check endpoints

### REQ 2 Performance Mrtics Requirements for worker nodes

The metrics from rust frontend we want to monitor **MUST** include:
- Inflight/Total Request: updated when a new request arrives (and finishes for inflight)
- TTFT: reported at first chunk response
- ITL: reported at each new chunk response
- ISL: reported at first chunk response (TODO: report right after tokenization)
- OSL: reported when requests finishes

The system **MUST** provide a standardized approach to collect and expose native metrics from AI inference frameworks (e.g., vLLM, SGLang, TensorRT-LLM) through the unified HTTP endpoint, using Prometheus format as the standard metrics exposition format.

### REQ 3 Component Health Check Endpoints

Each Dynamo component **MUST** expose HTTP endpoint for health monitoring:
- `/health` - Overall component health status
We will use `/health` for both liveness and readiness probes. If there is extra health check needed from k8s operator, we can add more endpoints.


### REQ 4 Core Health Check Implementation

The Rust runtime **MUST** implement basic health checks including:
- etcd connectivity and lease status
- NATS connectivity and service registration status

### REQ 5 Extensible Health Check Framework

The system **MUST** provide Python bindings that allow users to:
- Register custom health check functions



# Architecture

## Overview

The proposed solution consists of three main components:

1. **Unified HTTP Server Port**: Each Dynamo component will embed a single HTTP server that provides a unified interface for both metrics exposure and health check endpoints, eliminating the need for multiple ports or separate servers per component.

2. **Metrics**: Component-level metrics collection and exposure through standardized HTTP endpoints, migrating from the existing approach implemented for Rust frontend where each component serves its own metrics data in standard Prometheus format.

3. **Health Check**: Comprehensive health monitoring system with both Rust-implemented core health checks (etcd, NATS connectivity) and extensible Python-binding framework for custom health checks, exposed through standard HTTP endpoints compatible with container orchestration systems.

<figure>
  <img src="imgs/metric%20and%20helath%20check%20arch.png" alt="Metrics and Health Check Architecture" width="800">
  <figcaption>Figure 1: Metrics and Health Check Architecture Overview</figcaption>
</figure>

## Unified HTTP Server Port

Each Dynamo DRT/component will embed an HTTP server when it first registers an Endpoint. The HTTP server will be used to expose metrics and health check endpoints.

Once the HTTP server is embedded, one entry contains the HTTP server port will be registered to etcd for discovery. Each DRT will have lock to avoid race condition and DRT is responsible to check whether the HTTP server has already been booted. i.e. When a DRT opens many endpoints at the same time, it will check if the HTTP server has already been booted and if not, it will boot the HTTP server and register the port to etcd.



## Metrics Architecture

To be done


## Component Health Check Architecture

Each Dynamo component will embed an HTTP server that exposes health check endpoints:

**Core Health Check Implementation (Rust):**
- **etcd Health**: Verify etcd connectivity and lease validity
- **NATS Health**: Check NATS connection and service group registration
- **Runtime Health**: Validate distributed runtime state and component registration

**Extensible Health Check Framework (Python Bindings)**

Python bindings will allow users to register custom health check functions(example from Biswa Ranjan Panda during discussion):

```python
# Custom health check example
from dynamo.runtime import liveness # implemented in rust 

@service
class MyService:
# used by rust (to renew etcd lease)
# also exposes http endpoint which will be queried from k8s
  @liveness 
  async def foo():
     self.vllm_engine.health() == HEALTHY 
```


### Rust Core Implementation design
Modification will mainly happen in the `lib/src/runtime/distributed.rs` and `lib/src/runtime/component/endpoint.rs`

Draft PR: https://github.com/ai-dynamo/dynamo/pull/1504

### Python Binding Interface design
To be done






# Background

## References

- [Kubernetes Health Check Guidelines](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Prometheus Metrics Format](https://prometheus.io/docs/instrumenting/exposition_formats/)
- [RFC-2119 - Key words for use in RFCs to Indicate Requirement Levels](https://datatracker.ietf.org/doc/html/rfc2119)

## Terminology & Definitions

| **Term**            | **Definition**                                                                               |
| :------------------ | :------------------------------------------------------------------------------------------- |
| **Health Check**    | A mechanism to verify if a component or service is functioning correctly                     |
| **Liveness Probe**  | A health check that determines if a component is running and should be restarted if failing  |
| **Readiness Probe** | A health check that determines if a component is ready to receive traffic                    |
| **Metrics Gateway** | A centralized service that collects, aggregates, and serves metrics from multiple components |
| **Scraping**        | The process of collecting metrics data from components at regular intervals                  |
| **TTFT**            | Time To First Token                                                                          |
| **ITL**             | Inter Token Latency                                                                          |
| **ISL**             | Input Sequence Length                                                                        |
| **OSL**             | Output Sequence Length                                                                       |

