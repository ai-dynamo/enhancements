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
1. [Unit tests](###Unit)
2. [Integration tests](###Integration)

### Nightly
Nightly tests are run against ToT of the `main` branch.

Tests that are part of `nightly` include:
1. [Unit tests](###Unit)
2. [Integration tests](###Integration)
3. [End-to-End tests](###End-to-End)

### Weekly

### QA

### Pre-Release

## Test Types

### Unit

Unit tests evaluate the functionality of a unit of code independent of the rest of the code base or dependencies. The definition of a unit of Where possible this means that any functional dependencies, e.g. service references, should be mocked to limit the scope of errors to the unit under test.

Structurally unit tests should live adjacent to the code that they test and should be written in the language of the unit of code that they are testing. Our codebase has standardized on tools based on the language used:

| Language | Unit Test Framework|
|----------|--------------------|
| Rust     | [cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) |
| Python   | [pytest](https://docs.pytest.org/en/stable/) |

From a performance perspective, each test suite of a functional code unit should take less than a second to run. In the event that the setup or execution exceeds this threshold it is an indication that either the functional unit is too large, the dependencies are not mocked appropriately, or your code is inefficient. In any of these cases an evaluation of the design is warranted. In the event that your test case is (perhaps intentionally) long running pla

### Integration

Integration tests test the functionality of a set of code with external services or on hardware but are testable from one node from within the development container.

Structurally these tests exist within a folder at the top-level and are defined in the [pytest](https://docs.pytest.org/en/stable/) framework. Necessary services should be started within test fixtures and then [subprocess](https://docs.python.org/3/library/subprocess.html) or similar is used to interact the artifacts.

#### Development cycle 
This class of testing serves as the first level proving rudimentary function of Dyanmo's features and should function as the first step when planning a new feature for a release.

When planning for a release integration tests for new features should be marked as follows:
```python
# In conftest.py
def pytest_addoption(parser):
    parser.addoption(
        "--dynamo-version",
        action="store",
        default="0.1.0",
        help="Current release version of dynamo, used to gate tests which are expected to fail before a particular release"
    )
def pytest_configure(config):
    dynamo_version = Version(config.getoption('--dynamo-version'))
    pytest.dynamo_version = dynamo_version

# In the test file
from packaging.version import Version

@pytest.marks.xfail(
    pytest.dynamo_version < Version("1.0.0rc0"),
    reason="Feature expected in 1.0.0 release.")
def test_new_feature_for_1_0_0():
    assert False
```
When all tests for a particular release version are in the state [XPASS](https://docs.pytest.org/en/stable/how-to/skipping.html#xfail-mark-test-functions-as-expected-to-fail) a `<major>.<minor>.<micro>rc0` tag can be placed and the release branch cut (this assumes dynamic version calculation based on Git tags). If when you want to cut a release branch if any of the tests for features in that release version are in the `FAILED` state then it becomes a product decision to delay those features to a future release, and updating the test along with it. 

### End-to-End

End-to-end tests run simulate user-like flows using build artifacts as the units under test in deployment scenarios. Practically this means all the steps from installation of a build artifact into an container, initializing an execution environment with programattically hardware setup within that environment, and any datasets that are necessary.

Structurally these tests exist within a folder at the top-level and are defined in the [pytest](https://docs.pytest.org/en/stable/) framework.

The execution environments, where ephemeral, will be defined in [Terraform](https://developer.hashicorp.com/terraform) and created specifically for each run of the test suite to minimize environmental factors that could impact behavior.

**Note** Need to work with engineering team to define environments

### Performance

Performance tests can be viewed as an extension of [End-to-End tests](#End-to-End) with the focus being on performance of the system as measured by an external service such as the [NVIDIA PerfAnalyzer](https://github.com/triton-inference-server/perf_analyzer). This secondary tool generates a report on metrics of interest and they are evaluated against the historical performance record to judge conformance to the test standard.

For this the data to have validity the deployment environment must be well defined, stable, and cataloged for each run. Where possible these environments will be defined in [Terraform](https://developer.hashicorp.com/terraform) and created specifically for each run of the test suite to minimize environmental factors that could impact behavior.

Structurally these tests exist within a folder at the top-level and are defined in the [pytest](https://docs.pytest.org/en/stable/) framework. ** No idea how to run PerfAnalyzer.

**Note** Need to work with engineering teams to define customer deployments
**Note** Internal deployments

### Continuous Operation Tests

## Test Coverage

### Feature Coverage

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

|Language| Tool |
|--------|------|
| Rust   | [Tarpaulin](https://github.com/xd009642/tarpaulin) |
| Python | [coverage.py](https://coverage.readthedocs.io/en/7.8.0/) |


### Tutorials / Examples

* use pytest codeblocks / mynist

## Test Runners



## Test Environments

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

**Release Target**: Date

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

