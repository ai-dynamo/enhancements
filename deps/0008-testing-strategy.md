# Test Strategy

**Status**: Draft

**Authors**: Harrison, Pavithra, Neelay

**Category**: Architecture 

**Replaces**: N/A

**Replaced By**: N/A

**Implementation**: N/A

**Sponsor**: nnshah1

**Required Reviewers**: Meenakshi, Neelay, Harrison, Alec

**Review Date**: 2025-09-09 

**Pull Request**: TBD

**Issue(s)**: NA

# Summary

This document proposes a taxonomy of tests, a categorization tests into life-cycle stages, a description of test environments, and a continuous integration strategy for running these tests. These will be used to form actual test plans.

# Motivation

Currently the Dynamo project has a number of different test strategies and implementations which can be confusing in particular with respect to what tests run, when, and where. There is not a guide for developers, QA or operations teams as to the general theory and basic set of tools, tests, or when and how they should be run. We need a set of guidelines and overarching structure to help form the basis for test plans.

# Goals

* Provide a common set of terms to ease communication about test plans, test results, and 

* Tests should be easy to write and run both locally and for CI

* Describe how tests cover the entire development and release life-cycle

* Tests should cover code, features, and documentation and have a report of coverage.

* Strategy must fit with the polyglot nature of dynamo (support for multiple programming languages and deployment targets)

### Non Goals

* To enumerate all tests and test cases (test plan) in this document.

## Requirements

1. Tests MUST be able to run locally as well as in CI. This is subject to appropriate hardware being available in the environment
2. Tests MUST be deterministic. Tests deemed "flaky" will be removed.
3. Tests SHOULD be written before beginning development of a new feature.

## Dynamo Testing Strategy (Intro)

Dynamo is a distributed inference serving framework designed for generative AI use cases. Like any project it has a development and release life-cycle with different testing requirements, test types that have a different goal and modules and functionality that require test coverage.

## Test Characteristics
- **Fast**: Unit tests < 10ms, Integration tests < 1s
- **Reliable**: No flaky tests, deterministic outcomes
- **Isolated**: Tests don't affect each other
- **Clear**: Test intent obvious from name and structure
- **Maintainable**: Tests updated with code changes

## Code Coverage Requirements
- **Rust**: Minimum 80% line coverage, 90% for critical paths
- **Python**: Minimum 85% line coverage, 95% for public APIs


## Testing Directory Structure

``` shell
dynamo/
├── lib/
│   ├── runtime/
│   │   ├── src/
│   │   │   └── lib.rs          # Rust code + unit tests inside
│   │   └── tests/              # Optional Rust integration tests specific to runtime
│   |   └── benches/  
│   ├── llm/
│   │   └── src/
│   │       └── lib.rs          # Unit tests here
│   │   └── tests/              # Optional Rust integration tests specific to runtime
│   |   └── benches/  
│   └── ...
├── components/
│   ├── planner/
│   │   └── tests/              # Python unit and integration tests for planner module
│   ├── backend/
│   │   └── tests/              # Python unit and integration  tests for backend module
│   └── ...
├── tests/                      # End-to-end tests
    ├── server
    ├── kvbm
    ├── ...                        # Other python end-to-end tests
    ├── benchmark/
    └── fault_tolerance/
```


## Test Types
We define 6 categories of tests: linting, unit, integration, end-to-end, benchmark, and stress.

### Linting Tests

These tests are validation against established style standards for the code base to maintain legibility. They cover the following:
- Code formatting
- Spelling
- Basic Code linting. Linting that is fast and doesn't evaluate on project dependencies.

#### Structure 
This is managed by the [pre-commit framework](https://pre-commit.com/). This allows both one-off execution and integration with git pre-commit hooks to ensure compliance at the development time.

#### Execution
Running this type of test is done though the following command: `pre-commit run -a`

### Unit Tests
Unit tests evaluate the functionality of a unit of code independent of the rest of the code base or dependencies. The definition of a unit of Where possible this means that any functional dependencies, e.g. service references, should be mocked to limit the scope of errors to the unit under test.

#### Structure

Unit tests should live adjacent to the code that they test in a `tests/` directory and should be written in the language of the unit of code that they are testing. Our codebase has standardized on tools based on the language used:

| Language | Unit Test Framework|
|----------|--------------------|
| Rust     | [cargo test](https://doc.rust-lang.org/cargo/commands/cargo-test.html) |
| Python   | [pytest](https://docs.pytest.org/en/stable/) |

Rust tests should follow the naming convention that the unit under test should appear in the test name along with any additional grouping that segments a large portion of the test, e.g. the if a test evaluated model loading specific to vSGLang in the LLM module the test name should could be `llm_model_load_sglang`. This allows selection of sglang-specific tests using `cargo test sglang`.

Python-based tests use a similar structure using [PyTest Marks](https://docs.pytest.org/en/stable/how-to/mark.html). Use of the mark `@pytest.mark.unit` is **required** on any unit test.

#### Performance Expectations

Each test suite of a functional code unit should take fewer than 15 seconds to run, excluding time required to build. In the event that the setup or execution exceeds this threshold it is an indication that either the functional unit is too large, the dependencies are not mocked appropriately, or your code is inefficient. In any of these cases an evaluation of the design is warranted.

To meet this performance criteria tests should be thread safe and written with the expectation of being executed in parallel.

### Integration Tests

Integration tests validate the functionality of a framework module with respect to interactions with external services or between modules. This covers both testing interactions between the Dynamo framework components and a framework component and a third-party dependency, such as NATS. An example of an integration test would be validating runtime framework module initialization which requires [NATS](https://nats.io/) and [etcd](https://etcd.io/) for functionality.

The primary differentiation of integration tests from [unit tests](#unit-tests) is the use of real components and the absence of mocking of response values. Simplistic models are acceptable, e.g. toy LLM models, but they should always operate within fully-functional components of the Dynamo. The primary difference with [end-to-end](#end-to-end-tests) is integration tests are driven through APIs while end-to-end tests describe end-user flows using the command line such as `agg.sh`. Or deployed in K8s.

#### Structure

Integration tests are defined in the [pytest](https://docs.pytest.org/en/stable/) framework and are located within the `tests/` directory of the component being tested, alongside the unit tests.. Use of the [pytest mark](https://docs.pytest.org/en/stable/how-to/mark.html#mark) `@pytest.mark.integration` is required.

Naming of the test files is normalized for categorization and clarity in the event of test failures. We use the standard of `test_<component>_<flow>.py`. `component` should be specific to either the framework element being tested or the CLI command which is being invoked. `flow` is left to the discretion of the developer for grouping similar functionality together, e.g. initialization of the component with different parameters. Additionally if there are utilities that are not test fixtures should go in a `<component>_utils.py` file

An example of the proposed folder and file structure for integration tests is shown below.

``` shell
dynamo/
├── ...
├── components/
│   ├── planner/
│   │   └── tests/              # Python unit and integration tests for planner module
│   ├── backend/
│   │   └── tests/              # Python unit and integration  tests for backend module
│   └── ...
├── ...
```


#### Test segmentation

Integration tests use the `@pytest.mark.integration` mark, which is required for all integration tests, as well as further classification within the integration test class. The four most prominent use cases are: hardware requirements for operation, classification of the test to a test lifecycle, worker framework type, and execution related. 

- System configuration marks
    - `@pytest.mark.gpus_needed_0`
    - `@pytest.mark.gpus_needed_1`
    - `@pytest.mark.gpus_needed_2`
- Life-cycle marks
    - `@pytest.mark.premerge`
    - `@pytest.mark.postmerge`
    - `@pytest.mark.nightly`
    - `@pytest.mark.release`
- Worker Framework marks
    - `@pytest.mark.vllm`
    - `@pytest.mark.tensorrt_llm`
    - `@pytest.mark.sglang`
- Execution specific marks
    - `@pytest.mark.fast` - Tests that execute quickly, typically using small models
    - `@pytest.mark.slow` - Tests that take a long time to run (> 10 minutes)
    - `@pytest.mark.skip(reason="Example: KV Manager is under development")` - Skip these tests
    - `@pytest.mark.xfail(reason="Expected to fail because...")` - Tests expected to fail


Usage of these marks can be chained together for example to select tests that run post-merge to `main` for the vLLM worker backend with 1 or 2 GPUs required: `pytest -m "integration and postmerge and vllm and (gpus_needed_1 or gpus_needed_2)`

#### Performance Expectations

Integration tests should provide extensive coverage of the API surface of the codebase but as they interact across API boundaries and may not be highly prallelizable wall-clock execution standards are defined according to the life-cycle mark:

| Life-cycle mark | Maximum execution time|
|-----------------|-----------------------|
|   premerge      |         5 minutes     |
|   postmerge     |         15 minutes    |
|   Nightly       |         2 hour        |
|   Release       |         4 hours       |

### End-to-End Tests

End-to-end tests are test user flows that are centered on command-line calls using the Dynamo command constructions, e.g. `dynamo serve ...` or `dynamo deploy ...`. 

#### Structure

End-to-end tests are defined using the [pytest](https://docs.pytest.org/en/stable/) framework and exist in within the top-level`tests` directory in subdirectories mimicking their usage, e.g. tests of `dynamo serve` should exist at `tests/serve`.

#### Test segmentation

##### Python Tests Segmentation (pytest)

We use **pytest markers** to categorize tests by their purpose, requirements, and execution characteristics. This helps selectively run relevant tests during development, CI/CD, and nightly/weekly runs.
End-to-end tests use the `@pytest.mark.e2e` mark, which is required for all end-to-end tests, as well as further classification within the end-to-end test class. Mirroring integration tests, the four most prominent use cases are: hardware requirements for operation, classification of the test to a test lifecycle, worker framework type, execution related.

- **System configuration marks:**
    - `@pytest.mark.gpus_needed_0`
    - `@pytest.mark.gpus_needed_1`
    - `@pytest.mark.gpus_needed_2`
- **Life-cycle marks:**
    - `@pytest.mark.premerge`
    - `@pytest.mark.postmerge`
    - `@pytest.mark.nightly`
    - `@pytest.mark.release`
- **Worker Framework marks:**
    - `@pytest.mark.vllm`
    - `@pytest.mark.tensorrt_llm`
    - `@pytest.mark.sglang`
- **Execution specific marks:**
    - `@pytest.mark.fast` - Tests that execute quickly, typically using small models
    - `@pytest.mark.slow` - Tests that take a long time to run
    - `@pytest.mark.skip(reason="Example: KV Manager is under development")` - Skip these tests
    - `@pytest.mark.xfail(reason="Expected to fail because...")` - Tests expected to fail
- **Deployment target:**
    - `@pytest.mark.k8s` - Tests of `dynamo deploy` that deploy to a Kubernetes instance
    - `@pytest.mark.slurm` - Tests of `dynamo deploy` that deploy to a Slurm instance
- **Component Specific Marks:**
  - `@pytest.mark.kvbm` – Tests for KVBM behavior.
  - `@pytest.mark.planner` – Tests for planner behavior.
  - `@pytest.mark.router` – Tests for router behavior.
- **Infrastructure Specific Marks:**
   - `@pytest.mark.h100` – wideep tests requires to be run on H100 and cannot be run on L40. Also, certain pytorch versions support compute capability 8.0 and require a higher CC. 

##### How to Run Python Tests by Marker

Run all tests with a specific marker:

```bash
pytest -m <marker_name>
```
##### Rust Tests Segmentation using Cargo features

Tests can be conditionally compiled using `#[cfg(feature = "feature_name")]`. For example:

```rust
#[cfg(feature = "gpu")]
#[test]
fn test_gpu_acceleration() {
    // GPU-specific test code here
}
```
```rust
#[cfg(feature = "nightly")]
#[test]
fn test_nightly_only_feature() {
    // Nightly-only test code here
}
```

##### How to Run Rust Tests by features
To combine features;

```rust
#[cfg(all(feature = "gpu", feature = "vllm"))]
#[test]
fn test_gpu_and_vllm() {
    // Test requiring both features
}

cargo test --features "gpu vllm"
```

#### Performance Expectations

End-to-End tests should provide extensive coverage of the user flows but as they evaluate full workflows and may not be highly prallelizable, wall-clock execution standards are defined according to the life-cycle mark:

| Life-cycle mark | Maximum execution time|
|-----------------|-----------------------|
|   premerge      |         5 minutes     |
|   postmerge     |         15 minutes    |
|   Nightly       |         2 hour        |
|   Release       |         4 hours       |

### Benchmark Tests

Benchmark tests provide a mechanism to quantify an established set of canonical deployments of Dynamo against a defined load scenarios. However unlike both unit and integration tests the definition of a passing test cannot be defined a priori and is a configuration parameter left to the user.

While the performance testing framework can provide a rudimentary success or failure response according to whether the test completes execution without errors, the gathered metrics are only valid in comparison to previous testing under the same environmental conditions. This level of control goes beyond the purview of the testing framework as the performance of a distributed system is highly dependent on the underlying hardware, network infrastructure, and existing load on the system.

This establishes these tests as acceptance tests for use of a new version in a production environment. As of writing the reports will not be published but internal standards will be established and used to determine when an unacceptable performance regression has occurred.

#### Structure

Structurally these tests are defined in the [pytest](https://docs.pytest.org/en/stable/) framework and exist in within top-level `benchmark` directory.

This makes use of performance analysis tools such as [GenAI Perf](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/genai-perf/README.html) to define the load conditions and 

Configuration parameters are needed here in order to direct the system to the deployment environment that you are targeting. While the structure of these files is not defined in this document there will be a new pytest argument `-perf-test-deployment-config` added that specifies the location and access credentials of the deployment environment.

**TODO Define the file structure for connecting with a deployment environment**

#### Segmentation

Since these tests are meant to target specific deployment use cases the following classes of marks: worker framework, deployment target:

- Worker Framework marks
    - `@pytest.mark.vllm`
    - `@pytest.mark.tensorrt_llm`
    - `@pytest.mark.sglang`
- Deployment target:
    - `@pytest.mark.k8s`
    - `@pytest.mark.slurm`

These will be used in conjunction the `-perf-test-deployment-config` pytest argument to specify which tests should be executed in the environment. 

#### Execution Time

Performance tests are snapshots of short-lived deployments to limit the exposure of the system to underlying failures which may skew the results. Thus a single test instance should run, at the upper limit, on the order of minutes excluding deployment.

However, as benchmark tests are for deployment scenarios, the setup time of different classes will be highly variable making wall-clock execution time standards challenging to enforce.

### Stress Tests

Dynamo is meant for large-scale deployments which means that stability and performance of the system needs to be maintained across various operational states and load conditions. To characterize these we define stress tests as chaos engineering-like practice to introduce randomized failures and unexpected loads.

While these could be considered an extension of [Performance Tests](#Performance-Tests) where system behavior is characterized over long durations in conditions of time-varying loading and in the presence of infrastructure failures, these are intended primarily to identify problems that arise in real-world deployments. While it is beyond the scope of this document, and Dynamo in general, to specify a multi-tiered deployment architecture to meet every use case this test type will run a typical, bare Dynamo deployment to understand its behavior which could then be supplemented for a production deployment.

We consider this class of test as informative rather than gating. It is up to the engineering team to determine if any issues uncovered in this class of testing is prohibitive for a product release. 

#### Structure

The duration and style of these tests do not lend themselves to the normal test execution frameworks that we have mentioned so far. It is left as an exercise to the engineering team to determine the best tools to deploy, manipulate, and characterize the .

Some tools that are suggested in this effort are [Chaos Monkey](https://netflix.github.io/chaosmonkey/), [GenAI Perf](https://docs.nvidia.com/deeplearning/triton-inference-server/user-guide/docs/perf_analyzer/genai-perf/README.html), and [NVIDIA Resiliency Extension](https://github.com/NVIDIA/nvidia-resiliency-ext). We also anticipate that there will be significant use of centralized logging infrastructure and metrics dashboards to understand system behavior when established baselines of performance or stability are violated for root-cause analysis.

## Testing Life cycles

Tests are grouped according to when in the development and product lifecycle they are executed.

### Pre-commit
Pre-commit tests are meant to evaluate compliance with coding standards. These are not evaluating functionality but are defined as a lifecycle as they are vital to maintaining a coherent codebase across multiple developers.

Tests that are part of `pre-commit` include:
1. [Linting Tests](#linting-tests)

#### Actions for failure

Pre-commit tests are intended to be run during git's pre-commit hook execution. As such this is the most developer-facing failure as it will block checking any code. In the event of a failure the developer of the code must alter the source to meet the guidelines. 

### Pre-merge

Pre-merge tests are required to pass before code can be merged to the `main` branch of the repository. These tests are designed to be a set of sanity and core functionality tests that have broad coverage and give confidence that a change hasn't broken core functionality.

Tests that are part of `pre-merge` include:
1. [Linting Tests](#linting-tests)
1. [Unit tests](#unit-tests)
1. [Integration tests](#integration-tests) with the `premerge` mark. Depending on the execution environment additional marks will be added by the [CI Infrastructure](#usage-in-continuous-integration-environments)
1. [End-to-end test](#end-to-end-tests) with the `premerge` mark. Depending on the execution environment additional marks will be added by the [CI Infrastructure](#usage-in-continuous-integration-environments)

#### Dynamic Discovery

A subset of pre-merge tests will be dynamically added to the the pre-merge gate based on the files changed. 

Todo: Determine how to specify this mapping and enable it for local testing.

#### Actions for failure

This class of test will be run against every pull request issued against `main` and there will be rule infrastructure which prevents merging if any of these tests fail. As such it falls to the developer(s) of the PR to fix any errors identified by the tests.

Pull request reviewers are also empowered to block merging and suggest additional testing needs if they determine the current testing is insufficient.

### Post-merge

Post-merge tests are evaluated after code is merged to `main` and are more expansive set of tests than `pre-merge` intended to catch regressions that may not be practical from a performance perspective to run on every commit.

Tests that are part of `post-merge` include:
1. [Linting Tests](#linting-tests)
1. [Unit tests](#unit-tests)
1. [Integration tests](#integration-tests) with the `premerge` and `postmerge` marks. Depending on the execution environment additional marks will be added by the [CI Infrastructure](#usage-in-continuous-integration-environments)
1. [End-to-end tests](#end-to-end-tests) with the `premerge` and `postmerge` marks. Depending on the execution environment additional marks will be added by the [CI Infrastructure](#usage-in-continuous-integration-environments)

#### Actions for failure

In the event that a pull request passes the [pre-merge](#pre-merge) lifecycle but fails in the post-merge level the Operations team or a designate, such as a bot, will inform the developer to of the offending pull request that they must issue a revert commit. This will block all other development in the repo and should be treated with the highest priority.

In the event that a developer cannot be reached a member of the Dynamo Operations team will revert the merge.

### Nightly

Nightly tests are evaluated after close of business Pacific time and provide a daily regression test with a longer performance window.

Tests that are part of `nightly` include:
1. [Linting Tests](#linting-tests)
1. [Unit tests](#unit-tests)
1. [Integration tests](#integration-tests) with the `premerge`, `postmerge`, `nightly` marks.
1. [End-to-end tests](#end-to-end-tests) with the `premerge`, `postmerge`, `nightly` marks.
1. [Benchmark tests](#benchmark-tests) with the `nightly` mark.

#### Actions for failure

In the event of a regression in the nightly level it is probable that multiple pull requests will be in the changeset since the last success. A failure here is not, generally, considered as blocking all development in the repository so a revert will not be immediately required.

Instead the operations team or a delegate, e.g. a bot, will message all of the pull requests authors in the changeset since the previous successful with a summary of the error encountered. They may, at their discretion, also provide a cursory analysis to more-closely target the source of the issue.

### Release

Release testing is run against every commit on a release branch of the form `release/<major>.<minor>.<micro>`. It is the highest 

Tests that are part of `release` include:
1. [Linting Tests](#linting-tests)
1. [Unit tests](#unit-tests)
1. [Integration tests](#integration-tests) with the `premerge`, `postmerge`, `nightly`, `release` marks.
1. [End-to-end tests](#end-to-end-tests) with the `premerge`, `postmerge`, `nightly` `release` marks.
1. [Benchmark tests](#benchmark-tests) with the `nightly` and `release` marks.
1. [Stress tests](#stress-tests) are deployed to the testbed(s) once [Linting Tests](#linting-tests),[Unit tests](#unit-tests), [Integration tests](#integration-tests), and [End-to-end tests](#end-to-end-tests) have concluded

#### Actions for failure

The release manager will be responsible for messaging the appropriate distribution channels for failures of this class. In addition to NVIDIA internal communication channels, if an external contributor has contributed code which is may be contributing to failures they will be messaged directly using GitHub.

With the bi-weekly release cadence of Dynamo failures in this class need are considered showstopping and developer who have contributed to the areas of the code under suspicion should turn their attention to diagnosis and repair as soon as practicable.

As this is the only lifecycle which includes the [stress tests](#stress-tests) and they are informative, rather than automatically gating, it is up to the Operations team to surface concerns to the broader engineering team based on standards established with the engineering team.
