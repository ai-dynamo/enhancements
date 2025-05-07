# Test Strategy

**Status**: Draft

**Authors**: nnshah1, harrison

**Category**: Architecture 

**Replaces**: N/A

**Replaced By**: N/A

**Implementation**: N/A

**Sponsor**: nnshah1

**Required Reviewers**: meenakshi, pavitra, anant

**Review Date**: 2025-05-05 

**Pull Request**: TBD

**Issue(s)**: NA

# Summary

Proposes a set of development and release life-cycle test stages, test
types, coverage areas, test environments and test runners for
dynamo. These will be used to form actual test plans.

# Motivation

Currently the dynamo project has a number of different test strategies
and implementations which can be confusing in particular with respect
to what tests run, when, and where. There is not a guide for
developers, QA or operations teams as to the general theory and basic
set of tools, tests, or when and how they should be run. We need a set
of guidelines and overarching structure to help form the basis for
test plans.

## Goals

* Provide a common set of terms to ease communication about test plans and results

* Tests should be easy to write and run both locally and for CI

* Tests should cover the entire development and release life-cycle 

* Test should be automated 

* Tests should cover code, features, and documentation and have a report of coverage.

* Strategy must fit with the polyglot nature of dynamo (support for
  multiple programming languages and deployment targets)

### Non Goals

* To enumerate all tests and test cases (test plan)

## Requirements

> TBD, May remove

**\[Optional \- if not applicable omit\]**

List out any additional requirements in numbered subheadings.

**\<numbered subheadings\>**

### REQ \<\#\> \<Title\>

Describe the requirement in as much detail as necessary for others to understand it and how it applies to the TEP. Keep in mind that requirements should be measurable and will be used to determine if a TEP has been successfully implemented or not.

Requirement names should be prefixed using a monotonically increasing number such as “REQ 1 \<Title\>” followed by “REQ 2 \<Title\>” and so on. Use title casing when naming requirements. Requirement names should be as descriptive as possible while remaining as terse as possible.

Use all-caps, bolded terms like **MUST** and **SHOULD** when describing each requirement. See [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119) for additional information.

# Proposal

The testing strategy will be a living document stored in the
repository detailing the decisions and definitions outlined here. This
proposal will serve as a starting point and enumerate the sections of
the living testing strategy.


## Dynamo Testing Strategy (Intro)

Dynamo is a distributed inference serving framework designed for
generative AI use cases. Like any project it has a development and
release life-cycle with different testing requirements, test types that
have a different goal and modules and functionality that require test
coverage.



## Development and Release Life-Cycle Test Stages

At each stage of development and release there are a different set of
tests that are required. To help organize this we will group tests
into categories and label them to clearly indicate where and when they
run and what part of the life-cycle they "gate". Tests can be in more
than one category.

### Pre-Commit

Pre-commit tests and gates are controlled and exercised using the [pre-commit framework](https://pre-commit.com/).

Configuration for dynamo is at the top level [pre-commit-config.yaml](https://github.com/ai-dynamo/dynamo/blob/main/.pre-commit-config.yaml).

Tests that are part of pre-commit are:

1. Code formatting
2. Spelling
3. Basic Code linting. Linting that is fast and doesn't depend on
   project dependencies.

These must include sections for all languages used in dynamo including rust and python.

Coverage: all files

Exception: 

* auto generated files

* foreign files (files from other projects used without modification)

> Consider a way to audit exceptions 

### Pre-Merge

Pre-merge tests are required to pass before code can be merged to
main. These tests are designed to be a set of sanity and core
functionality tests that have broad coverage and give confidence that
a change hasn't broken core functionality.

Tests that are part of `pre-merge` include:
1. [Unit tests](#unit)
2. [Integration tests](#integration)
3. [End-to-End tests](#end-to-end)
4. [Performance tests](#performance-tests)

### Nightly
Nightly tests are run against ToT of the `main` branch.

Tests that are part of `nightly` include:
1. [Unit tests](#unit)
2. [Integration tests](#Integration)
3. [End-to-End tests](#end-to-end)

### Weekly

### QA

### Pre-Release
Tests that are part of `pre-release` include:
1. [Unit tests](#Unit)
2. [Integration tests](#Integration)
3. [End-to-End tests](#End-to-End)
4. [Performance test](#Performance)

## Test Types

### Unit

Unit tests evaluate the functionality of a unit of code independent of the rest of the code base or dependencies. The definition of a unit of Where possible this means that any functional dependencies, e.g. service references, should be mocked to limit the scope of errors to the unit under test.

Structurally unit tests should live adjacent to the code that they test and should be written in the language of the unit of code that they are testing. Our codebase has standardized on tools based on the language used:

| Language | Unit Test Framework|
|----------|--------------------|
| Rust     | [cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) |
| Python   | [pytest](https://docs.pytest.org/en/stable/) |

From a performance perspective, each test suite of a functional code unit should take less than a second to run. In the event that the setup or execution exceeds this threshold it is an indication that either the functional unit is too large, the dependencies are not mocked appropriately, or your code is inefficient. In any of these cases an evaluation of the design is warranted. In the event that your test case is (perhaps intentionally) long running place a 

### Integration

Integration tests validate the functionality of a code module with respect to interactions with external services or between modules. An example of this class would be testing spinning up two instances of the runtime module which requires [NATS](https://nats.io/) and [etcd](https://etcd.io/) for functionality. A good short-hand for differentiating whether your test is an integration test or an [end-to-end] test is whether the interface your code uses to drive the test is a library call or a CLI-based command: if it is the former mark it as an integration test.

Structurally these tests exist within the `tests/integration` folder at the top-level of the Dynamo directory structure and are defined in the [pytest](https://docs.pytest.org/en/stable/) framework. To organize the test files the folder structure mimics the folder hierarchy for the source implementation as seen below. This allows integration tests to be selected simply by defining the `tests/integration` folder, i.e. `pytest tests/integration`, or more specific integration test targets by specifying one of more folders of interest, e.g. `pytest test/integration/lib/runtime test/integration/lib/llm`

`conftest.py` is found at each level of the hierarchy where there is configuration that is specific only the the components of within the directory tree of that folder. Fixtures for setup should be promoted and generalized where possible to permit reuse across test domains.

``` shell
# Dynamo top-level folder
tests/
└── conftest.py
└── integration/
    └── lib/
        └── conftest.py
        └── runtime/
            └── conftest.py
            └── test_runtime_initialization.py
            ...
        └── llm/
            └── conftest.py
            └── test_llm_initialization.py
            ...
        ...
    ...
```

Integration tests should utilize mark decorators of the form `@pytest.mark.needs_<dependency>` for each additional dependency that the unit under test will interact with. This can be verbose if many are required to operate the unit under test so we encourage grouping tests with the same external dependencies within a class and placing the decorator on the class.

### End-to-End

End-to-end tests run simulate top-level user experience flows. Examples of these flows include installation of artifacts from Python wheels, interacting with CLI commands such as `dynamo serve` and `dynamo build`, executing the examples from the documentation, and deployment onto infrastructure. Special care needs to be taken in these tests to ensure that the environment which in which the test executes resembles the environment a user would run them within to ensure validity.

Structurally these tests exist within a folder at the top-level and are defined in the [pytest](https://docs.pytest.org/en/stable/) framework.

```
tests/
└── conftest.py
└── e2e/
    └── examples/
        └── conftest.py
        └── test_serve_disagg.py
    └── installation/
        └── conftest.py
        └── test_ai_dynamo_runtime_wheel.py
    └── serve/
        └── conftest.py
        └── test_serve_agg.py
        └── test_serve_disagg.py
        ...
```

Execution or deployment environments, where necessary, will be defined in an infrastructure as code fashion and launched from within a pytest fixture.

**Note** Need to work with engineering team to define environments

### Performance Tests

Performance tests can be viewed as an extension of [End-to-End tests](#End-to-End) with the focus being on performance of the system as measured by an external observer such as the [NVIDIA PerfAnalyzer](https://github.com/triton-inference-server/perf_analyzer). This secondary tool generates a report on metrics of interest and they are evaluated against the historical performance record to judge conformance to the test standard.

Unlike other test suites defined so far, passing this test suite requires not only functional correctness but meeting a performance bar specific to the hardware execution environment which can be defined in either absolute (e.g. time to first token must be less than 200 ms) or relative (e.g. no more than 5% slower than the previous release) terms. The existence of a relative comparison requires a source of historical truth for previous executions. For the purposes of this document an oracle provided by a fixture will be interrogated to determine passage of a particular test. It is yet to be defined the storage engine for these comparison metrics.

Structurally these tests exist within a `benchmarks` folder at the top-level and are defined in the [pytest](https://docs.pytest.org/en/stable/) framework. Pytest marks should be used here to enable selection of tests according to the hardware targeted, system configurations, and duration.

The metrics of performance of interest include, but are not limited to: (**Note**: need metrics from tools team)
* time-to-first token
* ...
* ...

**Note** Need to work with engineering teams to define customer deployments
**Note** Internal deployments by NVIDIA may differ

### Continuous Operation Tests

Dynamo is meant for large-scale deployments which means that stability of the system over extended periods, varying loads, and infrastructure deployments should be characterized. Continuous operations tests are a subset of [Performance Tests](#Performance-Tests) where performance is characterized over long durations and/or in the presence of infrastructure disturbances. 

## Test Coverage

### Feature Coverage

**Note** need engineering input on features to t5est

* dynamo run
  * Serving model with Open AI compat endpoints locally
* dynamo build
    * ??
* dynamo serve
  * Serving model with kvrouting
  * Serving model with planner
  * Serving model with disagg
  * Serving model with disagg + planner + kvrouting
  * hello world graph
  * mock model
* dynamo deploy
  
### Code Coverage

[Unit tests](#unit) serve as the basis for determining how well our code is tested. While not gating at this time these numbers will be recorded by our CI tooling and if, over time, it is determined that our testing discipline is lacking there will be a gating function.

|Language| Tool |
|--------|------|
| Rust   | [Tarpaulin](https://github.com/xd009642/tarpaulin) |
| Python | [coverage.py](https://coverage.readthedocs.io/en/7.8.0/) |


### Tutorials / Examples

To keep our 

## Test Runners (CI execution environment)



## Test Environments

-- Define what is public and private

* CPU Only with Mock Model -> support all graphs

* GPU single node - 8 gpus

* GPU multi-node

* GPU cluster 

* GPU perf cluster

# Implementation Details

-------------------------------------------------------------------------------------------
| Life-cycle | Test Runner | Local Environment | Local Command | CI Environment | CI Command | 
|------------------------------------------------------------------------------------------- |
| pre-commit | [pre-commit framework](https://pre-commit.com/) | Host | `pre-commit run --all` | github action | [pre-merge.yml](https://github.com/ai-dynamo/dynamo/blob/main/.github/workflows/pre-merge.yml) |
| pre-merge  | 


> TBD

**\[Optional \- if not applicable omit\]**

Add additional detailed items here including interface signatures, etc. Add anything that is relevant but seems more of a detail than central to the proposal. Use sub sections / bullet points as needed. Try to provide images and diagrams to facilitate understanding. If applicable link to PR.

## Deferred to Implementation

> TBD

**\[Optional \- if not applicable omit\]**

List out items that are under discussion but that will be resolved only during implementation / code review. 

# Implementation Phases


## Phase 1 Baseline

**Release Target**: 0.3.0

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to work items, usually to JIRA tasks or user stories\>

**Supported API / Behavior:**

* Ensure formatting and linting is in pre-commit for all languages


**Not Supported:**

* \<name and concise description of the API / behavior\>

# Related Proposals

**\[Optional \- if not applicable omit\]**

* File

* File

* File

* File

* File

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

