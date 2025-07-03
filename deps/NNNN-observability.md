# Common Observability Framework

**Status:** Draft

**Authors:** nnshah1, keivenchang, whoisj

**Category:** Architecture

**Replaces:** N/A

**Replaced By:** N/A

**Sponsor:** keivenchang

**Required Reviewers:** TBD

**Review Date:** [Date for review]

**Pull Request:** [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue:** TBD

# Summary

This document outlines and defines the Common Observability Framework, detailing its goals and test cases. The DOF allows Dynamo programmers to add and expose observability throughout the system in a common, consistent, and performant manner, avoiding the use of disparate raw libraries that could compromise safety, performance, consistency, and maintainability.

# Motivation

## Goals

The Common Observability Framework in Dynamo is designed to improve system visibility and support greater fault tolerance. Relying on various different libraries often results in compatibility issues as well as performance and safety issues, which can impact service reliability. By providing a common framework, we simplify implementation, reduce development time, and promote best practices in observability.

* Provide a common profiling endpoint for each component/process
* Provide common APIs for programmers to add profiling and tracing
* Provide a way to create a common data formats (e.g., structs and profiling types)
* Provide flexibility – allow different implementations using the same APIs
* Improve safety: use tested and well-known APIs instead of disparate libraries
* Evaluate and establish standards for logging infrastructure. While current Python logging and Rust tracing capabilities (info/debug/warn) may be adequate for some use cases, alternative solutions such as Loki warrant consideration for enhanced logging capabilities.


## Non-Goals

* Legacy support (e.g., Triton)
* Malicious actor resistance (DDOS protection) on /metrics endpoints

## Requirements

### REQ 1: Consistent Profiling Endpoint with Prometheus Support

* **Description:** Each component/process in Dynamo MUST expose an HTTP /metrics endpoint in Prometheus format to support the Prometheus server, and to support components that query the Prometheus server (e.g. Grafana and even other Dynamo components).
* **Rationale:** This ensures that all components can be monitored uniformly and profiling data can be collected without discrepancies. Prometheus output format is a widely adopted standard that enables seamless integration with existing monitoring ecosystems, including Grafana dashboards, alerting systems, and other visualization tools. This approach leverages established tooling for historical analysis, trend identification, and operational monitoring without requiring custom dashboard implementation within the framework itself.
* **Measurability:** Verify that each component has a /metrics endpoint accessible via HTTP, and that it follows a standardized format. Verify that the API can export profiling data in Prometheus format with proper metric naming conventions and labels. Test that external dashboard tools (such as Grafana) can successfully consume the Prometheus output and display meaningful visualizations. Ensure that the Prometheus endpoint responds efficiently to scraping requests.

### REQ 2: Profiling Declaration and Registration

* **Description:** Each component MUST declare a struct that contains profiling types and register what metrics they are profiling with the observability framework.
* **Rationale:** Standardizing how components declare and register their profiling structure ensures consistency in the data reported across different components and enables the framework to properly manage and expose these metrics.
* **Measurability:** Confirm that each component has a defined profiling struct, registers its metrics with the observability framework, and that the registered metrics are used for reporting profiling data.

### REQ 3: Common Profiling Interface

* **Description:** Each component MUST use the common interface for counts,  gauges, histograms, and tracing.
* **Rationale:** A common interface ensures that profiling data collection is consistent and reliable across all components.
* **Measurability:** Check that all components use the common interface for profiling operations and that the profiling data collected are consistent.

### REQ 4: Pluggable Backend Interface

* **Description:** The profiling interface (API) MUST support pluggable backends, such as Prometheus library, OpenTelemetry (OTel), and/or custom C++ implementations, etc. The common profiling libraries will be exposed through the Dynamo Rust runtime via PyO3 to ensure consistent access across both Rust and Python components.
* **Rationale:** Pluggable backends provide flexibility in how profiling data are collected and reported, enabling integration with various monitoring tools. Exposing these through the Dynamo Rust runtime ensures a unified interface regardless of the underlying implementation language.
* **Measurability:** Validate that the API can switch between different backend implementations without requiring changes to the components, and verify that both Rust and Python components can access the profiling libraries through the Dynamo runtime interface.

### REQ 5: Python Bindings

* **Description:** The common API MUST provide Python bindings to ensure compatibility with Python components in Dynamo.
* **Rationale:** Python bindings ensure that components written in Python can also utilize Rust structs (with well-defined profiling types) and interfaces, maintaining consistency across different layers.
* **Measurability:** Verify the existence and functionality of Python bindings for the profiling interface, and ensure that Python components can use these bindings to report profiling data.

### REQ 6: Support for Callback API

* **Description:** The profiling interface (API) MUST include a callback mechanism to support components that require immediate feedback instead of relying on periodic polling updates (such as those provided by the Prometheus server).
* **Rationale:** Immediate feedback allows components to react to critical profiling data changes in real-time, enhancing system responsiveness and fault tolerance. While callbacks for same-process updates may not be necessary since components can directly access shared metric structures, they are used in loosely coupled architectures (e.g. different processes, machines, or pods). Additionally, explicit callback definitions clarify component relationships rather than inferring them through scattered call graphs in the code.
* **Measurability:** Verify that the API includes a callback mechanism and that components can register for immediate feedback. Test that callbacks provide timely updates when profiling data changes, especially across process or network boundaries.

### REQ 7: Support for Composite Profiling

* **Description:** The profiling interface (API) MUST support composite profiling data, such as average, minimum, and maximum values, for certain clients that need to react to changes quickly, without having to wait for Prometheus to poll periodically.
* **Rationale:** Providing immediate access to composite profiling data enables components to make timely decisions based on real-time data, improving overall system performance and responsiveness.
* **Measurability:** Confirm that the API can calculate and provide composite profiling data (avg, min, max) on-demand. Test that clients, such as the Planner, can access these profiling data directly through the API without relying on periodic polling.

These requirements ensure that the observability framework in Dynamo is consistent, flexible, and easy to implement across all components, supporting the overall goal of enhanced fault tolerance and system reliability.


# Proposal

The proposal is to create a common library that allows each component/process to:

* Expose metrics and/or health profiling data on an HTTP endpoint.
* Create an observability struct containing profiling data (e.g., incr counter, gauge, and histogram).
* Call a common API that mutates the observability struct.
* Automatically export the observability struct to a Prometheus key-val format.

Below is a diagram showing the architecture of the observability framework. The design uses trait-based abstractions to support pluggable backends: 1) Core traits (`MetricContainer`, `MetricCounter`, `MetricGauge`) define the common interface for metric operations, 2) Backend-specific implementations (Prometheus and OpenTelemetry) provide concrete implementations of these traits, and 3) The container pattern allows components to create and manage metrics through a unified API regardless of the underlying backend.

```mermaid
classDiagram
    %% Core Traits
    class MetricContainer {
        <<trait>>
        +create_counter()
        +create_gauge()
        +get_metrics()
    }
    
    class MetricCounter {
        <<trait>>
        +inc()
        +get_value()
    }
    
    class MetricGauge {
        <<trait>>
        +set()
        +get_value()
    }
    
    %% Implementations
    class PrometheusContainer {
        +new(prefix)
    }
    
    class OpenTelemetryContainer {
        +new(service_name, prefix)
    }
    
    class PrometheusCounter
    class PrometheusGauge
    class OpenTelemetryCounter
    class OpenTelemetryGauge
    
    %% Relationships
    MetricContainer <|.. PrometheusContainer
    MetricContainer <|.. OpenTelemetryContainer
    
    MetricCounter <|.. PrometheusCounter
    MetricCounter <|.. OpenTelemetryCounter
    
    MetricGauge <|.. PrometheusGauge
    MetricGauge <|.. OpenTelemetryGauge
    
    PrometheusContainer --> PrometheusCounter
    PrometheusContainer --> PrometheusGauge
    
    OpenTelemetryContainer --> OpenTelemetryCounter
    OpenTelemetryContainer --> OpenTelemetryGauge
``` 

```Rust
    // Example usage:
    pub struct ExampleServiceMetrics {
        // BaseServiceMetrics, MetricCounter, MetricGauge are part of the framework
        pub base: BaseServiceMetrics,
        pub some_specific_counter: Box<dyn MetricCounter>,
        pub some_response_ms: Box<dyn MetricGauge>,
    }

    impl ExampleServiceMetrics {
        /// Create a new ExampleServiceMetrics instance using the parent's new_prometheus
        pub fn new(prefix: &str) -> Self {
            let base = BaseServiceMetrics::new_prometheus(prefix);
            let request_counter = base.container.create_counter(
                "requests_total",
                "Total number of requests",
                &[("service", "api"), ("version", "v1")]
            );
            
            let response_time_gauge = base.container.create_gauge(
                "response_time_seconds",
                "Response time in seconds",
                &[("service", "api"), ("version", "v1")]
            );
            
            ExampleServiceMetrics {
                base,
                some_specific_counter: request_counter,
                some_response_ms: response_time_gauge,
            }
        }
    }

    #[test]
    fn test_service_metrics_struct() {
        println!("=== Testing ServiceMetrics Struct with Prometheus Backend ===");
        
        // Create a new ServiceMetrics instance
        let metrics = ExampleServiceMetrics::new("myapp");
        
        println!("Created ServiceMetrics with Prometheus backend");
        println!("Initial metrics:");
        metrics.print_metrics();
        
        // Simulate some API requests using direct access to public fields
        println!("\n--- Simulating API Requests (Direct Access) ---");
        
        // Record individual requests directly
        metrics.some_specific_counter.inc(); // Request 1
        
        // Record batch of requests directly
        metrics.some_specific_counter.inc_by(5); // 5 more requests
        
        // Set response times directly
        metrics.some_response_ms.set(0.15); // 150ms
        metrics.some_response_ms.inc(0.05); // Add 50ms
        
        println!("\nFinal metrics after simulation:");
        metrics.print_metrics();
        
        // Export to Prometheus format (for HTTP service)
        println!("\n--- Prometheus Format Export ---");
        let prometheus_output = metrics.base.container.export_prometheus();
        println!("{}", prometheus_output);
        // Expected output would look like:
        // # HELP myapp_requests_total Total number of requests
        // # TYPE myapp_requests_total counter
        // myapp_requests_total{service="api",version="v1"} 6
        // # HELP myapp_response_time_seconds Response time in seconds
        // # TYPE myapp_response_time_seconds gauge
        // myapp_response_time_seconds{service="api",version="v1"} 0.2
     }
```

# Alternate Solutions

## Alt 1: Use Third Party Libraries
**Pros:**
* Utilizes well-tested, existing libraries.
* Potentially quicker initial setup.

**Cons:**
* Profiling data may not be interoperable between components (e.g., different types and semantics).
* Changing the library would require significant refactoring.
* Increased maintenance costs due to varying coding styles among developers.
* Higher flexibility can introduce performance and safety risks.

**Reason Rejected:**
* Inconsistent profiling data and potential interoperability issues.
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
