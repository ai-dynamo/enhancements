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

This document outlines and defines the Common Observability Framework for Dynamo, providing a unified approach to profiling component metrics, tracing, and logging.

# Motivation

The Dynamo system currently lacks a unified approach to observability, with different components implementing their own monitoring, tracing, and logging solutions using disparate libraries. This fragmentation results in compatibility issues, performance and safety concerns, and inconsistent data formats that impact service reliability and system visibility.

Relying on various different libraries often results in interoperability problems between components, making it difficult to correlate data across the system and maintain consistent observability practices. This approach also increases maintenance costs due to varying coding styles and introduces potential performance and safety risks.

## Goals

The Common Observability Framework addresses these challenges by providing a unified approach that:

* Improves consistent system visibility
* Promotes best practices in observability across all components
* Avoids the use of disparate raw libraries that could compromise safety, performance, consistency, and maintainability

## Requirements

### [Component Metrics](./NNNN-observability-component-metrics.md)

### [Tracing](./NNNN-observability-tracing.md)

### [Logging](./NNNN-observability-logging.md)

## Non-Goals

* Legacy support (e.g., Triton)
* Malicious actor resistance (DDOS protection) on /metrics endpoints
