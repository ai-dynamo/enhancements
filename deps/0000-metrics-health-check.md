# Metrics and Health Check Probe for Dynamo Component

**Status**: Draft

**Authors**: [Zicheng Ma] 

**Category**: Architecture 

**Replaces**: [Link of previous proposal if applicable] 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

This proposal introduces a comprehensive metrics collection and health check system for Dynamo components. The design includes a centralized metrics gateway for aggregating and serving metrics data from all Dynamo components, and HTTP-based health check endpoints on each component for monitoring component health, liveness, and readiness. The solution aims to provide robust observability and monitoring capabilities while maintaining the distributed nature of the Dynamo architecture.

# Motivation

Currently, the Dynamo runtime does not provide direct support for comprehensive metrics collection or standardized health check mechanisms at the component level. While some metrics reporting to Prometheus exists in the Rust frontend, there is no centralized system for aggregating, querying, and managing metrics across all Dynamo components. Additionally, there is no standardized way to check the health, liveness, and readiness of individual Dynamo components, making it difficult to monitor system health and implement proper load balancing and failover mechanisms.

This lack of observability infrastructure creates several operational challenges:
- Difficult to monitor system performance and troubleshoot issues
- No standardized health check mechanism for container orchestration systems (e.g., Kubernetes)
- Limited visibility into component-level metrics and resource utilization
- No centralized way to query and aggregate metrics across the distributed system

## Goals

* Implement a centralized metrics gateway that can scrape, aggregate, and serve metrics from all Dynamo components
* Provide standardized HTTP-based health check endpoints on each Dynamo component
* Enable customizable health checks through Python bindings while maintaining core health checks in Rust
* Support standard observability patterns compatible with container orchestration systems
* Maintain performance and minimize overhead on core Dynamo functionality

### Non Goals
- Specific metrics aggregation algorithms (sum, average, percentiles)
- Metrics storage optimization and memory management strategies
- Advanced health check dependency modeling (e.g., component A depends on component B)
- Integration with external monitoring systems beyond basic HTTP endpoints
- To be done...

## Requirements

### REQ 1 Metrics Gateway Component

The system **MUST** include a standalone metrics gateway component that can:
- Scrape metrics from all registered Dynamo components
- Aggregate and store metrics data in memory with configurable retention
- Expose HTTP endpoints for querying metrics data
- Support standard metrics formats (e.g., Prometheus format)

### REQ 2 Performance Mrtics Requirements

The metrics we want to monitor **MUST** include:
- TO be done

### REQ 3 Component Health Check Endpoints

Each Dynamo component **MUST** expose HTTP endpoints for health monitoring:
- `/health` - Overall component health status
- `/liveness` - Component liveness probe
- `/readiness` - Component readiness probe

### REQ 4 Core Health Check Implementation

The Rust runtime **MUST** implement basic health checks including:
- etcd connectivity and lease status
- NATS connectivity and service registration status

### REQ 5 Extensible Health Check Framework

The system **MUST** provide Python bindings that allow users to:
- Register custom health check functions
- Define component-specific health criteria
- Customize health check response formats and thresholds



# Architecture

## Overview

The proposed solution consists of two main components:

1. **Metrics Gateway**: A standalone Dynamo component responsible for collecting, aggregating, and serving metrics from all Dynamo components
2. **Component Health Check Endpoints**: HTTP server embedded in each Dynamo component that exposes standardized health check endpoints

## Metrics Gateway Architecture

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
from dynamo.core import liveness # implemented in rust 

@service
class MyService:
# used by rust (to renew etcd lease)
# also exposes http endpoint which will be queried from k8s
  @liveness 
  async def foo():
     self.vllm_engine.health() == HEALTHY 
```

### Health Check Endpoints

- **`/health`**: Aggregated health status combining all registered health checks
- **`/liveness`**: Basic liveness probe (component is running and responsive)
- **`/readiness`**: Readiness probe (component is ready to serve requests)

# Implementation Details

## Metrics Gateway Component
To be done

## Component Health Check Implementation

### Rust Core Implementation design
To be done

### Python Binding Interface design
To be done





# Background

## References

- [Kubernetes Health Check Guidelines](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Prometheus Metrics Format](https://prometheus.io/docs/instrumenting/exposition_formats/)
- [RFC-2119 - Key words for use in RFCs to Indicate Requirement Levels](https://datatracker.ietf.org/doc/html/rfc2119)

## Terminology & Definitions

| **Term** | **Definition** |
| :---- | :---- |
| **Health Check** | A mechanism to verify if a component or service is functioning correctly |
| **Liveness Probe** | A health check that determines if a component is running and should be restarted if failing |
| **Readiness Probe** | A health check that determines if a component is ready to receive traffic |
| **Metrics Gateway** | A centralized service that collects, aggregates, and serves metrics from multiple components |
| **Scraping** | The process of collecting metrics data from components at regular intervals |

## Acronyms & Abbreviations

**TTFT:** Time To First Token
**ITL:** Inter Token Latency  
**ISL:** Input Sequence Length
**OSL:** Output Sequence Length
**HTTP:** HyperText Transfer Protocol
**API:** Application Programming Interface
**CPU:** Central Processing Unit

