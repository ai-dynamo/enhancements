# Common Metrics Provider LIbrary
<!-- limited-template.md -->

**Status**: Draft <!-- Draft | Under Review | Approved | Replaced | Deferred | Rejected -->

**Authors**: J.Wyman (@whoisj)

**Category**: Architecture <!-- Architecture | Process | Guidelines -->

<!--
**Replaces**: [Link of previous proposal if applicable]

**Replaced By**: [Link of previous proposal if applicable]

**Sponsor**: [Name of code owner or maintainer to shepard process]
-->

**Required Reviewers**: N.Shah (@nnshah1), K.Chang (@keivenchang)

**Review Date**: [Date for review]

<!--
**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]
-->


# Summary

<!--
**\[Required\]**
-->

Common metrics collection library with bindings for Python, Rust, and C/C++ that enables cross-language sharing of registries and counters within the same process.
Provided by a reusable library with a stable ABI and support for Prometheus formatted metrics reporting.


# Motivation

<!--
**\[Required\]**

Describe the problem that needs to be addressed with enough detail for
someone familiar with the project to understand. Generally one to two
short paragraphs. Additional details can be placed in the background
section as needed. Cover **what** the issue is and **why** it needs to
be addressed. Link to github issues if relevant.
-->

Existing solutions are, generally speaking, language specific.
This means that an application comprised of C++, Rust, and Python components would require at least three separate metrics solutions each providing its own registry and counter objects.

In the case of Dynamo Runtime, a Rust based solution is used and API entrypoints have been added to the Dynamo Runtime that can be accessed from Python to enable a pseudo-cross-language solution.
The downside of this approach is that external projects need to take a dependency on Dynamo Runtime to enable centralized metrics collection and reporting.
Third-party projects are unlikely to depend on Dynamo Runtime for metrics collection, and even NVIDIA projects like TRTLLM require the ability to independent of Dynamo and therefore are unable to take a direct dependency on it.

A simple, focused library which any customer could depend on that avoided a direct dependency on Dynamo, provided cross-language, shared objects, and provided multi-language support via bindings would make the unification of metrics collection and reporting a possibility.

## Goals

<!--
**\[Optional \- if not applicable omit\]**

List out any additional goals in bullet points. Goals may be aspirational / difficult to measure but guide the proposal.
-->

- Multi-language support via language specific bindings (Python, Rust, C/C++, _others?_)

  - Bindings that feel "natural" to developers experienced with the language the bindings are provided in.

- Single, common root registry regardless of which language interacts with the library first.

- Nested registry support.

- Support the following counter types:
  - Monotonically increasing counters
  - Increment by integer value counters
  - Increment by floating-point value counters
  - Increment by time-duration counters.
  - Integer value gauges
  - Floating-point value gauges
  - Time-duration gauges

- Full support for serializing metrics as Prometheus metrics reports.

- High performance, low overhead implementation.

- Stable, forward- and backward- compatible ABI.

  - Consistent interface.

  - Actionable errors.

- Provide a library that non-Dynamo projects can adopt and therefore more easily integrate with Dynamo.

### Non Goals

<!--
**\[Optional \- if not applicable omit\]**

List out any items which are out of scope / specifically not required in bullet points. Indicates the scope of the proposal and issue being resolved.
-->

- Redesign how metric collection is done.
- Redesign how metrics reporting is done.

## Requirements

<!--
Describe the requirement in as much detail as necessary for others to understand it and how it applies to the DEP. Keep in mind that requirements should be measurable and will be used to determine if a DEP has been successfully implemented or not.

Requirement names should be prefixed using a monotonically increasing number such as “REQ 1 \<Title\>” followed by “REQ 2 \<Title\>” and so on. Use title casing when naming requirements. Requirement names should be as descriptive as possible while remaining as terse as possible.

Use all-caps, bolded terms like **MUST** and **SHOULD** when describing each requirement. See [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119) for additional information.
-->

### REQ 1: Language "Native" Bindings for Supported Languages

Provide language bindings for popular programming languages, starting with Python and Rust.
These bindings **MUST** be designed to be consistent with how other popular libraries for the language are presented.
SOL would be that a third-party developers working with the bindings are unaware that the functionality of the provided bindings is not developed using the language in question.

This includes, but is not limited to:

- Leverage a languages memory management solution.
- Providing actionable errors in a manner consistent wit the language.
- Provide documenting comments to improve IDE integration.
- Use the language preferred presentation for functionality (i.e. OOP for Python, Programmatic OOP for Rust, etc.)

### REQ 2: Cross-Language Shared Registries and Counters

Provide a mechanism so that regardless of language used, references to library objects can be accessed safely.
For example, in a program comprised of Python, Rust, and C++ code: all parts of the application interact with the same root-registry object.

### REQ 3: High-Performance, Low-Overhead Implementation

Optimized for performance and strive to minimize overhead incurred by the library.

### REQ 4: Serialization of Metric Data to Prometheus Formatted Output

Provide a simple mechanism to produce Prometheus formatted metrics serialization from any binding.

### REQ 5: Designed for Testability

Library and its bindings must be designed with testability in mind.
This includes third-party testability.

- Use abstractions where possible.
- Provide or enable the development of facades where possible.

# Proposal

<!--
**\[Required\]**

Describe the high level design / proposal. Use sub sections as needed, but start with an overview and then dig into the details. Try to provide images and diagrams to facilitate understanding.
-->

- Provide a common library via .so files (.dll for Windows) for amd64 and arm64 based machines.
- Provide wheel file for Python consumers.
- Provide a cargo for Rust consumers.
- Provide header files for C++ consumers.
- Provide a stable Application Binary Interface (ABI) such that any language wrapper is viable.

## Registries

- Library provides a singleton to a "root" metrics registry.
- Any registry can create counters and subordinate registries.
- Functionally, there is no difference between subordinate registries and the root registry, except that the root registry is a singleton.
- Supports prefixing all counter names including counters of subordinate registries.
- Supports default set of labels for all counters including counters of subordinate registries.

## Counters

- **Monotonically increasing counters**.
- **Increment by value counters** which track the number of times they've been incremented (i.e. total and count values).
- **Value set gauge counters**.
- Native support for integer, floating-point, and nanosecond counters and gauges.
- Assigned set of labels at creation.
  - Variants by label set are separate counters by design (performance optimization).
- Future support for histogram metrics (if necessary).
- Support priority levels to allow wide spread metric collection with variable verbosity reporting.

## Stable ABI

- Versioned.
- Consistent.
- Provide actionable error messages.
- Provides sufficient error information to enable language specific exception or error handling.

## Python Wrapper

- "pythonic" by design.
- Intended to "naturally" consumed by Python developers.
- Leans heavily on Python's memory management hooks to properly integrate with the language.

## Rust Wrapper

- Intended to be "naturally" consumed by Rust developers.
- Leans heavily on Rust's trait hooks to properly integrate with the language.


# Alternate Solutions

<!--
**\[Required, if not applicable write N/A\]**

List out solutions that were considered but ultimately rejected. Consider free form \- but a possible format shown below.
-->

## Alt 1: Prometheus Client Library

**Pros:**

- Preexists.
- Supports Python and Rust.

**Cons:**

- Reimplemented for each language; no common components.
- No mechanism to share registries or counters across language boundaries.
- Hashmap based implementation can cause performance bottlenecks.
- Limited ability to influence design direction or make changes to implementation.

**Reason Rejected:**

- Poor cross-language support (not "poor multi-language support").
