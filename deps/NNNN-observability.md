# Dynamo Observability Framework

**Status:** Draft  

**Authors:** Neelay Shah US, Keiven Chang NVIDIA US, J Wyman US  

**Category:** Architecture  

**Replaces:** N/A  

**Replaced By:** N/A  

**Sponsor:** keivenchang  

**Required Reviewers:** TBD  

**Review Date:** [Date for review]  

**Pull Request:** [Link to Pull Request of the Proposal itself]  

**Implementation PR / Tracking Issue:** TBD  

# Summary

This document outlines and defines the Dynamo Observability Framework, detailing its goals and test cases. The DOF allows Dynamo programmers to add and expose observability throughout the system in a common, consistent, and performant manner, avoiding the use of disparate raw libraries that could compromise safety, performance, and consistency.

# Motivation

## Goals
The common observability framework in Dynamo is designed to improve system visibility and support greater fault tolerance. Relying on disparate, low-level libraries often results in inconsistencies and sometimes performance and safety issues, which can impact service reliability. By providing a standardized framework, we simplify implementation, reduce development time, and promote best practices in observability.

* Consistent metrics endpoint for each component/process
* Consistent API for programmers to add metrics and tracing
* Consistent formats (e.g., structs and Prometheus outputs)
* Performant
* Flexibility – allow different implementations using the same interface
* Improved Safety: use tested and known APIs instead of disparate raw libraries

## Non-Goals

* Legacy support (e.g., Triton)
* Malicious actor resistance (DDOS protection) on /metrics endpoints
* For now, we are not focused on logging.

## Requirements

### REQ 1: Consistent Metrics Endpoint

* **Description:** Each component/process in Dynamo MUST expose an HTTP /metrics endpoint in a consistent manner.  
* **Rationale:** This ensures that all components can be monitored uniformly and metrics can be collected without discrepancies.  
* **Measurability:** Verify that each component has a /metrics endpoint accessible via HTTP, and that it follows a standardized format.

### REQ 2: Statistics Declaration

* **Description:** Each component MUST declare a struct of statistics that it will track and report.  
* **Rationale:** Standardizing the statistics structure ensures consistency in the data reported across different components.  
* **Measurability:** Confirm that each component has a defined statistics struct and that it is used for reporting metrics.

### REQ 3: Common Metrics Interface

* **Description:** Each component MUST use the common interface to implement metrics counting, gauging, histogram, and tracing functionalities.  
* **Rationale:** A common interface ensures that metrics collection is consistent and reliable across all components.  
* **Measurability:** Check that all components use the common interface for metrics operations and that the metrics collected are consistent.

### REQ 4: Pluggable Backend Interface

* **Description:** The metrics interface (API) MUST support pluggable backends, such as Prometheus library, OpenTelemetry (OTel), custom C++ implementations, etc.  
* **Rationale:** Pluggable backends provide flexibility in how metrics are collected and reported, enabling integration with various monitoring tools.  
* **Measurability:** Validate that the API can switch between different backend implementations without requiring changes to the components.

### REQ 5: Python Bindings

* **Description:** The common metrics interface MUST provide Python bindings to ensure compatibility with Python components in Dynamo.  
* **Rationale:** Python bindings ensure that components written in Python can also utilize structs (with well-defined metric types) and interfaces, maintaining consistency across the system.  
* **Measurability:** Verify the existence and functionality of Python bindings for the metrics interface, and ensure that Python components can use these bindings to report metrics.

### REQ 6: Callback API

* **Description:** The metrics interface (API) MUST include a callback mechanism for components that require immediate feedback instead of relying on periodic polling updates (such as those provided by the Prometheus server).  
* **Rationale:** Immediate feedback allows components to react to critical metrics changes in real-time, enhancing the system's responsiveness and fault tolerance capabilities.  
* **Measurability:** Verify that the API includes a callback mechanism and that components can register for immediate feedback. Test that the callback mechanism provides timely updates whenever relevant metrics change.

### REQ 7: Composite Metrics Support

* **Description:** The metrics interface (API) MUST support composite metrics, such as average, minimum, and maximum values, for certain clients (e.g., Planner) that require these metrics quickly without waiting for Prometheus to poll periodically.  
* **Rationale:** Providing immediate access to composite metrics enables components to make timely decisions based on real-time data, improving overall system performance and responsiveness.  
* **Measurability:** Confirm that the API can calculate and provide composite metrics (avg, min, max) on-demand. Test that clients, such as the Planner, can access these metrics directly through the API without relying on periodic polling.

These requirements ensure that the observability framework in Dynamo is consistent, flexible, and easy to implement across all components, supporting the overall goal of enhanced fault tolerance and system reliability.


# Proposal

The proposal is to create a common library that allows each component/process to:
* Expose metrics and/or health statistics on an HTTP endpoint.
* Create an observability struct containing statistics (e.g., incr counter, gauge, and histogram).
* Call a common API that mutates the observability struct.
* Automatically export the observability struct to a Prometheus key-val format.

Below is an ASCII diagram, showing 1) An observability struct containing pre-defined metric and tracing types, 2) common Observability APIs that take in the structs as parameters to mutate the struct (e.g., incr, gauge), and 3) A common HTTP library that also takes in the struct to expose an endpoint.

```
+-------------------------------------+
|          Observability              |
|                 Struct              |
|                                     |
|   +-----------------------------+   |
|   |       Metrics               |   |
|   |   - counter                 |   |
|   |   - gauge                   |   |
|   |   - histogram               |   |
|   +-----------------------------+   |
|   |       Tracing               |   |
|   |   - span                    |   |
|   |   - context                 |   |
|   +-----------------------------+   |
+-------------------------------------+
       |                |                 |
       |                |                 |
       v                v                 v
+-------------+   +-------------+   +----------------------+
|  Incr API   |   |  Gauge API  |   |  Common HTTP Library |
|             |   |             |   |                      |
|  - incr()   |   |  - set()    |   |  - expose endpoint() |
|             |   |             |   |                      |
|  +---------+|   |  +---------+|   |                      |
|  | Call-   ||   |  | Call-   ||   |                      |
|  | backs   ||   |  | backs   ||   |                      |
|  +---------+|   |  +---------+|   |                      |
+-------------+   +-------------+   +----------------------+
```
In this diagram:
* The `Observability Struct` encompasses both `Metrics` (like counter, gauge, histogram) and `Tracing` (like span, context).
* The `Incr API` and `Gauge API` represent common Observability APIs that take the `Observability Struct` as parameters to modify it.
* The `Common HTTP Library` also takes in the `Observability Struct` to expose an endpoint for monitoring or data collection.


# Alternate Solutions

## Alt 1: Use Particular Libraries
**Pros:**
* Utilizes well-tested, existing libraries.
* Potentially quicker initial setup.

**Cons:**
* Metrics may not be interoperable between components (e.g., different types and semantics).
* Changing the library would require significant refactoring.
* Increased maintenance costs due to varying coding styles among developers.
* Higher flexibility can introduce performance and safety risks.

**Reason Rejected:**
* Inconsistent metrics and potential interoperability issues.
* High refactoring effort if a library change is needed.
* Increased maintenance complexity and potential performance/safety concerns.

## Alt 2: Custom Library
**Pros:**
* Tailored to specific needs.
* Full control over implementation.

**Cons:**
* Requires significant time and resources to develop.
* Not suitable for immediate needs.

**Reason Rejected:**
* Immediate solution needed, and custom development is time-intensive.
* While feasible in the long term, it does not meet our current timeline requirements.
