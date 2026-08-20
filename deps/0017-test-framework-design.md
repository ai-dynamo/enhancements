# Component-Oriented Test Framework for Deployment-Agnostic Testing

**Status**: Draft

**Authors**: Dmitry Tokarev

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: (this PR)

**Implementation PR / Tracking Issue**: [ai-dynamo/dynamo#12690](https://github.com/ai-dynamo/dynamo/pull/12690)

# Summary

Introduce a component-oriented object model for Dynamo's test suite: one class
per Dynamo component (frontend, worker, router, planner, operator, KVBM, …),
each wrapping that component's wire interface **and its lifecycle**, composed by
a single `Dynamo` façade that is constructed from configuration supplied at run
time and injected into tests as a fixture.

The goal is one functional suite that runs **unchanged** across every way Dynamo
is deployed — local processes, a single all-in-one container, components in
separate containers, Docker Compose, and Kubernetes — so that a new model, a new
accelerator, or a new recipe becomes a *configuration* of the existing suite
rather than a new suite. A test states intent —
`dynamo.frontend.query("What is the capital of France?")` or
`dynamo.frontend.restart(router_mode="kv")` — and never constructs an HTTP
request or a `python -m dynamo.frontend` command line. Which tests apply to a
given target is decided automatically by evaluating declared dependencies
against the hardware, model and deployment configuration in use.

DEP 0008 defines *what kinds* of tests exist and DEP 0009 defines *when and
where CI runs them*. This proposal addresses the layer neither covers: how an
individual test is **written** against a system with this many deployment
shapes, model capabilities and hardware targets. See
[Related Proposals](#related-proposals).

# Motivation

Dynamo is deployed in many shapes and is currently tested once per shape. We
want to point one suite at any of them:

| Environment | Shape | Today |
|---|---|---|
| Local dev | processes on localhost, no container | ad hoc scripts |
| Dev, single container | Dynamo and tests in the same container | the all-in-one suite |
| CI, tests out-of-container | pytest in its own process/container, Dynamo in another | partially working |
| CI, component-per-container | frontend / worker / sidecar each in their own container | not supported — the target shape for the sidecar work |
| Docker Compose | multi-container topology | not supported |
| Kubernetes | frontend exposed by service hostname | the deploy suite |

And to re-run that one suite across four independent axes:

- **Platform** — local, all-in-one container, container-per-component, Compose, Kubernetes
- **Model** — differing capabilities and resource envelopes (FP16, NVFP4, MoE,
  multimodal, context length, memory requirement)
- **Hardware** — A100, H200, GB200, …; single-node and multi-node
- **Recipe / example** — each shipped DynamoGraphDeployment is a configuration
  worth validating

The payoff is that validating a new recipe, model or accelerator means *running*
the suite, not *writing* one.

**What it costs today.** Coverage is authored per cell of that matrix, so it
grows multiplicatively and most cells stay empty. The same functional assertion
— the model answers, tools are called, streaming works — exists twice, once for
the in-container suite and once for the deploy suite, and neither version can
run in the other's environment. Pointing tests at a new deployment is manual.

Recent work ([dynamo#12690](https://github.com/ai-dynamo/dynamo/pull/12690))
took the first step: payloads carry their own target address and are executed by
a shared runner, deployment-coupled tests are identified by an explicit
predicate, and the conventions are documented in `tests/README.md` and
`.ai/pytest-guidelines.md`. What is still missing is an object model — there is
no `Dynamo` to hand a test, so each test re-derives how to reach a component,
how long to wait, how to reconfigure it, and what the deployment can be asked.

## Three classes of test, and what each needs

**1. Functional (deployment-agnostic).** The majority. Sends inference requests
and asserts on responses. Should run in *every* cell of the matrix, unchanged.
This is the portability target.

**2. Configuration and topology.** Verifies Dynamo behaviour under varying flags
and component layouts. Today these mostly run inside the same container as
Dynamo and restart processes in place with different flags — quick, dirty, and
acceptable for now, but they are **not marked as such** and cannot run anywhere
else. They need (a) explicit classification, and (b) a lifecycle abstraction so
the identical test can restart a local process, `docker exec` a restart inside a
running container, delete and recreate a container with new flags, or patch a
DGD — chosen by the platform, not written into the test.

**3. Deployment-artifact.** Validates the DGD YAMLs and manifests themselves.
Kubernetes-specific by nature, and should be labelled as such rather than
treated as a portability failure.

## A secondary thread: test shallowness, and shallow health checks

Portability is worth little if the assertions underneath are weak — and the
platform's own health signals are weaker than they look. Three patterns, each
cheap to fall into and expensive to detect:

- **Accept-but-ignore** — a request is accepted with HTTP 200 and silently not
  honoured (`tool_choice: "required"` with no constrained-decoding backend
  configured). Asserting on status codes reports the capability as working.
- **Presence instead of substance** — asserting that a field exists rather than
  that it is language; a worker emitting one repeated token passes.
- **Readiness is not correctness** — a component can report `Ready`, with zero
  restarts and nothing logged, while serving unusable output.

Multi-replica deployments compound all three: prefix-affinity routing sends a
fixed prompt to the same worker every time, so one healthy sample is mistaken
for a healthy fleet.

Separately, fault-tolerance and chaos suites deliberately manipulate
infrastructure. Without an explicit axis for that power, infrastructure access
leaks back into ordinary tests and portability erodes silently.

## Goals

* One functional suite that runs unchanged across local, container,
  container-per-component, Compose and Kubernetes deployments.
* Tests express intent against components, never transport or process detail.
* Test dependencies — model capability, Dynamo configuration, topology,
  hardware and interconnect, infrastructure powers — declared explicitly and
  evaluated automatically.
* Re-runnable across models, hardware targets and shipped recipes by
  configuration alone.
* Provide the authoring mechanism that DEP 0008 assumes when it states tests
  should be "easy to write and run both locally and for CI" across "multiple
  deployment targets".

### Non Goals

* To enumerate test cases, or to replace the test taxonomy and lifecycle
  defined in DEP 0008.
* To redefine CI pipeline structure, triggering or gating — DEP 0009 owns that.
  This proposal must not break marker-based selection or the VRAM-aware GPU
  scheduler that CI depends on.
* To replace existing deployment tooling; the Kubernetes lifecycle provider
  wraps the existing `tests/deploy/` utilities.
* To specify a coverage metric.

## Requirements

### REQ 1 Portability across deployment platforms

A functional test MUST run unmodified on: local processes, an all-in-one
container, tests-out-of-container against a sibling container,
container-per-component, Docker Compose, and Kubernetes.

### REQ 2 No low-level calls in tests

Tests MUST NOT construct HTTP requests, gRPC stubs, or process command lines
directly. All component interaction goes through component classes.

### REQ 3 Component lifecycle control

Components MUST expose `start(**flags)`, `stop()`, `kill()` and `restart()`,
implemented per platform, so configuration and topology tests are portable
rather than bound to in-container process restarts.

### REQ 4 Explicit, typed test dependencies

Test dependencies MUST be declared explicitly and typed, covering model
capability, Dynamo configuration, topology, hardware/interconnect, and
infrastructure powers. Capability MUST be derived from configuration; a test
MUST NOT infer support from an HTTP status code.

### REQ 5 Automatic selection, manual narrowing preserved

Applicable tests MUST be selected automatically by evaluating declared
dependencies against the target hardware, model and deployment configuration;
hand-maintained selection MUST NOT be required. Marker-based selection MUST
remain available for narrowing by functional area (e.g. `pytest -m multimodal`),
and declared dependencies MUST project to equivalent markers so that existing
CI selection and the VRAM-aware GPU scheduler continue to work.

### REQ 6 Trustworthy dependency evaluation

Evaluation MUST be three-valued: `SATISFIED`, `UNSATISFIED`, `UNKNOWN`.
`UNKNOWN` MUST NOT be treated as unsatisfied, MUST be reported, and MUST be
gateable in CI. Requirements (unmet ⇒ skip) MUST be distinguishable from
preconditions (unmet ⇒ error). An unsatisfied requirement MUST produce an
attributed skip or xfail naming the missing capability and its fact source —
never a silent pass.

### REQ 7 Assertion depth

Fleet-level assertions MUST defeat prefix affinity (unique prefix per request)
and MUST attribute each response to the worker that served it. Readiness
(`wait_until_serving`) MUST be separate from correctness assertions.

### REQ 8 Test class visibility

The three test classes — functional, configuration/topology, and
deployment-artifact — MUST be explicitly distinguishable, so that non-portable
tests are visible rather than assumed portable.

### REQ 9 Isolation of infrastructure access

A test that only sends inference requests MUST be unable to reach pod logs,
worker system ports, replica counts or the Kubernetes API; that access MUST
require explicitly constructing `Dynamo` with a deployment handle.
Fault-tolerance and chaos tests MUST be expressible and MUST sit explicitly on
that axis.

### REQ 10 Matrix re-runnability and reuse

The suite MUST be re-runnable across models, hardware targets and shipped
recipes/examples by configuration alone, with no test edits. Existing
`tests/deploy/` utilities MUST be reused by the Kubernetes lifecycle provider
rather than reimplemented. Absent components MUST fail fast and by name, not by
timeout. The framework MUST run without a GPU where the mocker backend
suffices.

# Proposal

Model each Dynamo component as a class that owns two things: the component's
**wire interface** (how you talk to it) and its **lifecycle** (how you start,
stop and reconfigure it). Compose those into a `Dynamo` façade built from
run-time configuration. Give tests that façade and nothing else.

Reachability and controllability are separate axes, and keeping them separate is
what makes the same test portable:

```python
Dynamo(transport=Http("http://localhost:8000"))                    # attached; query only
Dynamo(transport=Http("http://dyn-serve:8000"))                    # sibling container
Dynamo(transport=Http(...), deployment=Local(...))                 # + process control
Dynamo(transport=Http(...), deployment=Docker(container="dyn"))    # + exec / recreate
Dynamo(transport=Http(...), deployment=Compose(project=...))       # + service control
Dynamo(transport=Http(...), deployment=K8s(ns=..., name=...))      # + DGD patch, pod delete
```

`deployment` is opt-in: a test that only queries cannot reach infrastructure,
because the handle is not present on the object it was given (REQ 9).

Applicability is declared, not selected by hand:

```python
@requires(
    Model.modality(IMAGE), Model.quant("nvfp4"), Model.arch(MOE),
    Config.guided_decoding,
    Topology.workers(">=3"),
    Interconnect.nvlink_domain(min_nodes=2),
    Powers.CAN_KILL_POD,
)
def test_disagg_survives_leader_loss(dynamo): ...

def test_tool_choice_required(dynamo):
    dynamo.require(Capability.CONSTRAINED_DECODING)   # skip/xfail, with a reason
```

# Implementation Details

## Object model

```python
class Component:
    """Wraps one component's wire interface and lifecycle.
       Owns readiness, retry and protocol detail."""

class Frontend(Component):
    def query(self, prompt, **kw) -> Response: ...
    def stream(self, prompt, **kw) -> Iterator[Chunk]: ...
    def models(self) -> list[str]: ...
    def start(self, **flags) -> None: ...
    def stop(self) -> None: ...
    def kill(self) -> None: ...
    def restart(self, **flags) -> None: ...
    def wait_until_serving(self, timeout) -> None: ...

class Dynamo:
    frontend: Frontend | None
    worker:   Worker | None
    router:   Router | None
    planner:  Planner | None
    operator: Operator | None
    kvbm:     Kvbm | None
    fleet:    Fleet
    capabilities: frozenset[Capability]
    deployment: Deployment | None      # present only when explicitly requested
```

## Component lifecycle

`start` / `stop` / `kill` / `restart` are declared on the component and
implemented by the platform's lifecycle provider:

| Platform | start / stop / restart | kill |
|---|---|---|
| Local | spawn / terminate subprocess | `SIGKILL` |
| Same container | in-container process control | `SIGKILL` |
| Container-per-component | `docker exec` restart, or `docker rm` + `docker run` with new flags | `docker kill` |
| Compose | service up / stop / recreate | `compose kill` |
| Kubernetes | patch DGD, roll pods | delete pod without grace |

This is what makes class 2 portable: the test says
`dynamo.frontend.restart(router_mode="kv")`, and whether that is a subprocess
restart or a DGD patch is the provider's problem.

## Dependency kinds and where facts come from

| Kind | Source of truth | Resolved | Unmet ⇒ |
|---|---|---|---|
| Model capability — multimodal, NVFP4, MoE, ctx length, memory | model config ∧ engine args | session | skip |
| Dynamo config — router mode, guided decoding, disagg, spec-dec | engine args / DGD / CLI | session | skip or xfail |
| Topology — platform, ≥N workers, nodeCount, TP/EP | deployment handle or fleet discovery | session | skip |
| Hardware / interconnect — GPU model, VRAM, compute capability, NVLink domain, RDMA | node labels / `nvidia-smi` / ComputeDomain | collection | deselect |
| Powers — restart component, kill pod, partition network | the deployment axis *type* | construction | skip |

Notes that matter in practice:

* **Model capability is an intersection**, not a lookup: a multimodal model
  served with `--language-model-only` is not multimodal *in that deployment*.
* **Interconnect is a domain, not a flag**: an MNNVL test needs ≥2 nodes in the
  same clique, which is readable from ComputeDomain status, not from a boolean.
* **Discovery yields bounds**: worker count inferred by probing distinct worker
  IDs is a *lower* bound — sound for `">=3"`, unsound for `"==3"`. Predicates
  that a provider cannot honestly answer MUST evaluate to `UNKNOWN` (REQ 6).

**Selection is automatic; markers express intent.** Requirements answer *"can
this run here?"*; markers answer *"what do I want to run?"* Because CI selects
with `-m` expressions and the GPU scheduler reads `profiled_vram_gib(N)`,
`@requires(...)` emits equivalent markers at collection (REQ 5).

## The fleet is first-class

```python
result = dynamo.fleet.probe(prompt, n=12)   # unique prefix per request
assert result.workers_seen >= 2
assert not result.degenerate
assert result.all_answered(expect="paris")
```

On a single-replica deployment this degrades to a repeated single query.

## Component interfaces wrapped by the classes

| Component | Interface | Surface |
|---|---|---|
| Frontend | HTTP (axum), default `:8000` | `/v1/chat/completions`, `/completions`, `/models`, `/embeddings`, `/responses`, `/messages` (Anthropic), `/classify`, `/pooling`, `/batches`, `/files`, `/images/generations`, `/videos`, `/audio/speech`, `/realtime` (WS); `/health`, `/live`, `/ready`, `/metrics` |
| Frontend (opt-in) | KServe gRPC | `--kserve-grpc-server`, `--grpc-metrics-port` |
| Any runtime component | HTTP system server on `DYN_SYSTEM_PORT` | `/health`, `/live`, `/metrics`, `/metadata`, `/v1/loras`, `/custom/{health,live}` — the per-worker plane |
| Runtime substrate | discovery | etcd \| Kubernetes (CRD + EndpointSlices) \| file/kv-store \| mock |
| Runtime substrate | request plane | TCP \| NATS |
| Runtime substrate | event plane | ZMQ \| NATS |
| Router | in-process in the frontend (`--router-mode kv`) | observable via `nvext.extra_fields=["worker_id"]` |
| Router (gateway) | Envoy `ext_proc` gRPC | inference-gateway ext-proc |
| Planner | plugin transport: gRPC or in-process | no stable HTTP port |
| Operator | Kubernetes API | CRDs `DynamoGraphDeployment`, `DynamoComponentDeployment`, `DynamoWorkerMetadata` |
| Engine sidecars | gRPC | per-backend protos — the component-per-container shape |
| Backends | runtime endpoints (e.g. `generate`) | via discovery, plus their system port |
| KVBM | ZMQ + metrics | leader ZMQ pub/ack, metrics port |
| Telemetry | Prometheus | forward-pass metrics, NIXL telemetry |
| Mocker | backend substitute | component tests with no GPU |

## Deferred to Implementation

* The concrete `Capability` enumeration and per-backend fact extractors.
* Whether the standalone router component warrants its own class, or is only
  ever observable through the frontend.
* Compose topology definitions for the component-per-container shape.
* Whether `UNKNOWN` should fail CI by default from day one, or warn for one
  release before becoming an error.

# Implementation Phases

## Phase 1 Transport, Frontend, Fleet

**Work Item(s):** TBD

Covers most existing functional tests across local, same-container,
sibling-container and Kubernetes for class 1. Builds on the endpoint fixture and
shared payload runner already landed.

## Phase 2 Dependency model and automatic selection

**Work Item(s):** TBD

Typed requirements, three-valued evaluation, precondition/requirement split, and
marker projection so CI selection and the VRAM scheduler keep working.

## Phase 3 Lifecycle providers

**Work Item(s):** TBD

Local and same-container first — matching what class 2 tests already do — then
container-per-component and Compose, then Kubernetes.

## Phase 4 Deployment axis and test-class migration

**Work Item(s):** TBD

Migrate topology-coupled and DGD-artifact tests onto the deployment axis and
label them.

## Phase 5 Remaining components

**Work Item(s):** TBD

Worker system port, planner, operator, KVBM — added only when a test needs one.

# Related Proposals

* **[DEP 0008 — Test Strategy](0008-testing-strategy.md)** defines the shared
  vocabulary, the test taxonomy (lint, unit, integration, E2E, benchmark,
  stress), directory structure, coverage expectations, and how tests map onto
  the development and release life-cycle. It states as a goal that "tests should
  be easy to write and run both locally and for CI" and that the strategy must
  fit "multiple programming languages and deployment targets" — but it does not
  specify the mechanism by which a single test achieves that. This proposal
  supplies that mechanism and does not change the taxonomy.

* **[DEP 0009 — Testing in CI Strategy](0009-testing-in-ci-strategy.md)** defines
  CI workflows and which pipeline runs which tests, test segmentation via pytest
  markers and cargo groupings, coverage and adequacy metrics, and quality gates.
  It governs *when and where* tests execute. This proposal governs *how a test is
  written*, and is deliberately constrained by 0009: marker-based selection and
  the VRAM-aware GPU scheduler must keep working, so declared dependencies
  project down to markers (REQ 5) rather than replacing them. Automatic
  selection complements 0009's manual segmentation — dependencies decide
  applicability, markers remain for a human narrowing scope.

Neither proposal addresses test framework *design*: the authoring-time
abstraction needed given the number of Dynamo deployment options, model
capabilities and requirements, and hardware targets. That gap is what this
proposal fills.

# Alternate Solutions

## Alt 1 Keep the current payload and marker approach

Already delivers deployment agnosticism for the payload/verification path, and
is strictly less work. Rejected as insufficient: there is no home for readiness
policy, capability negotiation, lifecycle control or fleet semantics, so every
test re-implements them — which is how the shallowness patterns above arose —
and it does nothing for the platform matrix.

## Alt 2 Fixture-per-endpoint, no component classes

A `frontend_url` fixture plus free functions. Simpler, but behaviour that
belongs to a component (waiting, retry, restart, protocol quirks) ends up in
ownerless helper modules, and there is no natural place for `capabilities`,
`fleet` or lifecycle.

## Alt 3 A single monolithic client class

One `DynamoClient` exposing everything. Cannot express "this deployment has no
planner", and invites methods that quietly require infrastructure access,
defeating REQ 9.

## Alt 4 Adopt an off-the-shelf harness

Container-lifecycle libraries solve container lifecycle, not Dynamo's
components, capabilities or multi-replica semantics. Suitable *underneath* the
Docker and Compose lifecycle providers rather than as a replacement.

## Alt 5 Generate per-platform suites from a shared core

Code generation preserves the per-cell structure and its multiplicative cost,
and generated tests are harder to debug than a runtime-configured object.

# Background

## References

* [DEP 0008 — Test Strategy](0008-testing-strategy.md)
* [DEP 0009 — Testing in CI Strategy](0009-testing-in-ci-strategy.md)
* [ai-dynamo/dynamo#12690](https://github.com/ai-dynamo/dynamo/pull/12690) —
  makes pytests agnostic of how Dynamo is deployed; introduces the shared
  payload runner and the deployment-coupling predicate this proposal builds on.
* `tests/README.md` and `.ai/pytest-guidelines.md` in `ai-dynamo/dynamo` — the
  authoring conventions added alongside that PR.

## Terminology & Definitions

| Term | Definition |
|---|---|
| Component | A Dynamo process or service with its own wire interface — frontend, worker, router, planner, operator, KVBM. |
| Transport | How a component is reached (URL, hostname, socket). Independent of how it was deployed. |
| Lifecycle provider | Platform-specific implementation of start/stop/kill/restart. |
| Deployment handle | Opt-in object granting infrastructure control (process, container, Compose project, Kubernetes objects). |
| Capability | A property of the deployed system derived from configuration — e.g. constrained decoding enabled. |
| Requirement | A declared dependency; unmet ⇒ the test is skipped as inapplicable. |
| Precondition | An environmental invariant; unmet ⇒ error, because skipping would hide a defect. |
| Fleet | The set of workers serving a model behind one frontend. |
| Prefix affinity | KV-aware routing sending identical prompt prefixes to the same worker. |

## Acronyms & Abbreviations

| Acronym | Meaning |
|---|---|
| DGD | DynamoGraphDeployment (Kubernetes custom resource) |
| TP / EP | Tensor Parallel / Expert Parallel size |
| MoE | Mixture of Experts |
| MNNVL | Multi-Node NVLink |
| KVBM | KV Block Manager |
| FT | Fault Tolerance |
