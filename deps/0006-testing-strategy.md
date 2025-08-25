# Dynamo Testing Strategy

## Status
Draft

## Authors
Pavithra Vijayakrishnan

## Category
Test Design Proposal

## Required Reviewers
Meenakshi, Harrison, Neelay

---

## Overview
Dynamo is a distributed, high-throughput inference framework. This strategy defines a layered, automated, and scalable approach to testing, CI/CD, and observability, with the objective of achieving high reliability and rapid iteration.

---

## Test Architecture

### Linting & Static Analysis
- **Tools:** pre-commit, clippy, rustfmt, black, flake8, semgrep
- **Trigger:** Every commit/PR

### Unit Tests
- **Coverage Target:** 85% (Rust), 80% (Python)
- **Location:** client/common/server/tests/unit/
- **Trigger:** Every PR, push

### Integration Tests
- **Location:** tests/integration/
- **Trigger:** PRs, nightly, weekly
- **Focus:** Real service interaction, distributed scenarios

### End-to-End (E2E) Tests
- **Location:** tests/e2e/
- **Trigger:** PRs, nightly, weekly
- **Focus:** User workflows, CLI/API

### Performance/Benchmark Tests
- **Location:** benchmark/
- **Trigger:** Nightly, weekly
- **Metrics:** Latency, throughput, resource usage

### Stress/Chaos Tests
- **Location:** tests/stress/
- **Trigger:** Dedicated long-duration CI (weekly)
- **Vision:** Early detection of rare, long-tail failures

### Security Tests
- **Location:** tests/security/
- **Trigger:** PRs, nightly, weekly

---

## CI/CD Vision

### Pipeline Design Criteria
The CI/CD pipelines for Dynamo should be designed with the following criteria:
- **Fail Fast:** Early termination on critical failures, quick lint/syntax checks, and immediate feedback.
- **Parallelization:** Matrix builds, parallel test execution, and concurrent pipeline stages.
- **Caching & Efficiency:** Build and test result caching, Docker layer caching, and resource-aware scheduling.
- **Flakiness Management:** Automatic retries, quarantine for unstable tests, and flakiness analytics.
- **Environment Isolation:** Clean, containerized builds and reproducible infrastructure.
- **Rich Reporting:** Coverage trends, performance regression detection, and resource usage metrics.

These criteria ensure robust, efficient, and scalable pipelines that support rapid development and high-quality releases.

### Gated Branch Definition
A gated branch refers to any branch in the repository that is subject to branch protections and quality controls. This includes main, release, or any other protected branch, such as long-term support (LWS/PB) branches.

- **Pre-commit:** Linting, static analysis
- **PR:** Lint, unit, integration, E2E (CPU), security
- **Post-merge:** Full integration/E2E, coverage, performance smoke
- **Nightly:** All tests, all hardware, benchmarks, coverage trend
- **Weekly:** Nightly plus 24-72h stress/chaos, cross-component (using NIXL, aiperf, modelexpress), distributed scale

**Long-Standing Test CI:** A weekly pipeline for multi-day stress/chaos on dedicated runners is recommended. This enables detection of memory leaks, resource starvation, and rare distributed bugs.

**Dynamic Test Selection:** CI should adapt test selection based on code changes and historical flakiness.

---

## Monitoring & Alerting

- **Metrics:** All test jobs emit duration, pass/fail, flakiness, resource usage (Prometheus)
- **Dashboards:** Grafana for test health, coverage, flakiness, performance
- **Alerts:**
  - Critical failures: Immediate Slack/email
  - Coverage drop >2%: Block merge, alert
  - Flaky test rate >5%: Quarantine, review
  - Performance regression >5%: Alert, block release

**Test Flakiness Heatmap:** Visualization of unstable areas is recommended to prioritize engineering effort.

**Open Metrics:** All test and coverage data should be visible to the team for transparency and accountability.

---

## Directory Structure

```
dynamo/
├── client/common/server/tests/unit/
├── tests/integration/
├── tests/e2e/
├── tests/stress/
├── tests/performance/
├── tests/security/
├── benchmark/
└── .github/workflows/
```

---

## CI Variables

- DYNAMO_TEST_GPU_COUNT
- DYNAMO_TEST_FRAMEWORK
- DYNAMO_TEST_MODE
- CI_FAIL_FAST
- CI_COVERAGE_THRESHOLD
- CI_SKIP_SLOW_TESTS
- DYNAMO_LOG_LEVEL

---

## Test Segmentation and Grouping

Dynamo's test suite is segmented by component, scenario, and environment to enable targeted CI runs, clear ownership, and efficient debugging. Pytest markers are used for Python, and module/feature-based grouping is used for Rust.

### Python (pytest) Markers

Markers are used to select, group, and report on tests in CI and local runs. Example markers:

- **Lifecycle:**
  - pre_merge, nightly, weekly
- **Hardware:**
  - gpu_1, gpu_2, gpu_4, gpu_8, h100
- **Test Type:**
  - unit, integration, e2e, stress, smoke, regression, performance, scalability, distributed, flaky, slow
- **Component/Feature:**
  - kv_cache, kvbm, planner, router, api, config, logging, security, data_plane, control_plane
- **Framework/Backend:**
  - vllm, trtllm_marker, sglang, openai, custom_backend
- **Environment/Infra:**
  - k8s, slurm, docker, cloud

**pyproject.toml registration example:**
```
[pytest]
markers =
    pre_merge: marks tests to run before merging
    nightly: marks tests to run nightly
    weekly: marks tests to run weekly
    gpu_1: marks tests to run on GPU
    gpu_2: marks tests to run on 2GPUs
    gpu_4: marks tests to run on 4GPUs
    gpu_8: marks tests to run on 8GPUs
    e2e: marks tests as end-to-end tests
    integration: marks tests as integration tests
    unit: marks tests as unit tests
    stress: marks tests as stress tests
    vllm: marks tests as requiring vllm
    trtllm_marker: marks tests as requiring trtllm
    sglang: marks tests as requiring sglang
    slow: marks tests as known to be slow
    h100: marks tests to run on H100
    kvbm: marks tests for KV behavior and model determinism
    kv_cache: marks tests for key-value cache
    planner: marks tests for planner logic
    router: marks tests for router logic
    api: marks tests for API endpoints
    config: marks tests for configuration
    logging: marks tests for logging/metrics
    security: marks security/auth tests
    data_plane: marks data path tests
    control_plane: marks control/config path tests
    smoke: marks smoke tests
    regression: marks regression tests
    flaky: marks known-flaky tests
    performance: marks performance tests
    scalability: marks scalability tests
    distributed: marks distributed/multi-node tests
    k8s: marks Kubernetes tests
    slurm: marks Slurm tests
    docker: marks Dockerized tests
    cloud: marks cloud-only tests
    openai: marks OpenAI backend tests
    custom_backend: marks custom backend tests
```

**Usage Example:**
```python
@pytest.mark.integration
@pytest.mark.kv_cache
@pytest.mark.gpu_2
def test_kv_cache_multi_gpu_behavior():
    ...
```

### Rust (cargo test) Segmentation

- By module/file: Place tests in `tests/kv_cache.rs`, `tests/planner.rs`, etc.
- By function name: Use descriptive names: `test_kv_cache_basic`, `test_router_sharding`
- By features: Use Cargo features to enable/disable test groups: `cargo test --features planner`
- By #[ignore]: Mark slow or special-case tests: `#[ignore]`
- By directory: Organize by component: `server/tests/integration/`, `common/tests/unit/`

**Example:**
```rust
#[cfg(test)]
mod kv_cache_tests {
    #[test]
    fn test_kv_cache_basic() { /* ... */ }
    #[test]
    #[ignore]
    fn test_kv_cache_long_running() { /* ... */ }
}
```

### CI and Ownership

- CI pipelines use marker expressions to select relevant tests for each stage (e.g., `pytest -m "integration and planner and gpu_2"`).
- Markers support component ownership, reporting, and targeted debugging.
- Rust and Python tests are both first-class; segmentation enables fast, relevant, and reliable CI runs.
- Ownership and escalation for each test group/component are documented in the project.

---

## Forward-Looking Recommendations

- Long-standing CI for stress/chaos is established as a first-class pipeline.
- Dynamic, code-aware test selection is implemented in CI.
- Test flakiness and coverage are tracked, visualized, and used to drive engineering priorities.
- All test failures and regressions are actionable and monitored.
- Test ownership and escalation paths are documented.

---

## Goal

Enable Dynamo to deliver reliable, scalable inference with confidence, using a modern, automated, and transparent testing and CI/CD ecosystem. 

## Test Coverage and Adequacy

Test coverage is a primary metric for assessing the quality and completeness of the Dynamo test suite. The following concepts and criteria are used to define and measure coverage:

- **Test Case:** A set of input values, expected results, and any necessary pre/post conditions to evaluate a s/w unit.
- **Test Set:** A collection of related test cases.
- **Test Requirement:** A specific element or behavior of the software that a test case must satisfy or cover.
- **Coverage Criterion:** A rule or set of rules that impose test requirements on a test set.
- **Adequacy Criterion:** The set of test requirements that must be satisfied for a test suite to be considered adequate.

A test suite satisfies an adequacy criterion if:
- All tests succeed.
- Every test requirement in the criterion is satisfied by at least one test case in the suite. For us, this will be a coverage criterion and the threshold needs to be >80%.

### Coverage Metrics and Goals

Coverage is measured using multiple criteria, including:
- Line coverage
- Statement coverage
- Function/method coverage
- Condition/decision coverage
- Path coverage
- Loop coverage

The initial goal is to achieve at least 80% code coverage on the main branch. This target will evolve as the project matures and as new components are added.

### Sources of Test Requirements
- **Functional (black box, specification-based):** Derived from software specifications
- **Fault-based:** Derived from hypothesized faults and common bug patterns

### Control Flow and Reachability
Choosing the right tool is imortant. Ensure the line coverage >80 % for pytest integration tests. Ensure syntactic and semantic reachability of the function in comopnents using the right test case and logic. 
- **Line Coverage:** Not solely based on lines, but on the control flow graph (CFG) of the code
- **Syntactic Reachability:** A node is syntactically reachable from another if a path exists in the CFG
- **Semantic Reachability:** A node is semantically reachable if a path can be executed on some input (undecidable in general)
- Standard graph algorithms compute syntactic reachability; semantic reachability is not always computable


### Deterministic and Non-Deterministic Testing
- **Deterministic CFG:** All paths and outcomes are predictable and repeatable
- **Non-Deterministic CFG:** Some paths or outcomes may vary due to concurrency, randomness, or external state. Does Dynamo have non-determinism in any of the additional components ? Example based on the infarstructure and m/y availabilit do we get different outputs in plnner? Is thsi currently tested?
- Tests with non-deterministic output should be explicitly marked and handled differently in reporting and analysis
- Non-deterministic tests are essential for evaluating fault tolerance, robustness, and system behavior under uncertainty

### Fault Tolerance and Robustness
- Coverage criteria and adequacy should include tests for fault tolerance, such as simulating failures, network partitions, or resource exhaustion
- Marking and tracking non-deterministic and fault-injection tests supports continuous improvement in system resilience

Coverage metrics, adequacy criteria, and the handling of non-deterministic tests are reviewed regularly to ensure the test suite remains effective as Dynamo evolves. 

More informaton on fault tolerance and testing can be found in @Neelay Shah.

## Gating Checks and Prime Path Testing

Gating checks are essential jobs in the CI pipeline that must pass before code can progress to the next stage. These checks ensure that only builds and tests meeting minimum quality and stability standards are allowed to merge or release.

### Prime Path Definition and Role
- **Prime Path (Project Definition):** The prime path is defined here as the longest happy path through the system, ideally integrating the happy paths of all critical components (e.g., planner, router, kv cache, dynamo serve). This path represents the most comprehensive, non-redundant execution flow.
- **E2E as Prime Path:** In this strategy, E2E tests are designed to implement the prime path. These tests validate that the system's core workflows, spanning multiple components, work together as intended.
- **Gating Role:** Passing the prime path (E2E) tests is required for any code to merge or release. These tests are the main gating (functional sanity) tests in the pipeline.

### Gating Checks = Gating Build + Gating Tests + (more)
- **Gating Build:** The build job is a gating job. If the build fails, no further tests are executed.
- **Gating Tests / Sanity Tests:** The primary gating tests are the end-to-end (E2E) tests that exercise the prime path. These serve as sanity checks, ensuring that the most critical workflows function as expected.

### Application in CI Pipelines
- **[GITHUB] Pull Requests (PRs):**
  - The build job is a gating check for the default or changed framework.
  - Prime path E2E tests for the default or changed framework/component are exercised as the gating test set.
- **[GITHUB] Push to Protected Branches (main, release, LWs/PB branches):**
  - All framework builds on the default platform (e.g., x86) are gating jobs.
  - All prime path E2E tests for all frameworks on the default platform are included as gating tests.
- **[GITHUB] Nightly Pipelines:**
  - All framework builds on the default platform (e.g., x86) are gating jobs.
  - All prime path E2E tests for all frameworks on the default platform are included as gating tests.
  - Security checks are also required as gating jobs.
- **Release Pipelines:**
  - All gating checks from nightly (builds, prime path E2E tests, and security checks) are required.
  - Performance gating tests (benchmarking) are added as additional requirements for release.
  - Additional performance and security gating checks may be included as needed.

This approach ensures that only code passing the most critical, cross-component workflows is allowed to merge or release, improving overall system reliability and reducing the risk of regressions in production.

## CI Pipeline Flow and Stages

The CI/CD process for Dynamo is structured into distinct stages, each with specific triggers, checks, and alerting mechanisms:

- **Pull Request (PR) Pipeline:**
  - Triggers on every PR.
  - Runs only relevant tests for the changed framework or defaults to the main framework.
  - Executes build, sanity checks (prime path E2E), and coverage tests.
  - Fails silently (does not block merge) if any check fails.

- **Merge to Protected Branches (main, release, LWS/PB):**
  - Triggers on every merge.
  - Builds for the relevant or default framework.
  - Runs all performance tests for the chosen/default framework.
  - Alerts the latest commit author and Ops team on failure.
  - Optionally auto-reverts the commit on failure.

- **Nightly Pipeline:**
  - Triggers nightly.
  - Builds for all frameworks on the default platform.
  - Runs all performance, security, and benchmarking tests.
  - Alerts the Dynamo Ops team and sends notifications (Slack/Email) on failure.

- **Release Pipeline:**
  - Triggers on release branch events.
  - Builds for all frameworks.
  - Runs all performance, security, and benchmarking tests.
  - Alerts the Ops team and sends notifications (Slack/Email) on failure.

The following flowchart illustrates the CI pipeline stages and decision points:









