# Dynamo 1.0 API and Non Feature Requirements

**Status**: Draft

**Authors**: Neelay Shah

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Motivation

As Dynamo enters GA we want to clearly articulate what "1.0" implies  in terms of API stability, component and module boundaries, and any other non feature requirements (tracing, logging, observability, naming, testing requirements).
These will underlie a basic contract that we can continue to evolve.

## Goals

Establish a simple, stable, consistent API surface for 1.0.

* Provide a mechanism to easily distinguish "experimental" APIs from "stable APIs". Remove notion of 'production vs pre production'.

* Provide stability and consistency for the primary user and developer contracts while retaining flexibility to change internal / experimental APIs / components.

* Provide a prioritized list of stable contracts and components. The priority is to harden APIs that most affect users and developers.

* Prioritized components should have clear versioned public interfaces.

* Prioritized components should be modular and support replacement, reuse, and customization.

* Prioritized Components should have a consistent DX driven from industry standards and best practices. For example: common ways of configuration, common ways of reporting metrics, common ways of tracing, common ways of reporting errors.

* Prioritized Component interfaces should support extension without requiring modification for common customization points. Each component should have a plan on how to support extensions without requiring deep changes.

* Prioritized Component interfaces should extend to any gen ai use case that is a realistic target for 2026 (multi-modal, agents, diffusion models ..)

* All internal and external APIs should be clear and well documented. Including [Agents.md](http://Agents.md) to codify the basic building blocks and how they relate.

* Prioritized components must support n and n-1 compatibility at a minimum.

### Non Goals

* Cover projects related to but not directly in the core dynamo repo.

* Harden all APIs

# Proposal

# Consistency Guarantees for Prioritized Components

* Clear directory structure and common look and feel for documentation
* Versioned public interfaces and schemas
* Consistent naming for core concepts (ex. request backend, discovery backend, event backend  or planes)
* Code Style. Python type hints everywhere as well as explicit public and private naming
  * underscore for private
* Configuration. Consistent environment variable, command line parameters, config files
  * Configuration parameters for services should be supported as command line parameters, environment variables, and potentially as config files.
* Metrics. consistent metric naming, labeling, semantics
* Public APIs and contracts will have complete testing coverage
* Public APIs will have well defined error codes, exceptions, messages
* All internal and external APIs should be clear and well documented. Including [Agents.md](http://Agents.md) to codify the basic building blocks and how they relate.
* Backwards N, N-1 Compatibility
  * DGD N,  Operator N-1
  * Frontend N-1, worker N
  * Worker N , Framework  N-1

# Support Levels

* 1.0 (Maintain backwards compatibility with suitable deprecation strategy)
  * Service / Cmd Line
    * Environment variables and command line flags.
    * Internal libraries not in scope
    * Public APIs, Metrics, Flags, are in scope
  * Library
    * Curated public python api
  * Schemas (yamls, json, etc.)
* Experimental (No backwards compatibility guarantee)
* Internal (No backwards compatibility guarantee)

# Component Registry

For dynamo 1.0 each of the following is considered a separate reusable unit and will have its own set of tests, documentation, and backwards compatibility guarantees. Each component will have it's own proposal for its API.

Definition: A part of Dynamo that runs as it's own process, or something that could easily live in a separate repo.

Each component owner will be responsible for a component markdown file similar to the deep dive \- but with spelled out 1.0 feature set, test strategy, metrics, etc.

| Component | Purpose | Languages | Deep Dive | Support Level at 1.0 | Standardization Priority | PIC |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| Planner | Auto Scaling  | Python | [enhancements/deps/planner-component-summary.md at dynam v  o\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/planner-component-summary.md) | Service / Command | P1 | [Hongkuan Zhou US](mailto:hongkuanz@nvidia.com) |
| Router | Intelligent routing  | Python, Rust | [enhancements/deps/router-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/router-component-summary.md) | Service / Command Library Embedded / Standalone | P0 | [Rudy Pei US](mailto:rupei@nvidia.com) |
| Frontend | Open AI / KServe compatible server | Python, Rust, REST, gRPC | [enhancements/deps/frontend-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/frontend-component-summary.md) [enhancements/deps/0000-frontend-cli-formalization.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/0000-frontend-cli-formalization.md) | Service / Command Library Schemas  | P0 | [Graham King US](mailto:grahamk@nvidia.com), [Guan Luo US](mailto:gluo@nvidia.com) |
| Multimodal Backends | Example | Python |  | Experimental | P1 | [Indrajit Maloji Bhosale US](mailto:ibhosale@nvidia.com) |
| Backend vLLM | Python wrapper for vLLM | Python | [enhancements/deps/backend-vllm-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/backend-vllm-component-summary.md) | Service / Command | P0 | [Alec Flowers US](mailto:aflowers@nvidia.com) |
| Backend SGLang | Python wrapper for SGLang | Python | [enhancements/deps/backend-sglang-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/backend-sglang-component-summary.md) | Service / Command | P0 | [Ishan Dhanani US](mailto:idhanani@nvidia.com) |
| Backend TRT-LLM | Python wrapper for TRT-LLM | Python | [enhancements/deps/backend-trtllm-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/backend-trtllm-component-summary.md) | Service / Command | P0 | [Tanmay Verma US](mailto:tanmayv@nvidia.com) |
| Python Bindings | Bindings for dynamo-llm and dynamo-runtime | Python, Rust | [enhancements/deps/python-bindings-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/python-bindings-component-summary.md) | Library | P0 | [Graham King US](mailto:grahamk@nvidia.com), [Neelay Shah US](mailto:neelays@nvidia.com) |
| C Bindings |  | Rust, C | [enhancements/deps/c-bindings-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/c-bindings-component-summary.md) | Internal | P2 | [Anna Tchernych](mailto:atchernych@nvidia.com) |
| Core Libraries |  | Rust | [enhancements/deps/core-libraries-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/core-libraries-component-summary.md) | Internal | P2 | [Graham King US](mailto:grahamk@nvidia.com), [Neelay Shah US](mailto:neelays@nvidia.com) |
| KVBM | KV Block Manager | Rust, Python | [enhancements/deps/kvbm-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/kvbm-component-summary.md) | Experimental | P1 | [Ziqi Fan US](mailto:ziqif@nvidia.com)  |
| DGDR (Profiling) | Profiling and Initializing DGD |  | [enhancements/deps/deployment-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/deployment-component-summary.md) | Service / Command Schemas | P0 | [Hannah Zhang](mailto:hannahz@nvidia.com) |
| Deployment | Operator  | Go, YAML | [enhancements/deps/0000-kubernetes-api-formalization.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/0000-kubernetes-api-formalization.md) [enhancements/deps/deployment-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/deployment-component-summary.md) | Service / Command Schemas | P0 | [Thomas Montfort US](mailto:tmontfort@nvidia.com), [Julien Mancuso US](mailto:jmancuso@nvidia.com) |
| Recipes |  | YAML, JSON, Python | [enhancements/deps/recipes-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/recipes-component-summary.md) | Service / Command Experimental | P0 | SA / DevEx, [Ben Hamm US](mailto:bhamm@nvidia.com) |
| Examples |  | YAML, JSON, Python | [enhancements/deps/examples-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/examples-component-summary.md) | Service / Command Experimental | P1 | SA / DevEx,[Neal Vaidya US](mailto:nealv@nvidia.com) |
| Benchmarks |  | YAML, JSON, Python | [enhancements/deps/benchmarks-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/benchmarks-component-summary.md) | Service / Command Experimental | P0 | SA / DevEx, [Ben Hamm US](mailto:bhamm@nvidia.com) |
| IGW |  |  | [enhancements/deps/igw-integration-component-summary.md at dynamo\_api · ai-dynamo/enhancements](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/igw-integration-component-summary.md) | Service / Command Experimental | P1 | [Anna Tchernych](mailto:atchernych@nvidia.com) |

#

# Testing Requirements

* Goal: good tests that catch bugs before merge by making the test cycles fast, reliable, and easily runnable locally.
* 100% coverage of public APIs
* Continuous use in prod setting / traffic
* Use industry's best practices:
  * Use a common/shared harnesses/launchers
  * Use composition/inheritance patterns over copy-paste.
  * Use a common library layer to generate random paths and to allocate random ports.
  * Long running tests must have a time-bound guard, to prevent tests hanging and wasting developers' time.
  * Use reliable mock components instead of large dependencies like the entire vLLM-Engine, for reproducibility and for speed.
* Hermetic-by-default: Tests must be reproducible and isolated.
  * No shared ports: use dynamic port allocation to avoid collisions and enable parallel runs.
    * Use a central registry to reserve ports between different processes.
  * No common hard-coded paths.
  * Hermetic tests are parallelizable and results repeatable, when given sufficient resources.
  * Hermetic tests removes (or minimizes) external dependencies, especially those that are not reliable (e.g. network calls to some repo).
* Layered test pyramid (required):
  * Rust unit tests: Fast, deterministic; run on every local iteration.
  * Rust integration tests: Cover real edge cases (esp. TCP comm edge cases); deterministic; minimal flake surface.
  * Python e2e tests: Validate real workflows; keep small and high-signal; use consistent launchers/harnesses.
    * Use a central registry to reserve GPU RAM, in order to run multiple GPU tests in parallel
* Coverage is measurable and enforced:
  * Coverage reporting for core Rust libraries (esp. TCP edge cases) with trend charts.
  * Acceptance: Coverage metrics published per-commit (or per-MR) and visible on a dashboard.
* Flake management is required:
  * Track flaky test rate and identify top offenders.
  * Add timeouts for long/flaky tests; classify tests via pytest markers and enforce.
* Test failures must be actionable:
  * Every failure should surface: what failed, why, owner/signal, and suggested next step. It is much more than seeing Exception errors in the logs.

#

# Framework Requirements

* Use only public API's from the upstream framework
* Support two version at any given time (current version and previous version). For example vLLM 0.12.0 and 0.11.0.
  * Reason:
* Upstream frameworks CI should cover core models that Dynamo relies upon
  * KV Events \+ Disagg
  * Upstream teams have such a huge feature matrix
  * We should be testing nightly containers from frameworks
  * **Nightly Canaries with upstream main \- CI**
  * Work [Akbar Nurlybayev CA](mailto:anurlybayev@nvidia.com)  Xin on tests \+ compute for upstream
* Where possible drive consistency between frameworks

# Developer Velocity Requirements

* Goal: Developer Velocity is the measurement of a developer's productivity.
* Initial Priorities
  * **No Flakey Tests, Coverage, Build and pre-merge test time \< 1 hour**
* Measure the developer's end-to-end workflow:
  * Time from (local image build → coding → Rust/Python tests) → PR submitted → review → CI pass → merge.
  * Track: local build/test time, remote build time, CI pre-merge time, rerun rate, code review time, flaky test stats.
* Visibility must create actionable work:
  * Every slowdown must map to: what broke, where, why it matters, and the next fix.
  * Prioritize "clear actionable signals" over raw logs.
  * Uses easy to understand dashboards, messaging, tools.
  * Release bugs with P0 P1 … categorization, statistics ([Dan Gil US](mailto:dagil@nvidia.com)), what to focus on to drive up release quality and driving down fatal bugs.
* Fast feedback loops are mandatory:
  * Optimize build→test→profile→iterate across local and CI.
  * Encourage iterative use by making "do the right thing" the fastest thing.
* SOTA practices and superb tooling:
  * Standardize on shared harnesses/launchers/common frameworks that bake correctness in.
  * Invest in workflow accelerators (Async Agent in the Cloud [Alec Flowers US](mailto:aflowers@nvidia.com)) that shortens the loop.
* Ownership model for velocity:
  * Each KPI (coverage, flake rate, build time, rerun rate, review latency) has a responsible owner and an improvement plan.
  * Tie improvements to release quality outcomes (bug escapes, regressions, hotfix frequency).

# CI/CD Requirements

* CI
  * \[P0\] Buildkit runners to speed up builds and create multi arch images \- Ran
  * \[P0\] Docker Templating \- Dillon
  * \[P0\] Python Dependency management and pinning
    * use virtual env in sglang containers \- Anant
    * Python dev/build flow should start with uv \- check on trtllm issue \- Anant
    * Separation of test deps from runtime deps \- Anant
    * Pinning and lock files for python packages
  * \[P0\] Nightly should be stable \- ops-support
  * Testing Coverage/Stability
    * \[P0\] Address Flakiness \- Dmitry/Tushar
    * \[P0\] Flow to add new tests \- Dmitry/Tushar
      * Keep pre merge testing within time limits
      * When should test be added to pre merge
    * \[P0\] Examples and recipes testing \- Dmitry/Tushar
      * Provides partial K8 coverage
    * \[P1\] Multi GPU testing in github \- Saravana
  * \[P1\] Workflow consolidations \- cleanup github CI and standardize required checks \- Anant/Dillon
  * \[P1\] Build and Test node separation \- Dillon
  * \[P1\] Externa/Internal Contributors workflow \- Harrison
    * PR review assignment to users
    * Easier CI kickoff for external users
* Monitoring/Alerting \- Nate/ Meenakshi
  * \[P0\] CI pipeline health Monitoring and alerting
  * \[P0\] Time limit checks for CI
  * \[P0\] Infra monitoring and alerting
* Time limits \- Complete ops team
  * \[P1\] less than 30 mins for PR checks
    * \[P0\] 45 mins
* Release Automation \- Harrison/Pavi
  * \[P0\] Automated RC staging
  * \[P0\] Container source tar
  * \[P1\] Proactive OSRB checks on incoming changes to main
* Surveys on what makes developers effective, to prioritize and to optimize workflows
* **Wont have in 1.0 but want**
  * Arm GPU testing on github (covered in gitlab)
* Refs:
  * [CI Pipeline Health](https://grafana.nvidia.com/d/ef15rkc8yp6o0c/pipeline-health-overview?orgId=283)
  * [Initiatives here](https://docs.google.com/document/d/1lEIGs_dM2zb9VMYdvVgEZUXAnVTCr44Hv4pXczuvHyA/edit?tab=t.0#heading=h.ql94s4au4euh)
  * [Dynamo Test Automation Initiative](https://docs.google.com/document/d/16vwxjVaM0oy4t6key7lyfmrC7jMtTNkHFuc6ez6FZjY/edit?tab=t.0#heading=h.9reyrfkazrip)
  * [Ops Roadmap 2026](https://docs.google.com/document/d/1M3Cv7tcGNLOnGsAaDL6L6yr3hWxoF-uhwdMp2MqT1B4/edit?tab=t.0#heading=h.bjva2htwn4z6)

# Packaging and Repo Structure

Need some updates and finalize soon [Neelay Shah US](mailto:neelays@nvidia.com)

* What artifacts will be released and configs
  * What containers, wheels, creates will be release
  * Do we want cuda13 to be default?
* [Agent.md](http://Agent.md) file in repo
* How to structure the dynamo repository
