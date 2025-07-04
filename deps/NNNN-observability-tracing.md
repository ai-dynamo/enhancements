# Observability - Tracing

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

This document outlines and defines a common tracing interface for Dynamo, providing a unified approach to distributed tracing, span management, and request correlation across components.

# Motivation

The Dynamo system currently lacks a unified approach to distributed tracing. This lack of a standard often results in compatibility issues and inconsistent trace formats that make  observability an issue.

Relying on various different libraries often results in interoperability problems between components, making it difficult to correlate traces across the system and maintain consistent tracing practices. This approach also increases maintenance costs due to varying coding styles and introduces potential performance and safety risks.

## Goals

The common tracing interface (API) addresses these challenges by providing a unified approach that:

* Improves consistent distributed tracing visibility across the system
* Promotes best practices in trace collection and span management across all components
* Avoids the use of disparate raw libraries that could compromise safety, performance, consistency, and maintainability


## Requirements

### REQ 1: Common Tracing Interface

* **Description:** Each component that wants tracing MUST use the common tracing interface for spans, traces, and context propagation.
* **Rationale:** The common tracing interface ensures that tracing data collection is consistent and reliable across all components.
* **Measurability:** Check that all components use the common tracing interface for tracing operations and that the tracing data collected are consistent.

### REQ 2: Pluggable Backends

* **Description:** The common tracing interface MUST support pluggable backends, such as OpenTelemetry (OTel). We may consider Jaeger, Zipkin, and/or custom C++ implementations, etc in the future. The common tracing libraries will be exposed through the Dynamo Rust runtime via PyO3 to ensure consistent access across both Rust and Python components.
* **Rationale:** Pluggable backends provide flexibility in how tracing data are collected and reported, enabling integration with various tracing tools. Exposing these through the Dynamo Rust runtime ensures a unified interface regardless of the underlying implementation language.
* **Measurability:** Validate that the common tracing interface can switch between different backend implementations without requiring changes to the components, and verify that both Rust and Python components can access the tracing libraries through the Dynamo runtime interface.

### REQ 3: OpenTelemetry Tracing Integration

* **Description:** The common tracing interface MUST include OpenTelemetry as one of the backends, leveraging OpenTelemetry's comprehensive tracing features including span creation, context propagation, trace correlation, and sampling strategies.
* **Rationale:** OpenTelemetry provides industry-standard distributed tracing capabilities that enable end-to-end request tracking across multiple components and services. Key features include automatic instrumentation, flexible sampling configurations, trace context propagation across process boundaries, and integration with popular tracing backends like Jaeger, Zipkin, and cloud-native solutions. This standardization ensures compatibility with existing observability ecosystems and provides rich debugging and performance analysis capabilities.
* **Measurability:** Verify that components can create and manage spans using OpenTelemetry APIs via the common tracing interface, confirm that trace context is properly propagated across component boundaries, and validate that tracing data can be exported to configured backends. Test that sampling strategies work as expected and that trace correlation enables end-to-end request tracking.

**Note:** The specific implementation details for OpenTelemetry tracing integration, including span lifecycle management, context propagation mechanisms, and backend configuration options, are currently under active development and will be detailed in a future iteration of this proposal.

### REQ 4: Python Bindings

* **Description:** The common tracing interface MUST provide Python bindings to ensure compatibility with Python components in Dynamo.
* **Rationale:** Python bindings ensure that components written in Python can also utilize Rust structs (with well-defined tracing types) and interfaces, maintaining consistency across different layers.
* **Measurability:** Verify the existence and functionality of Python bindings for the common tracing interface, and ensure that Python components can use these bindings to report tracing data.
