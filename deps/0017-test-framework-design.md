# Deployment-Agnostic Test Framework: Phases, Reach, Ownership and Evidence

**Status**: Draft

**Authors**: Dmitry Tokarev

**Category**: Architecture

**Replaces**: N/A — this is revision 2 of DEP 0017, revised **in place**. DEP 0000 §*Significant
Changes After Review* requires a *new* proposal only for significant changes made **after** review;
this proposal's `Review Date` is still TBD and no review has occurred, so revising in place is the
prescribed path rather than an exception to it. Revision 1 is preserved in the history of this file
and in [ai-dynamo/enhancements#100](https://github.com/ai-dynamo/enhancements/pull/100); see
[Revision History](#revision-history).

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: [ai-dynamo/enhancements#100](https://github.com/ai-dynamo/enhancements/pull/100)

**Implementation PR / Tracking Issue**: [ai-dynamo/dynamo#12690](https://github.com/ai-dynamo/dynamo/pull/12690)

# Summary

Dynamo's test corpus is authored per cell of a four-axis matrix (platform × model × hardware ×
recipe), so coverage grows multiplicatively and most cells stay empty. This proposal defines the
authoring-time abstraction that collapses the matrix into configuration, and it is deliberately
larger than revision 1 because revision 1 modelled roughly half the problem.

Six things change:

1. **Four phases, three receivers.** PLAN is pure and has no receiver. A test then ACTs on a live
   system, the harness COLLECTs evidence before teardown, and verdicts are computed in CHECK by
   synchronous pure functions over an on-disk bundle. Separating them is what makes a verdict
   replayable, offline, and independent of the platform that produced it.
2. **Two reach substrates.** `test process → SUT` and `deployed generator → SUT` have different
   addressing, different placement and different failure modes. Revision 1 models only the first;
   the entire load-generation path lives in the second.
3. **Ownership, not only attachment.** The framework must be able to *own* a deployment — render a
   per-arm spec, scrub the namespace, provision log storage, prefetch models, refuse to start dirty
   — not merely be handed a URL.
4. **Six test classes, not three**, plus `infra_helper` as an explicitly non-test category. Each
   missing class has a measured cost: a suite that reads as covered and is not.
5. **A verb-first façade with the role as an argument.** The provider layer underneath is already
   role-parameterised, and the change fixes a real scope defect. It **must not land before** the
   CHECK-phase receiver, or it exports a landmine (Phase 5's gate carries that ordering).
6. **Markers stay the wire format.** Declared requirements are applied as real `pytest.mark` objects
   at decoration time and read back as a *view*; the harness registers **no** collection hook that
   mutates markers, and the parity gate that proves CI selection is unchanged must not use
   `--collect-only`.

DEP 0008 defines *what kinds* of tests exist; DEP 0009 defines *when and where* CI runs them. This
proposal addresses the layer neither covers: how an individual test is **written** against a system
with this many deployment shapes, model capabilities and hardware targets, and what artifact it
leaves behind.

# Motivation

## Measurement provenance

Every quantitative claim below carries a tag.

| Tag | Meaning |
|---|---|
| **[m]** | **Measured for this revision** on `ai-dynamo/dynamo` at commit `0765d30ad8` unless another ref is named. The command or `file:line` is given, and any reviewer with the repository can re-run it. Note `origin/main` has moved on since; use the literal SHA. |
| **[p]** | Measured against a **peer artifact**: the prototype branch `dtokarev/tests-v2-harness` at `2fd6eb2586` (public, re-runnable), or the internal peer harness `dyntest` at `e77b0d3` (**not publicly readable** — a public reviewer must take those on trust; they are marked individually). |
| **[a]** | Carried from the author's design analysis and **not** re-verified for this revision. Indicative only. Where an `[a]` claim motivates a requirement, the requirement's normative text also carries a structural justification that stands without it, and that justification is stated. |
| **[e]** | The author's **estimate**, not a measurement. Used only for effort and calendar. |

Revision 1 measured `4c3e61f107`. Numbers retracted or corrected since are listed openly in
[§M2](#m2-six-corrections-carried-in-the-open), not applied silently.

## The gaps this proposal must close

Thirteen gaps were identified in the design analysis that precedes this proposal. They are named
here so that the coverage table at the end of *Requirements* has an antecedent.

| Gap | What is missing today |
|---|---|
| **G1** | **Load generation.** No first-class sustained, shaped, measured workload with a lifetime; load is either a loop in a test or a shelled-out CLI whose flags leak into the test. |
| **G2** | **A timed event timeline.** No canonical record of "what the harness did and when", so windowed assertions are written by string-matching event names. |
| **G3** | **An evidence bundle.** Artifacts are scattered across ad-hoc directories with no index, no schema and no version, so a run cannot be re-scored after the cluster is released. |
| **G4** | **Checks as values, and a third verdict.** Assertions are inline `assert`s; there is no way to say "measured on purpose, does not gate". |
| **G5** | **Reports that never gate.** Reporting code raises into the verdict path, so a broken plot fails a good run. |
| **G6** | **Run validity as a separate gate.** "The producer emitted nothing" and "the system failed its SLO" are the same red today. |
| **G7** | **A desired-state document.** There is no noun for *what gets patched* per arm, so mutation happens against shared, mutable spec objects. |
| **G8** | **An environment profile with a bleed check.** Per-site facts and per-test facts are mixed in the same dictionaries. |
| **G9** | **Replica and process selection.** "Kill a worker" cannot express *which* worker, and does not record what it actually picked. |
| **G10** | **Fault evidence as a contract.** A fault that never landed is indistinguishable from a fault that landed and had no effect. |
| **G11** | **A scenario document and a generated registry.** The imperative surface and the declarative surface drift because they are written twice. |
| **G12** | **Suites with negative controls.** `expect: fail` arms invert any red, including harness errors. |
| **G13** | **Long-run operational hazards.** Abandoned-run cleanup, signal escalation that does not depend on process-group delivery, and interpreter exit with hung non-daemon threads. |

## M1 The localhost coupling is real, and it has grown

The central empirical claim of revision 1 holds and is worse than reported **[m]**:

```bash
REF=0765d30ad8
git grep -l --full-name -e '' $REF -- 'tests/*.py' | wc -l                                   # 217
git grep -lE 'https?://' $REF -- 'tests/*.py' | wc -l                                        # 112
git grep -lE 'https?://(localhost|127\.0\.0\.1|0\.0\.0\.0)' $REF -- 'tests/*.py' | wc -l     # 78
```

| | revision 1 (`4c3e61f107`) | now (`0765d30ad8`) **[m]** |
|---|---:|---:|
| tracked Python files under `tests/` | 219 | **217** |
| build an HTTP URL at all | 109 | **112** |
| …of those, assume `localhost`/`127.0.0.1`/`0.0.0.0` | 75 (68%) | **78 (69%)** |

| form | revision 1 | now **[m]** |
|---|---|---|
| host **and** port literal | 19 occ / 12 files | **19 occ / 12 files** (unchanged) |
| host literal, port interpolated — `f"http://localhost:{port}"` | 227 occ / 70 files | **245 occ / 73 files** |
| host interpolated — `f"http://{host}:{port}"` | 14 occ / 6 files | **16 occ / 7 files** |

The middle row grew by 18 occurrences across 3 files between the two commits. This is not legacy
debt converging to zero: **`f"http://localhost:{port}"` is the suite's current default authoring
pattern**, and it is still being written. The fully-literal row is unchanged, consistent with
revision 1's note that roughly half of those 19 are legitimate (etcd endpoints the test itself
launches, a deliberately-dead OTLP collector, `e.g.` docstrings).

Because phases 1–4 change no test file, this pattern would keep being written for a year on the
burn-down schedule below. Phase 1 therefore ships a one-grep pre-commit rule that forbids **new**
loopback URL literals under `tests/`; it needs none of the harness and stops the bleeding on day one.

## M2 Six corrections carried in the open

Corrections are numbered **R1–R6** to keep them distinct from the test classes (**TC1–TC6**, §M5) and
from the implementation phases.

| # | Revision 1 said | Correct reading | Evidence |
|---|---|---|---|
| **R1** | A test is one phase against one `Dynamo` object | Four phases; verdicts run **after** teardown, over files | Peer harness `scenario.py:990` sets `ctx.deployment = None` before reports and checks run; **zero `async def`** in its checks package **[p]** |
| **R2** | Reachability is one axis (`transport=`) | Two substrates; the load path never speaks HTTP from the test process | `DeploymentSpec.get_in_cluster_frontend_url()` derives the address *from the deployment*; the generator has its own placement **[a]** |
| **R3** | Something else deployed Dynamo; the test attaches | The framework must be able to own the deployment | The peer harness's `ManagedDeployment.__aenter__` is a multi-step owned bring-up, and there is no attach mode anywhere in either corpus **[p, a]** |
| **R4** | Three test classes | Six, plus a non-test category | §M5 |
| **R5** | REQ 10: the k8s provider must reuse `tests/deploy/` utilities | **Reversed by layer**: Kubernetes *runtime* capabilities from the peer harness, *manifest model* from `tests/deploy` | [§R5 in detail](#r5-in-detail--the-two-kubernetes-implementations-diverge-by-layer-not-by-quality) |
| **R6** | *(design analysis)* "105 of 137 `profiled_vram_gib` applications are table-form" | **71 of 106 are non-decorator (67%)**, and the dominant non-decorator form **is** the parametrize-marks table | [§R6 in detail](#r6-in-detail--the-marker-forms-corrected-twice) |

R6 is stated here rather than quietly fixed because the conclusion drawn from the wrong number is the
one this proposal depends on, and because the correction itself was wrong once before — a reviewer is
entitled to check the arithmetic and to see the instrument that produced it.

### R1 in detail — the phase problem is already a live defect

The peer harness's verdict layer is synchronous and reads a directory. That is not an implementation
accident; it is the property that makes a long GPU run re-scorable for free after the cluster is
released. The cost of having only one receiver is measurable *inside* that harness **[p — internal,
not publicly readable]**:

| Check | File:line | Call |
|---|---|---|
| `RankProcessCount` | `checks/logs.py:280` | `ctx.deployment.get_pods(self.services)` |
| `CliffContained` | `checks/metrics.py:1403` | `ctx.deployment.get_pods([self.pinned_service])` |
| `PinningContained` | `checks/metrics.py:1484` | `ctx.deployment.get_pods([self.pinned_service])` |

All three are **reachable** checks that run after `scenario.py:990` and would raise
`AttributeError: 'NoneType'`. They have never been noticed because no scenario references them. One
sibling site (`checks/metrics.py:680`) guards with `if ctx.deployment is not None`, which is the same
defect caught by hand. (There is additionally one unguarded private helper with zero callers; it is
excluded from the count of three, which is a count of reachable checks.)

A single receiver — `dynamo.expect_no_errors(...)` on the same object that carries `dynamo.kill(...)`
— reads as available at every point in a test and is valid only after teardown. It invites this
defect at corpus scale.

### R5 in detail — the two Kubernetes implementations diverge by layer, not by quality

`tests/deploy/dgd_utils.py` (**1790 LOC**; `DeploymentSpec` **29** methods, `ServiceSpec` **28**,
`ManagedDeployment` **28** **[m]**) and the peer harness's `deployment/managed_deployment.py`
(**4260 LOC**, `ManagedDeployment` **62** methods **[p — internal]**) are forks of one ancestor.

**The peer harness owns the operations axis** — 62 versus 28 methods, plus namespace scrub with
Grove/DCD cascade, a log-collection PVC with an in-pod tee wrapper, RWX verification, model prefetch,
run provenance, signal handling with emergency delete, per-pod metrics capture, and a `kubectl`-based
pod transport **[p — internal]**.

**`tests/deploy` owns the manifest axis.** Measured on the loaders themselves:

| | peer harness `DeploymentSpec` **[p]** | `tests/deploy` `DeploymentSpec` **[m]** |
|---|---|---|
| document handling | `yaml.safe_load` — single document only (`managed_deployment.py:788`) | `_detect_schema()` at `dgd_utils.py:496`; multi-document `safe_load_all` landed on the implementation branch (`24b360f556`, PR #13979) |
| schema | `spec["services"]` indexed unconditionally (`:871`, `:904`, `:922`, `:1105`) | `apiVersion`-driven; reads both `spec.components` (v1beta1) and `spec.services` (v1alpha1) |

Measured reach over the in-tree manifest corpus of §M4, by loader shape **[m]**:

| Loader shape | Manifests it can construct |
|---|---:|
| `yaml.safe_load` + unconditional `spec["services"]` (peer harness) | **67 of 295** |
| `yaml.safe_load` + `_detect_schema` (`tests/deploy` at `0765d30ad8`) | **162 of 295** |
| `yaml.safe_load_all` + `_detect_schema` (`tests/deploy` on PR #13979) | **278 of 295** |

So the honest requirement is **not** "reuse `tests/deploy/`". It is: *the Kubernetes lifecycle
provider MUST consume the shared manifest model rather than reimplement it, and MUST adopt the mature
runtime capabilities rather than reimplement those either.* Two directions, named by layer.

### R6 in detail — the marker forms, corrected twice

The raw grep is **111 [m]**, but 5 of those hits are not marker applications: `tests/conftest.py:147`
(the `--max-vram-gib` help string), `tests/serve/test_sglang.py:401` and
`tests/serve/test_trtllm.py:829` (both commented out), and `tests/utils/profile_pytest.py:1262` and
`:1326` (f-strings that *print* a recommended marker). The design analysis's earlier correction also
concluded there were **zero** `pytest.param(marks=…)` uses; that was an artifact of a line-oriented
regex whose `[^)]*` cannot cross the closing paren of the nested `pytest.mark.…(N)` call. Both are
wrong. Parsed with `ast` **[m]**:

```bash
# the instrument: parse every file, count real pytest.mark.profiled_vram_gib(...) Call nodes,
# and classify each by its nearest enclosing construct.
git grep -l 'pytest.mark.profiled_vram_gib' 0765d30ad8 -- 'tests/*.py'   # 29 files contain the token
# -> 106 real applications in 27 files
```

| form | count **[m]** |
|---|---:|
| decorator `@pytest.mark.profiled_vram_gib(N)` | **35** |
| `marks=[…]` field on an engine-config dataclass, spliced into `pytest.param(name, marks=marks)` at `tests/serve/common.py:508` (`VLLMConfig` 27, `SGLangConfig` 16, `TRTLLMConfig` 13, `VLLMOmniConfig` 1) | **57** |
| direct multi-line `pytest.param(…, marks=[…])` (`tests/test_predownload_models.py`, `tests/serve/test_token_budget_parity.py`) | **7** |
| module-level `pytestmark = […]` (6 files) | **6** |
| **computed at collection** from a topology matrix — `tests/utils/multimodal.py:612` appends `pytest.mark.profiled_vram_gib(topo_cfg.profiled_vram_gib)` | **1** |
| **total applications** (27 files) | **106** |
| **non-decorator** | **71 (67%)** |

The same shape holds for the other three resource markers, also by `ast` **[m]**:
`requested_vllm_kv_cache_bytes` **57 total / 22 decorator** (19 files); `requested_sglang_kv_tokens`
**27 / 10**; `requested_trtllm_kv_tokens` **17 / 5**.

**The conclusion is unchanged and is stronger than the retracted version.** Two-thirds of
applications are attached through a `marks=` list — 64 of them through the parametrize table — and
one is *computed from a topology configuration at collection time*, which a decorator cannot express
at all, not merely awkwardly. And the pre-commit marker hook must keep reading them: it runs
`tests/report_pytest_markers.py` in an isolated `language: python` venv with exactly **8**
`additional_dependencies` (`.pre-commit-config.yaml:170-183` **[m]**), and it performs a **real**
pytest collection (`report_pytest_markers.py:683` calls `pytest.main(["--collect-only", …])`
**[m]**). The harness cannot be installed into that venv. Worse, its `DependencyStubber` would not
degrade gracefully: `_StubModule.__getattr__` returns a real class (`report_pytest_markers.py:409`)
whose `__call__` returns another stub instance (`:365-366`), so `@requires(...)` applied to a test
function **replaces the function object** and the test disappears from the report entirely rather
than reporting as marker-less — `total_missing` never fires. A red or silently-blind pre-commit hook
affects every PR in the repository, not only PRs touching the harness.

Therefore: **markers are the wire format; `Requirements` is a read view over them, and declared
requirements are applied as real marks at decoration time — never by a collection-time hook.**

## M3 How the suite actually brings Dynamo up

Files matching each signature under `tests/*.py` at `0765d30ad8`, promoted from the design analysis
to a measurement **[m]**:

```bash
for P in 'ManagedProcess|DynamoFrontendProcess' 'kubernetes|kr8s|kubectl' 'mocker' \
         'ManagedDeployment' 'EtcdServer|NatsServer' 'DeploymentSpec' 'examples/' 'recipes/' \
         'docker (run|exec|compose)|docker_(run|exec)|DockerCompose'; do
  git grep -lE "$P" 0765d30ad8 -- 'tests/*.py' | wc -l
done
```

| Mechanism | Signature | Files **[m]** |
|---|---|---:|
| **Subprocess** | `ManagedProcess` / `DynamoFrontendProcess` | **56** |
| Kubernetes client/CLI | `kubernetes` / `kr8s` / `kubectl` | 18 |
| Mocker backend (GPU-free) | `mocker` | 22 |
| k8s DGD via `ManagedDeployment` | `ManagedDeployment` | 11 |
| Launches etcd/NATS itself | `EtcdServer` / `NatsServer` | 10 |
| k8s DGD via `DeploymentSpec` | `DeploymentSpec` | 8 |
| References `examples/` | path literal | **18** |
| References `recipes/` | path literal | **0** |
| Docker | `docker run/exec/compose` | **1** |

Two consequences revision 1 did not draw. **A Local lifecycle provider wrapping `ManagedProcess` is
the highest-priority provider by 56 : 1 over Docker** — and the `tests-v2` prototype implements
`Attached` (`deployment.py:68`) and `Docker` (`:82`) and neither Local nor Kubernetes **[p]**. The
porting priority is inverted from what was built. And `recipes/` has zero test coupling at
`0765d30ad8`: recipe-driven testing is a category of one, and it is new (PR #13979 adds the first).

The union of the three signatures a façade could plausibly host —
`{ManagedProcess|DynamoFrontendProcess} ∪ {kubernetes|kr8s|kubectl} ∪ {examples/}` — is **89 files**
**[m]**. That is the denominator of the burn-down schedule in *Implementation Phases*.

## M4 The manifest corpus is the unlock, and it is bigger than reported

The pathspec matters and is published, because the unrestricted grep also matches fenced YAML inside
`docs/**` and `README.md` (334 files, 42 of them unparseable as YAML). Restricted to actual manifests
**[m]**:

```bash
git grep -lE '^kind: DynamoGraphDeployment' 0765d30ad8 | sed 's/^0765d30ad8://' | grep -E '\.(ya?ml)$'
# 295 files; each then parsed with yaml.safe_load_all
```

| Property | Files **[m]** |
|---|---:|
| DGD-bearing files (`*.yaml` / `*.yml`) | **295** |
| multi-document | **116** |
| v1beta1 `spec.components` | **186** |
| v1alpha1 `spec.services` | **92** |
| **single-document AND v1alpha1** | **67** |
| DGD document present but neither schema key | **1** — `deploy/operator/config/samples/nvidia.com_v1alpha1_dynamographdeployment.yaml`, whose `spec:` is comment-only and parses to `None` |
| unparseable as YAML | **3** — all three under `benchmarks/frontend/dgd/templates/`, which are Jinja templates |
| `kind:` present but no loadable DGD document | **13** — all DGDR inputs and generated-override fixtures |

186 + 92 + 1 = **279** files carry a DGD document; **278** of those carry a *constructible* one (the
operator sample has no roles to construct). By root: **178** `recipes/`, **71** `examples/`, 15
`deploy/`, 15 `components/`, 7 `lib/`, 6 `tests/`, 3 `benchmarks/`.

Two further measurements on the same corpus **[m]**:

* **76 of the 178 `recipes/` files with a DGD document also ship a ConfigMap in the same file, and in
  all 76 the DGD references that ConfigMap by name.** A loader that selects the DGD document and
  discards the rest silently drops a ConfigMap the deployment mounts, in 43% of the recipe corpus.
* **Shell-invoked containers.** Corpus: every `DynamoGraphDeployment` document under `recipes/` and
  `examples/` (the two roots holding 249 of the 295 DGD-bearing files). Container walk: v1alpha1
  `spec.services[].extraPodSpec` (`mainContainer` and `containers[]`) and v1beta1
  `spec.components[]` (`container`, `podTemplate.spec.containers[]`, `extraPodSpec`). Predicate: a
  `command` list of length ≥ 2 whose last token is a short-option cluster ending in `c`.
  **184 containers are shell-invoked: 84 use `-c`, 100 use `-lc` across 51 manifest files.**
  `dgd_utils.py:313` at `0765d30ad8` is
  `if not isinstance(cmd, list) or len(cmd) < 2 or cmd[-1] != "-c": return False`, so **100 of 184
  (54%) shell-invoked services are treated as argv**: every flag scan returns nothing *and* every
  flag write is silently dropped at runtime. One predicate.
  (Over the whole `*.yaml` corpus rather than `recipes/`+`examples/` the figures are 199 / 99 / 100;
  the miss — 100 `-lc` containers across 51 files — is identical under both corpora. A fix widening
  the predicate to any short-option cluster ending in `c` is in flight on branch
  `dtokarev/dgd-shell-style-lc`; the measurement above is the state this proposal is written against,
  and phase 3 lists that fix as a dependency.)

Widening the predicate is necessary and **not sufficient**, and that changes the design rather than
only a number. The existing read→write path is `shlex.split(s)` → mutate tokens →
`" ".join(quote(t))`, and that round-trip is lossy in three independent ways
(measured on the same 184 containers, at `ebedc6cc13` **[m]**): containers carrying control operators
or redirections — **47** under the operator regex used there, and the count is sensitive to that
regex — re-emit `&&` quoted, collapsing the command; **17** carry whole-line `#` comments that
`shlex` has no concept of, and an apostrophe inside one of them opens a quote that never closes,
raising `ValueError: No closing quotation` on **4** recipes; and backslash-continued multi-line
scripts flatten to one line, discarding the comments that explain each flag. This is why REQ 5 and
§I1 require the value type to **retain the original string and splice on write**, and to have an
explicit `UNPARSEABLE` state — returning an empty token tuple for a command `shlex` cannot tokenise
is the same false-green shape as the `-lc` defect itself.

This is why the manifest layer, not the façade, is the highest-value single step: it is the
difference between "run the suite against any shipped recipe" being a slogan and being a command.

## M5 Three classes is the wrong number, and the missing three each hide work that did not happen

| Class | Definition | Portable? |
|---|---|---|
| **TC1 functional** | asserts on responses; deployment-agnostic | **yes — the target** |
| **TC2 config / topology** | varies flags, planes, layout, replica shape | yes, once lifecycle is a verb |
| **TC3 deployment artifact** | validates manifests, CRDs, admission | k8s-specific by nature |
| **TC4 library unit / upstream-ABI pin** | asserts on an installed Python object or an upstream engine's internals | not a deployment test at all |
| **TC5 image / wheel probe** | asserts on the container or wheel it is executing inside | needs `exec_in` or a Job |
| **TC6 benchmark / experiment** | output is data, not a verdict | yes, but needs a Report concept |
| *(TC7 `infra_helper`)* | not a test; a substantial fraction of `tests/` is process managers, spec builders, payload runners and fixtures **[a — the specific share is not re-derived here]** | — |

Measured cost of each misfiling:

* **TC6 filed as TC1.** `tests/lmcache/` contains 10 files and **zero `def test_`**, so it collects
  no pytest items; **no** file under `.github/workflows/` references it; and its correctness verdict
  — an MMLU accuracy A/B — is a printed line, because `run_test.sh` has exactly **5** `exit 1` sites,
  all on setup failures, and exits 0 after printing **[m]**. It sits under `tests/`, so it reads as
  coverage. It is unreachable by CI through three independent mechanisms.
* **TC5 filed as TC1.** `tests/dependencies/test_no_opencv.py:46` globs a hard-coded
  `/usr/local/lib/python3.12/dist-packages` and passes vacuously if the directory moves **[m]**;
  `tests/basic/test_cuda_version_consistency.py` shells `nvcc`/`dpkg`/`pip` and can only ever be
  right inside the image under test, leaving nothing behind to diff **[m — the file exists at that
  path; the shell-out characterisation is [a]]**.
* **TC4 filed as TC1.** `tests/serve/test_sglang_mm_hashes_protocol.py` and
  `tests/serve/test_trtllm_mm_hashes_protocol.py` pin upstream engine constants **[m — the files
  exist; the pinning characterisation is [a]]**: they fail if the image was built against an older
  upstream, which is an image fact, not a serving fact. And the framework's whole obligation to TC4
  is *collection integrity*, which is exactly what it does not have: `pyproject.toml:219` carries the
  unanchored `--ignore-glob=*vllm_integration*`, which also matches
  `tests/kvbm_integration/test_kvbm_vllm_integration.py` — **5 test functions that have never been
  collected** **[m]**. `--ignore-glob=*trtllm_integration*` at `:220` currently matches nothing and
  is the same loaded gun.

## M6 Shallow assertions and shallow health signals

Portability is worth little if the assertions underneath are weak, and the platform's own health
signals are weaker than they look. All five are carried from the design analysis **[a]**:

* **Accept-but-ignore** — a request is accepted with HTTP 200 and silently not honoured
  (`tool_choice: "required"` with no constrained-decoding backend configured). Asserting on status
  codes reports the capability as working.
* **Presence instead of substance** — asserting a field exists rather than that it is language; a
  worker emitting one repeated token passes.
* **Readiness is not correctness** — a component reports `Ready`, zero restarts, nothing logged, and
  serves unusable output. Readiness is reported to be defined three incompatible ways (DGD condition,
  DGDR phase, HTTP probe); that specific enumeration is **not re-verified here**, and REQ 6's split
  of `wait_ready` from `wait_serving` stands on the structural point alone: "the resource exists" and
  "the model answers" are different predicates and must not share a verb.
* **Prefix affinity masks a bad replica** — a fixed prompt pins to the same worker every time, so one
  healthy sample is mistaken for a healthy fleet.
* **The fault may never have landed.** The reported case is an RST-injection event that shelled a
  binary the debug image does not ship, where a `|| echo` fallback masked the missing binary and the
  measured goodput came in *above* baseline. The incident is **not re-verified here**; REQ 7 does not
  need it, because the structural argument is decisive on its own: an optional `prove=` keyword
  yields `proof=None` when omitted, which is not the same value as "proof attempted and failed", so
  the run never enters the unproven set and every dependent check runs on evidence from a fault that
  may never have landed.

Revision 1 names the first three. The last two are the ones that produce a confident green.

## Goals

* One functional suite that runs unchanged across the deployment shapes for which a provider exists,
  selected by configuration rather than by editing tests.
* Tests express intent against roles and verbs, never transport, argv or process detail.
* Every run leaves a **sealed, self-describing evidence bundle**, and every verdict is a pure
  function over that bundle — so a run can be re-scored offline, forever, after the GPU is released.
* A run that could not produce the evidence its checks require is **inadmissible**, not green.
* Test dependencies — model capability, configuration, topology, hardware, infrastructure powers —
  declared explicitly, evaluated three-valued, and **carried by the pytest markers CI already selects
  on**.
* Re-runnable across models, hardware targets and shipped recipes/examples by configuration alone.
* The harness is an installable distribution consumable by both the public GitHub repository and the
  internal repository, and it does not import `dynamo.*`.
* Provide the authoring mechanism DEP 0008 assumes when it requires tests to be "easy to write and
  run both locally and for CI" across "multiple programming languages and deployment targets".

### Non Goals

* Enumerating test cases, or replacing the taxonomy and lifecycle defined in DEP 0008.
* Redefining CI **pipeline structure or triggering** — DEP 0009 owns that, and this proposal changes
  neither. It does add exactly **two gates**, named here so the scope is not silently wider than the
  sentence above: (a) the marker-parity gate of REQ 13, which fails a PR whose collection differs
  from its merge base in the loss direction; and (b) `INADMISSIBLE` as a distinct CI failure at
  exit 1 (REQ 11), which is a new class of red that is not a product regression. Both are additive
  and neither changes which pipeline runs which tests. If DEP 0009's owners want these recorded
  there instead, this proposal defers to that.
* Specifying a coverage metric.
* Hosting **all** of `tests/`. Six exclusions are stated explicitly and are load-bearing, because
  "one framework for `tests/`" would be false:
  1. **TC4 library-unit and upstream-ABI pins.** No SUT, no reach, no bundle. They stay plain
     pytest. The framework owes them markers and collection integrity, nothing else.
  2. **In-process PyO3 runtime-object tests.** REQ 17's no-`dynamo.*` rule forbids the harness from
     importing the runtime, and relaxing it welds harness version to runtime version and kills
     REQ 16. They get `@requires` for selection and nothing else. The portability answer for their
     *assertions* is HTTP twins, added on demand.
  3. **The reverse `examples/` coupling.** `Plan.from_launch_script` hosts the forward direction.
     The private profile-override environment variables the *shipped* scripts read, and the modules
     that shell out to an example's memory-argument helper (load-bearing for the VRAM scheduler),
     are a product/test boundary problem. Named, tracked separately, not smuggled in. Porting that
     helper to a Python function is a cheap independent win and is its own work item.
  4. **Offline replay of in-timeline findings.** `require_*` results are recorded but not
     re-derivable from the sealed bundle; `dynamo-test judge` re-runs deferred checks only. The more
     a test asserts live, the less of it replays. It is the right trade and it is a trade.
  5. **Interactive debugging from the verdict surface.** The ACT receiver is phase-guarded, so you
     cannot poke the SUT while iterating on a check. An ACT-phase REPL against a held-open deployment
     is the workaround — an operator tool, not a test.
  6. **Revision 1's full six-shape platform claim.** A functional test runs unmodified on Local,
     Kubernetes, Attached and Reference *given a provider*. Same-container, container-per-component
     and Compose are modelled and **not claimed**; no test exercises them today.
* Sharing test *content* across the repository boundary. Only schemas and mechanisms cross.

## Requirements

### REQ 1 Four Phases And Three Receivers

A test MUST be structured as **PLAN → ACT → COLLECT → CHECK**. PLAN is pure, has **no receiver**, and
produces a `Plan` and a `Site`. The three phase-scoped receivers are:

| Phase | Receiver | Holds | Forbidden |
|---|---|---|---|
| ACT | `Sut` | live system; verbs return `Handle`s | computing a verdict |
| COLLECT | `Recorder` | harness-driven; runs **before teardown**, unconditionally, including on abort or signal | authored logic |
| CHECK | `Evidence` | sealed on-disk bundle | any reference to the live system |

CHECK functions MUST be synchronous and MUST NOT be able to reach the SUT: `Evidence` MUST NOT carry
a deployment handle. The `Sut` MUST raise a typed `PhaseError` naming the phase and the replacement
verb if an ACT verb is called after the ACT phase has ended.

### REQ 2 Portability Across Deployment Platforms, Honestly Scoped

A functional test MUST run unmodified against every platform for which a provider exists. The
providers this proposal commits to are **Local (processes)**, **Kubernetes**, **Attached** (an
externally-owned deployment, query-only) and **Reference** (a non-Dynamo server, for comparison
arms). Docker ships as-is and experimental. Same-container, container-per-component and Compose are
representable in the model and are explicitly not claimed until a test needs them.

### REQ 3 Two Reach Substrates; Addressing Is Derived, Never Supplied

The framework MUST model at least two substrates, and MUST NOT conflate them:

| Substrate | Vantage | Properties |
|---|---|---|
| `Reach` | the test process | has a **lifetime and a budget** — a port-forward can leak and can be rate-limited |
| `Ingress` | a harness-owned workload **inside** the SUT's network | no lifetime; has **placement**, which may be the opposite of the SUT's |

A third direction — **the SUT reaching a test-owned sink** — MUST be representable (`Sink`): media
fixtures, an OTLP collector, an SSRF canary, a trace sink. Today these bind loopback and are
unreachable from a pod.

Addresses MUST be **derived from the `Plan` by the provider**, not passed to a constructor. A
provider that cannot supply a substrate MUST raise `Unreachable(kind, why)` at PLAN time rather than
widening or guessing. `attach(url)` remains expressible as a `Plan` whose base is `Attached`, whose
`Ingress` raises — so a load-generating test against an attached deployment fails by name at PLAN
time, not after a long readiness spin.

### REQ 4 Ownership

The framework MUST be able to **own** a deployment, not only attach to one:

* render a per-arm desired state from a manifest, an argv line, a launch script, or a declarative
  request;
* **apply every document** in the manifest, not only the DGD;
* prepare the environment above the lifecycle provider (namespace provisioning, log storage, model
  prefetch, credential preflight with a remediation string);
* refuse to start dirty, with a typed error;
* tear down owner-first with aggregate error reporting.

Every destructive behaviour (namespace scrub, infrastructure restart, PVC deletion) MUST be an
opt-in `Policy`, **default off**, and namespace scrubbing MUST additionally require a `Site`
declaring `namespace_ownership: dedicated`. A scrub that is correct in a dedicated namespace deletes
other tests' resources in a shared one.

### REQ 5 A Desired-State Document, Lossless Argument Mutation, And Semantic Settings

There MUST be a first-class noun for *what gets patched* — per-role `image`, `env`, `args`,
`replicas`, `resources`, `placement`, `probes`, `mounts` — rendered by the provider to a DGD, to argv
plus environ, or to a launch-script invocation.

Argument mutation MUST **replace**, never append; MUST support removal; and MUST emit boolean flags
correctly. It MUST additionally be **lossless**, which is stronger than preserving the shell-versus-
argv form:

* The argument value type MUST **retain the original command string** for shell-invoked containers
  and MUST perform writes by **splicing that string**, not by tokenise → mutate → re-join. §M4
  measures three independent ways the round-trip loses meaning (control operators, whole-line
  comments, line continuations).
* The form enumeration MUST NOT name the majority case as an exception: `-lc` is 100 of 184
  shell-invoked containers, so the variant is `SHELL`, not `SHELL_LC`.
* There MUST be an explicit **`UNPARSEABLE`** state distinct from "no arguments". A command the
  tokeniser cannot read MUST yield `UNKNOWN` facts and MUST make writes raise, never silently
  produce an empty token tuple.

Backend-portable settings MUST be expressed semantically (`context_length=4096`) and rendered per
engine; free-form kwargs that are backend-locked MUST NOT be the interface. Named configurations
(for example router modes) MUST be **validated, enumerable presets**, not free-form strings.

### REQ 6 Verbs Take A Role; Every Verb Returns A Handle

The ACT surface MUST be verb-first with the target as an argument (`sut.kill_process(at("worker"))`),
not component-first (`dynamo.worker.kill()`). Selection — role, replica, policy, rank, process, port
— MUST be carried by **one** selector value, whose constructor takes explicit keyword-only
parameters so that a misspelled selector is a `TypeError` at the call site rather than a silent
no-op. Every verb MUST return a `Handle` carrying `started_at`/`ended_at`, the arguments, and **what
the selection policy actually resolved to**. A provider that cannot honour the requested scope MUST
raise `Refused`; it MUST NOT silently widen. Readiness MUST be split: `wait_ready` (the resource or
process exists) is a different predicate from `wait_serving` (the model answers) and they MUST NOT
share a verb.

### REQ 7 Fault Evidence Is A Contract

Every fault-class verb MUST declare, **at registry import time**, the proof fields its providers must
produce. A verb whose provider cannot produce them MUST mark the handle `UNPROVEN`, and a run
containing an unproven fault MUST be **inadmissible**, not passing. An optional `prove=` keyword is
insufficient: omitting it yields no proof object at all, which is not the same as a failed proof, and
every dependent check then runs on evidence from a fault that may never have landed.

### REQ 8 Load Generation Is A First-Class Actor

A sustained, shaped, measured workload MUST be modelled as an actor with a lifetime — started,
observed, cancelled, harvested — distinct from a single request. It MUST support open- and
closed-loop shapes, rate staircases, bursts, sequence-length mixtures and trace replay, and its
placement MUST be independently expressible from the SUT's. Third-party generator CLI passthrough
MUST be quarantined in one field so the workload model does not drift with the tool's flags.

### REQ 9 The Evidence Bundle

Every run MUST write a versioned bundle with: a canonical timeline (one record per handle), a
**producer index** mapping every artifact to the handle and tool that wrote it, normalised request
records with **one** definition of failure, per-role logs including previous-container logs, metric
series, pod inventory, fault proofs, exec captures, every applied document, and the findings. Checks
MUST resolve artifacts through the producer index and **MUST NOT glob the filesystem**. A check that
addresses a *set* of producers MUST do so by querying the producer index (for example, "every
producer of kind `load`"), which is a typed query with a resolvable, recorded answer, not a path
pattern. Verbatim third-party output MUST be quarantined in a `raw/` subtree. The bundle MUST grow
append-only, so a killed run still leaves a judgeable bundle. CHECK output (findings, run outcome)
MUST be written **outside** the sealed subtree and versioned separately, so that re-judging a bundle
does not mutate the sealed evidence.

### REQ 10 Checks Are Values; Four Verdicts; Reports Never Gate

Checks MUST be named, parameterised, serialisable values with a declared read-set, not inline
`assert`s. Verdicts MUST be four-valued: `PASS`, `FAIL`, `OBSERVED` (measured on purpose; never
gates), `UNKNOWN` (could not be evaluated; **never contributes PASS**). Gating polarity MUST be
carried by the check's **name**, not by a keyword argument, and validated mechanically by
**longest-matching prefix**, so that a longer gating prefix is never shadowed by a shorter one that
is its proper prefix. Reports MUST run before checks, MUST never gate, and a raising report MUST
become a finding rather than an exception. **Every** check MUST run; there is no fail-fast.

### REQ 11 Run Validity Is A Separate Gate From The System Verdict

The framework MUST distinguish *"is this measurement admissible?"* from *"did the system pass?"*.
Producers MUST declare what they promise (artifact kind, key, minimum row count, window); SEAL MUST
measure what arrived; the delta MUST be recorded. A configuration that legitimately sheds 99% of
requests but ran its full window is a **valid** measurement that may fail its SLO honestly; a run
whose producer emitted nothing is **inadmissible** and MUST NOT emit a system verdict. Admissibility
MUST be evaluated first, and when it fails, every other failure MUST be downgraded to `OBSERVED` — a
collapsed run cannot indict the system.

There MUST be exactly **one** authority for admissibility. The `Seal` computed at COLLECT is that
authority; `require_valid_*` checks are CHECK-phase *readers* of the seal that add
producer-specific criteria, and MUST NOT be able to declare a run admissible that the seal marked
incomplete. Where they disagree, the seal wins and the disagreement is itself recorded as a finding.

### REQ 12 Three-Valued Evaluation, Pushed To The Fact Source

Applicability evaluation MUST be `SATISFIED | UNSATISFIED | UNKNOWN`, and `UNKNOWN` MUST NOT be
treated as unsatisfied.

Requirements (unmet ⇒ skip) MUST be distinguishable from preconditions (unmet ⇒ error), and the
distinction MUST be **carried by the type, not by a comment**: every declared need is a value
carrying an explicit polarity, and the polarity is serialised into the bundle alongside the need.

Additionally — and this is new — **every fact source MUST itself be three-valued**. A source that
cannot *see* a form MUST return `UNKNOWN`, never "absent". Blanket `except Exception: return None` in
a fact source MUST be lint-forbidden. An `UNKNOWN` rendered as `UNSATISFIED` is a confident, false,
green-looking skip.

### REQ 13 Markers Remain The Wire Format; The Parity Gate Must Be Real

Declared requirements MUST be **applied as real `pytest.mark` objects at decoration and
parametrization time**, exactly as hand-written markers are. Marker names, argument types and float
identity MUST be preserved exactly (§M2/R6).

**The harness MUST NOT register a `pytest_collection_modifyitems` hook that adds or removes
markers.** This is a hard requirement, not a preference, and it is what makes the migration
CI-safe. The repository-root `conftest.py:91-122` already runs `pytest_itemcollected`, which adds
`pre_merge`, `gpu_0` and `defaulted` to any item declaring no marker from the `Lifecycle` or
`Hardware` categories of `tests/marker_categories.py:REQUIRED_CATEGORIES` **[m]**. That hook fires
**before** any `pytest_collection_modifyitems`, including a `tryfirst` one. A collection-time
projector keyed on marker *name* would therefore see `gpu_0` already present, add `gpu_1` beside it,
and leave the item carrying `gpu_0`, `gpu_1` and `defaulted` — which the CPU parallel lane selects.
A declared GPU test would land on a GPU-less runner. Applying at decoration time avoids the problem
entirely: the marker is present before `pytest_itemcollected` runs, so the default is never added,
and no reconciliation is needed.

`Requirements.of(item)` MUST derive from existing markers when no declaration is present, and MUST
treat a category marker on an item that also carries the `defaulted` marker as **`UNKNOWN`**, never
as a declared requirement — otherwise the read view invents a `pre_merge`/`gpu_0` requirement for
every unmarked test in the repository.

Every marker name the declaration can emit MUST be validated at import against the marker list
registered in `pyproject.toml` (**68 markers** at `0765d30ad8` **[m]**). `--strict-markers` is in
`addopts`, so an unregistered name is a collection **error in every lane**, not a skip.

The parity gate MUST NOT use `--collect-only`. `tests/conftest.py:693-712` guards **both** the
`--max-vram-gib` deselection **and** `write_test_meta(items)` on `not config.option.collectonly`
**[m]**, so a `--collect-only` gate is structurally blind to the only mechanism that removes tests
and to the metadata the GPU orchestrator consumes. The gate MUST therefore:

* run a **real collection under `--dry-run`** — a flag that **already exists**, registered at
  `tests/conftest.py:153` and evaluated at `:717` **inside** the same `trylast`
  `pytest_collection_modifyitems`, *after* the deselect and *after* `write_test_meta`. It is not a
  new flag and MUST NOT be reimplemented;
* assemble the dump from three sources, because `tests/conftest.py:761` executes `items.clear()`
  under `--dry-run` and any post-collection reader therefore sees zero items: the full collected list
  captured in a `tryfirst` hook; **every removal reported to `pytest_deselected`** — which
  `tests/conftest.py:704` calls and `tests/fault_tolerance/deploy/conftest.py:65` also calls, so the
  gate must be correct against **three** `pytest_collection_modifyitems` implementations, not one;
  and the `write_test_meta` record read from its file;
* give the head and base runs **distinct `TMPDIR` values**, because `write_test_meta` writes one
  fixed path under `tempfile.gettempdir()` (`tests/utils/vram_utils.py:195`) and only when the record
  is non-empty (`:194`) **[m]**. Run in the same shell, the base run overwrites head's file — or,
  if base's record is empty, head's file survives and the gate diffs it against itself and is green
  by construction;
* compare against a base captured from an explicit `git worktree` at the merge base (`git stash` on
  a clean CI checkout is a no-op comparing HEAD to HEAD);
* be keyed by `(image_tag, arch, marker_expr)`, because collection is image-dependent.

Declaring GPUs without a VRAM declaration MUST be an import-time error, but requiring the VRAM marker
unconditionally is **wrong**: `.github/workflows/shared-test.yml:251` routes the sequential GPU lane
as `({0}) and not profiled_vram_gib` **[m]**, so the marker is a *routing selector* as well as a
scheduler budget. The rule MUST be three-way: profiled value → parallel lane; an explicit
`SEQUENTIAL` sentinel emitting no marker → sequential lane; neither → error.

### REQ 14 Six Test Classes, Declared And Defaulted

The six classes of §M5 MUST be distinguishable, and `infra_helper` MUST be a declared non-test
category. Class MUST default from a path map so the whole corpus acquires a class with zero edits,
with an explicit declaration overriding.

The gating classes are `functional`, `topology`, `artifact` and `image_probe`. An item of a gating
class **that entered PLAN** — that is, one that constructed a `Plan`, a `Sut` or an `Evidence` —
whose gate-polarity check list is empty MUST be a **PLAN-time error**. Items that never enter PLAN
(plain pytest tests, including every `library_unit` item and every unmigrated file) are unaffected;
this is what lets the path map cover the whole corpus on day one without turning every unmigrated
test into an error. Only the `experiment` class may declare an empty gate list, and that declaration
is the opt-out. Bare `assert` in a test body MUST NOT count as the assertion of record for a gating
class that entered PLAN, because `PYTHONOPTIMIZE` strips it.

### REQ 15 Isolation Of Infrastructure Access, On Both Planes

On the **control** plane, powers MUST be a value checked at construction; a call without the grant
MUST raise a named error. A collection-time AST lint SHOULD additionally flag a `functional`-class
item that names a lifecycle, fault, exec or log verb; that lint is **advisory and cannot be the
enforcement mechanism**, because a test body may delegate to a shared helper the lint cannot see
through. The construction-time grant check is the enforcement.

On the **evidence** plane, scoping MUST also apply: logs arrive as files, so without scoping any
check can read any component's logs with no handle at all. `Evidence` MUST carry the granted evidence
kinds and MUST raise on an out-of-scope read.

### REQ 16 Matrix Re-Runnability And One Kubernetes Implementation

The suite MUST be re-runnable across models, hardware targets and shipped recipes/examples by
configuration alone, with no test edits. Absent roles MUST fail fast and by name, not by timeout. The
framework MUST run without a GPU wherever the mocker backend suffices.

There MUST NOT be a third Kubernetes implementation. The provider MUST consume the shared
**manifest model from `tests/deploy`** rather than reimplement it, and MUST provide the **operations
capability set** enumerated in §M2/R5 — namespace lifecycle including scrub with dependent-resource
cascade, log-collection storage with in-pod capture, model prefetch, run provenance, signal handling
with emergency delete, per-pod metrics capture, and an in-pod exec transport — rather than
reimplement those either. The normative object is that capability set, which is stated in this
document. The internal peer harness is the identified *source* of that capability set and is where
the implementation should be lifted from; it is named in [Terminology](#terminology--definitions),
but conformance is defined against the enumerated capabilities, not against a repository a public
reviewer cannot read.

### REQ 17 Packaging

The harness MUST be an installable distribution with its own version, living in the
`ai-dynamo/dynamo` repository, consumable by `dynamo`'s own tests and by the internal repository by
pinned reference.

* The harness MUST NOT import `dynamo.*`. Otherwise its version is welded to the runtime version and
  REQ 16's "point one suite at an older release" becomes impossible.
* Exactly **one module may register pytest hooks or fixtures** (`pytest_plugin.py`). One further
  module — `requirements.py` — MAY `import pytest`, solely to construct `pytest.mark` objects and to
  type an `item` parameter; the import-boundary test MUST assert that it uses no pytest name other
  than `mark`, and that no third module imports pytest at all. (Stating this as "exactly one module
  may import pytest" would be self-contradictory, because REQ 13 requires `Requirements(...).marks`
  to *be* real mark objects.)
* The PLAN and CHECK subtrees (manifest model, evidence, checks, reports) MUST import stdlib plus a
  YAML parser and nothing else — no Kubernetes client, no HTTP client — so PLAN-time validation and
  offline judging run on a laptop, in CI's bare-checkout tier, and inside a runner with no cluster
  credentials.

### REQ 18 Environment Profile, Selection Policies, Scenario Documents, Suites, Long Runs

Five requirements that follow from operating this at scale:

* **Environment profile.** Per-site facts (hardware envelope, per-cluster workarounds, auth with a
  remediation string, provisioning recipe) MUST be an opaque validated object with a **bleed check**:
  test-scoped keys in a site profile, and site-scoped keys in a run, MUST be hard errors.
* **Selection policies.** Replica and process selection MUST support first / all / random / fraction
  / rank / "hottest by live inflight requests", and the resolution MUST be recorded in the handle.
* **Scenario document + generated registry.** A test MUST be serialisable to a document, and the
  primitive registry MUST be **generated** from the same definitions the imperative surface uses, so
  the two authoring modes cannot drift.
* **Suites with negative controls.** A suite MUST support `expect: fail` arms. A verdict-inverting
  layer MUST distinguish harness failure from system failure: only a genuine system `FAILED` may be
  inverted. Inadmissible, harness-error and argument-parse failures MUST NOT be scored as a pass.
* **Long-run hazards.** Abandoned-run cleanup, a signal-escalation ladder that does not depend on
  process-group delivery, and an interpreter-exit escape for hung non-daemon threads are **day-one
  requirements**, not hardening items. They land with the phase that introduces the loop thread
  (Phase 5b) and are named in that phase's gate. The interpreter-exit escape needs no measurement to
  justify: a blocking call made from a worker thread is not interruptible by `KeyboardInterrupt`, and
  a non-daemon thread blocks interpreter exit, so without an explicit escape a wedged provider call
  hangs the process indefinitely.

### Requirement Coverage Of The Identified Gaps

| Gap | Requirement |
|---|---|
| G1 load generation | REQ 8, REQ 3 (`Ingress` placement) |
| G2 timed event timeline | REQ 6 (handles), REQ 9 (canonical timeline) |
| G3 evidence bundle | REQ 9 |
| G4 checks as values, third verdict | REQ 10 |
| G5 reports that never gate | REQ 10 |
| G6 run validity as a gate | REQ 11 |
| G7 desired-state document | REQ 5 |
| G8 environment profile + bleed check | REQ 18 |
| G9 replica / process selection | REQ 18, REQ 6 |
| G10 fault evidence as a contract | REQ 7 |
| G11 scenario document + registry | REQ 18 |
| G12 suites with negative controls | REQ 18 |
| G13 long-run operational hazards | REQ 18 |

# Proposal

## P1 The phase machine

```
PLAN      pure. no receiver. no I/O beyond reading documents and manifests.
  Plan x Site x params -> ResolvedPlan; role bindings derived; requirements evaluated
  diagnostics, all before a GPU is allocated:
    unknown verb - dangling handle reference - duplicate name
    a check reads evidence no step produces
    GPUs declared with neither a profiled VRAM value nor the SEQUENTIAL sentinel
    gating-class item that entered PLAN with zero gate checks   (weak-verdict error)
    site/plan key bleed - override key not declared as a parameter

ACT       receiver: Sut. Owns the live system.
  every verb records a Handle with wall-clock brackets, resolved selection, and proof
  the bundle grows APPEND-ONLY, so a killed run still leaves a judgeable bundle

COLLECT   receiver: Recorder. Runs BEFORE teardown, unconditionally,
  including on abort or signal: logs, previous logs, metrics, inventory, restart counts,
  processes, applied documents -> bundle.  Then SEAL: pure; writes the run document,
  the producer index, and promises versus delivery. From here the SEALED SUBTREE is immutable.

CHECK     receiver: Evidence. Pure, synchronous, offline, separate process.
  1. admissibility first        -> is this measurement usable at all
  2. reports                    -> never gate; a raising report is a finding
  3. gates and observations     -> EVERY one runs; never fail-fast
  output is written OUTSIDE the seal, so re-judging never mutates the evidence
```

Five ordering rules, stated because revision 1 states none of them:

1. Tear the deployment down **before** analysis, so analysis reads artifacts and CI releases the GPU
   runner before judging.
2. Collect before teardown, unconditionally.
3. Reports run before checks and never gate.
4. Every check runs; there is no fail-fast.
5. Admissibility runs first, and if it fails, every other failure is downgraded to `OBSERVED`.

Rule 2 is what makes the other four work. It is also the rule the peer harness learned by having
checks fail against a torn-down deployment (§M2/R1).

## P2 Two reach substrates and a third direction

| Direction | Protocol | Why it is separate |
|---|---|---|
| test process → SUT | `Reach` | 78 of 112 HTTP-building files pin localhost **[m]**; a port-forward has a lifetime, a budget, and leaks |
| deployed generator → SUT | `Ingress` | the load path addresses the frontend by in-cluster service DNS derived from the manifest, and the generator's **placement may be the opposite of the SUT's** — on an arm64 GPU cluster an amd64-only generator must land on CPU nodes **[a]** |
| SUT → test-owned sink | `Sink` | media fixtures, an OTLP collector, an SSRF canary, a trace sink — all loopback-bound today, therefore unreachable from a pod **[a]** |

The addressing rule replaces revision 1's constructor. You never pass a URL to the façade:

```python
# withdrawn
Dynamo(transport=Http("http://localhost:8000"), deployment=K8s(ns=..., name=...))

# proposed
plan = Plan.from_manifest(recipe_path).with_image(site.image)
with bring_up(plan, site, name="sanity") as sut:
    sut.url()                                     # Reach:   from the test process
    sut.address(vantage=Vantage.WORKLOAD)         # Ingress: from inside the SUT's network
    sut.publish(png, kind="image", reachable_from=at("worker"))   # Sink
```

Two independent constructor arguments cannot express *"ask the deployment for its address"*, which is
exactly what the in-cluster load path does.

## P3 The ownership ladder

```
Plan.from_manifest(path)         # a DGD bundle: every document, both schemas
Plan.from_argv(role -> argv)     # local processes
Plan.from_launch_script(path)    # examples/**/launch/*.sh, env-injected
Plan.request(model=, sla=)       # declarative: the operator synthesises the deployment
Plan.attach(url)                 # someone else owns it; Ingress raises Unreachable
Plan.reference(framework, args)  # a non-Dynamo server, for comparison arms
Plan.empty()                     # no SUT; only the "self" role, for image probes
```

Each is a constructor of the same `Plan`; the provider renders it. `Plan` mutation returns a **new**
`Plan`. This structurally prevents a class of defect the design analysis reports in the
fault-tolerance scenarios, where many scenarios share a smaller number of spec objects and a single
import-time mutation leaks a flag into unrelated scenarios **[a — the specific scenario and spec
counts are not re-derived here and are omitted]**. The structural argument does not need them:
mutation of a shared object is observable by every other holder of that object, and an immutable
`Plan` makes the class unwritable.

## P4 The verb façade, with the role as an argument

The prototype's provider layer is **already** role-parameterised, which is what makes the change
cheap in the layer that matters **[p, `dtokarev/tests-v2-harness` @ `2fd6eb2586`]**:

```python
# tests-v2/dynamo_harness/deployment.py  -- the Deployment protocol
def restart_component(self, role: str, **flags: str) -> str: ...   # :57  role-aware
def stop(self) -> None: ...                                        # :45  NO role parameter
def kill(self) -> None: ...                                        # :51  NO role parameter

# tests-v2/dynamo_harness/components.py
def restart(self, **flags): self._controllable().restart_component(self.role, **flags)  # :69-71
def stop(self):             self._controllable().stop()                                 # :73-74
def kill(self):             self._controllable().kill()                                 # :76-77
```

The defect is at the **protocol** level, and it is one of expressiveness, not of an implementation
bug. `restart_component` takes a role; `stop` and `kill` cannot take one, so `dynamo.worker.kill()`
has nowhere to put "worker". In the only concrete provider, `Docker.stop` (`:191-194`) is
`docker rm -f <name>` and `Docker.kill` (`:196-198`) is `docker kill <name>` — both bounce the whole
container including the frontend. A fault-tolerance test written against that API ("kill the worker,
assert the frontend returns 503") looks correct and cannot be told from the outside that it tested
something else.

The prototype is honest about the coupling *where the API can express it*. `restart_component`'s own
docstring at `:211-215` reads:

> Both processes share a container in this topology, so recreating it bounces the other component
> too. That is a property of the deployment shape, not of the request: a container-per-component or
> Kubernetes provider would restart only the named one. The flags, however, are always routed to the
> named component alone.

That is exactly right, and it is the argument. A shape constraint is a legitimate provider answer;
the component-first surface has no way to *say* it, and no `Refused` to raise, so the caller cannot
distinguish a shape constraint from a bug. The verb form puts the scope at the call site and forces
every provider to answer "which role?" or raise `Refused`.

The same file shows why free-form kwargs are the wrong configuration interface **[p]**: at
`deployment.py:217-219`, `restart_component` does
`target += [f"--{key.replace('_','-')}", str(value)]` against the live list returned by `args_for`
— it **appends**, so repeated restarts produce `--max-model-len 4096 --max-model-len 2048 …`; there
is no removal; and a boolean flag renders `--enforce-eager True`.

**Ordering, not co-shipping.** The façade **must not land before** the CHECK-phase receiver. On its
own it moves the torn-down-deployment landmine into a new API: a single object carrying both
`sut.kill(...)` and `sut.expect_no_errors(...)` reads as valid at every point in the test and is only
ever half true. This is expressed as a phase-ordering constraint (`Evidence` is Phase 4, `Sut` is
Phase 5) and as an explicit line in Phase 5's gate, so that a revert of Phase 4 after Phase 5 lands
is caught rather than silently recreating the landmine.

## P5 Six classes, defaulted from a path map

Class defaults from a path map so the entire corpus acquires a class with zero test edits, which is
what makes the weak-verdict rule (REQ 14) enforceable across the corpus rather than only on migrated
files. Every path below is verified non-empty at `0765d30ad8` **[m]**:

```toml
[tool.dynamo_test.testclass]
"tests/dependencies/**"           = "image_probe"
"tests/wheels/**"                 = "image_probe"
"tests/basic/**"                  = "image_probe"   # test_cuda_version_consistency.py shells nvcc/dpkg/pip
"tests/deploy/test_dgd*_utils.py" = "library_unit"
"tests/utils/test_*.py"           = "library_unit"
"tests/lmcache/**"                = "experiment"
"tests/vllm_self_benchmark/**"    = "experiment"
"tests/**"                        = "functional"   # last match wins; explicit declaration overrides
```

`artifact` has no path-map entry because no directory holds artifact tests today; the recipe-lint
suite Phase 3 introduces is the first, and it declares its class explicitly. (Revision 1's draft map
contained a `tests/docs/**` entry; that directory contains **0 files** at `0765d30ad8` **[m]** and is
removed.)

## P6 Markers stay the wire format

```python
Requirements.of(item)      # reads a declaration if present; OTHERWISE derives from authored markers,
                           # treating any category marker on a `defaulted` item as UNKNOWN
Requirements(...).marks    # real pytest.mark objects: usable as decorators, inside
                           # pytest.param(marks=[...]), and appended to a computed marks list
```

Marks are applied at **decoration and parametrization time**, which is the same mechanism the corpus
already uses for 71 of 106 `profiled_vram_gib` applications (§M2/R6). The harness registers no
collection hook that mutates markers (REQ 13). The consequence is that the harness can consume all
217 files before one file adopts a decorator, the repository-root category defaults keep working
unchanged, and the pre-commit marker hook keeps reading real `pytest.mark` objects forever. This is
the single decision that makes the migration CI-safe, and it is the reason the requirement DSL is
deliberately second-class for a long time.

# Implementation Details

Import root `dynamo_test`; distribution `dynamo-test-harness`; directory `harness/` in
`ai-dynamo/dynamo`. Enum bodies below are given in full where the values are normative and elided
with `...` where they are not; every signature is real Python.

## I1 Values — stdlib only; no pytest, no HTTP, no Kubernetes, no `dynamo.*`

```python
Role = Literal["frontend", "worker", "prefill", "decode", "encode", "router",
               "planner", "operator", "kvbm", "load", "self", "etcd", "nats", "gateway"]

class Process(str, Enum):     # semantic; resolved per backend by EngineDialect
    MAIN = "main"; ENGINE = "engine"; WORKER = "worker"; RANK = "rank"; DEATH = "death"

class Policy(str, Enum):
    FIRST = "first"; ALL = "all"; RANDOM = "random"; HOTTEST = "hottest"

class PortName(str, Enum):
    SERVICE = "service"; SYSTEM = "system"; METRICS = "metrics"; GRPC = "grpc"

@dataclass(frozen=True, slots=True)
class Sel:                                  # THE selector; hoisted out of every verb signature
    role: Role
    replica: int | None = None
    policy: Policy | None = None
    fraction: float | None = None
    rank: int | None = None
    process: Process | None = None
    port: PortName = PortName.SERVICE

def at(role: Role, *, replica: int | None = None, policy: Policy | None = None,
       fraction: float | None = None, rank: int | None = None,
       process: Process | None = None, port: PortName = PortName.SERVICE) -> Sel: ...
```

`at()` takes **explicit keyword-only parameters**, not `**kw`. With `**kw`, `at("worker", replicas=0)`
— a plausible typo for `replica` — is a silent no-op that selects a different worker set; with the
signature above it is a `TypeError` at the call site.

### Role resolution happens in exactly one place

```python
@dataclass(frozen=True)
class RoleBinding:
    role:          Role
    service:       str                    # "VllmDecodeWorker" | "decode" | "dynamo.vllm"
    log_key:       str                    # bundle directory AND log-scrape key — ONE spelling
    metric_labels: Mapping[str, str]
    identity:      IdentityScheme         # the ONE pod/replica identity encoding
    argv_target:   str
    processes:     Mapping[Process, str]
    ports:         Mapping[PortName, int]

class RoleTable(Mapping[Role, RoleBinding]):
    @classmethod
    def derive(cls, plan: Plan, dialect: EngineDialect) -> RoleTable: ...
    def require(self, role: Role) -> RoleBinding:
        """Raise UnknownRole(role, sorted(self)) at PLAN time, never at fault time."""
```

The table is derived once and **serialised into the bundle**, so a default admissibility check can
prove that every role bound. This matters because the failure mode is not a typo: a role that
resolves to a *wrong-but-valid* log key yields an empty log stream, which is "present and empty", not
"absent", so an absence rule alone does not catch it. Resolution proof does. String addressing
without one resolution table is what produces vacuous log-reading passes on backends whose service
names differ from a hard-coded alias list **[a]**.

### Facts and needs are three-valued, and polarity lives in the type

```python
class Status(str, Enum):
    KNOWN = "known"; ABSENT = "absent"; UNKNOWN = "unknown"

@dataclass(frozen=True)
class Fact(Generic[T]):
    status: Status
    value:  T | None
    source: str        # "DGD spec.services[VllmDecodeWorker].args[7]" | "nvidia-smi" | "node label"
    detail: str = ""
    def __bool__(self) -> NoReturn:
        raise TypeError("Fact has no truth value; branch on .status")

class Polarity(str, Enum):
    REQUIREMENT  = "requirement"    # unmet -> attributed skip
    PRECONDITION = "precondition"   # unmet -> ERROR, because skipping would hide a defect

@dataclass(frozen=True)
class Need(Generic[T]):
    value:    T
    polarity: Polarity = Polarity.REQUIREMENT
```

`__bool__` raising is deliberate: it makes `if not fact:` unwritable. `Need` carries REQ 12's
polarity **in the type**, so `source_tree=Need(True, Polarity.PRECONDITION)` is a value the PLAN
diagnostics and the bundle both see — a comment saying `# PRECONDITION` is not a mechanism. Lint rule
inside the fact sources: no blanket `except Exception: return None`, and a source that cannot see a
form returns `UNKNOWN`, never `ABSENT`.

### Argument vectors: lossless by construction

```python
class ArgForm(str, Enum):
    ARGV        = "argv"          # command is the program; args are already tokens
    SHELL       = "shell"         # command ends in a -...c cluster; the payload is a script
    UNPARSEABLE = "unparseable"   # a script is present and the tokeniser cannot read it

@dataclass(frozen=True, slots=True)
class ArgV:
    form:   ArgForm
    raw:    str | None                     # the ORIGINAL script, byte-for-byte, when form is SHELL
    tokens: tuple[str, ...]                # () if and only if form is UNPARSEABLE
    shell:  tuple[str, ...] = ()           # ("bash", "-lc") — the invocation, preserved verbatim
    spans:  tuple[tuple[int, int], ...] = ()   # byte offsets of each token within `raw`

    def get(self, flag: str) -> Fact[str]: ...    # sees --f v, --f=v, -f v, repeated
    def set(self, flag: str, value) -> ArgV: ...  # REPLACES, never appends
    def unset(self, flag: str) -> ArgV: ...
    def semantic(self, d: EngineDialect, name: str, value) -> ArgV: ...

class EngineDialect(Protocol):
    def process_name(self, p: Process) -> Fact[str]: ...
    def flag(self, semantic: str, value) -> tuple[str, ...]:
        """context_length=4096 -> --max-model-len 4096 | --context-length 4096 | --max-seq-len 4096"""
    def preset(self, family: str, name: str) -> Mapping[str, Any]: ...
    def presets(self, family: str) -> tuple[str, ...]: ...
```

Three rules make `ArgV` satisfy REQ 5:

1. **`SHELL`, not `SHELL_LC`.** `-lc` is 100 of the 184 shell-invoked containers (§M4). Naming the
   majority form as an exception is how the current predicate came to test `cmd[-1] != "-c"`.
2. **Reads tokenise; writes splice `raw`.** `set()` and `unset()` edit the original string at the
   byte offsets in `spans` and re-emit it, so control operators, `#` comments and backslash
   continuations survive untouched. Tokenise → mutate → re-join is measurably lossy (§M4) and is
   therefore not the write path.
3. **`UNPARSEABLE` is a state, not an empty list.** When the tokeniser fails, `tokens == ()`, every
   `get()` returns `Fact(status=UNKNOWN)` rather than `ABSENT`, and every `set()`/`unset()` raises
   `Unrepresentable` naming the file and the container. Returning an empty token tuple would
   reproduce exactly the false-green the `-lc` defect produces today.

`EngineDialect.preset` is why `restart(router_mode="kv")` is rejected as an API: named router
configurations differ by several environment variables and a worker argument, and one variant exists
*because* another cannot surface the defect it was written for. Presets are named, validated and
enumerable.

## I2 Capability protocols — composition, not inheritance

Seven narrow protocols, each implementable per platform in an afternoon, so a new platform is seven
small classes rather than a fork of a four-thousand-line module.

| Protocol | Vantage / role | Surface |
|---|---|---|
| `Reach` | this process | `url(sel)`, `open(sel)` as a context manager, `health()`, and a **budget** |
| `Ingress` | a workload inside the SUT network | `url(sel)`, `scrape_targets(role)`, `placement()` |
| `Sink` | the SUT reaching a test-owned service | `publish(blob, reachable_from=sel)`, `listener(kind, reachable_from=sel)` |
| `Control` | lifecycle and faults | `supports(action) -> Fact[bool]`, `perform(action, ctx) -> Handle`, `apply(desired, strategy)`, `roles()` |
| `Observe` | reading | `logs(sel, since=, previous=, structured=)`, `metrics(sel, name=, labels=)`, `inventory(role)`, `exec_in(sel, argv)`, `processes(sel)` |
| `Generate` | load | `vantage` = CALLER (binds `Reach`) or WORKLOAD (binds `Ingress`); `start`/`stop`/`await`/`harvest` |
| `Recorder` | the bundle writer | `open(kind, key, producer=)`, `note(handle)`, `declare(kind, key, producer=, promise=)`, `seal()` |

Several separately-reported defects are each one `Observe` parameter **[a]**: `structured=True`
returns parsed records rather than a file (the tracing pretty format wraps fields in ANSI escapes, so
substring matching on a field value silently fails); `previous=True` is a **named parameter** rather
than one swallowed by `**kwargs`, so previous-container logs are absent rather than wrong;
`metrics(..., labels=)` does label matching and summation rather than substring-matching the first
line, which against a multi-replica deployment reads one replica's counter and asserts it equals the
total.

`Recorder` is not optional for any class. It is how the producer index becomes mandatory rather than
aspirational.

## I3 The three receivers

```python
plan = Plan.from_manifest(recipe_path).with_image(site.image)             # PLAN
with bring_up(plan, site, name="sanity") as sut:                          # ACT
    sut.wait_serving(timeout=1200)
    h = sut.load.start(SANITY, name="sanity")
    h = sut.load.await_(h, timeout=600)      # returns a NEW, completed Handle
                                                                          # COLLECT + SEAL: automatic
ev = sut.evidence()                                                       # CHECK
judge(ev, reports=[report_per_worker_latency()],
          checks=[require_valid_load(of=h),
                  expect_min_requests(of=h, n=10),
                  expect_zero_errors(of=h)])
```

`Handle` is frozen (§I4), so a verb that completes a running handle **returns a new one**; the
timeline records both records and the second supersedes the first. Checks address handles by their
`id` string — `of=h` is sugar for `of=h.id`, resolved through the producer index — because REQ 10
requires checks to be serialisable values and CHECK runs in a separate process that cannot hold a
live object.

`Sut` is synchronous and every verb takes a `Sel` at the call site or a declared default:

| Group | Verbs |
|---|---|
| reach | `url`, `address`, `channel`, `client` |
| inference | `query`, `stream`, `chat`, `probe`, `models` |
| direct-to-component | `admin`, `metrics`, `logs`, `exec_in`, `files`, `which_worker_served` |
| waiting | `wait`, `wait_ready`, `wait_serving`, `wait_log`, `wait_stable` — **one deadline threaded through all of them** |
| lifecycle | `start`, `stop`, `restart`, `scale`, `reconfigure` (live, no restart), `rolling_replace` |
| faults | `kill_process`, `stall_process`, `delete_replica`, `partition`, `reset_connections`, `restart_infra`, `inject`, `arm` |
| sinks | `publish`, `listener` |
| producers | `capture`, `monitor` (an agent inside the SUT), `snapshot` |
| in-timeline assertions | `require_restarted`, `require_replaced`, `require_now` |
| load | `sut.load.start / stop / await_ / apply` — the one stateful sub-namespace |

`wait_ready` (the resource or process is up) is deliberately **not** `wait_serving` (the model
answers), per REQ 6. There is one verb per *kind* of fault, never a `granularity=` keyword.

**Sync façade over async providers is an adjudication, not a preference.** The 56 subprocess files
and the local corpus are synchronous **[m]**; the Kubernetes runtime and its event bodies are async;
all checks are synchronous **[p: zero `async def` in the peer harness's checks package]**. Converting
either direction is thousands of lines of churn for zero coverage. The runner owns a loop in a
dedicated thread, and an `sut.aio` view is generated mechanically from the same registry for callers
already inside a loop.

The price is real and is priced. A blocking call made from a worker thread is not interruptible by
`KeyboardInterrupt`, and a non-daemon thread blocks interpreter exit; the design analysis reports a
long unkillable wedge from exactly this shape **[a — the duration is not re-verified and is omitted]**.
The structural hazard needs no measurement, which is why the interpreter-exit escape in REQ 18 is a
day-one dependency of the loop thread rather than a hardening item, and why the loop thread lands in
its own phase (5b) rather than alongside the first test migration.

### Phase guard

```python
# after the with-block exits
sut.kill_process(at("worker"))
# PhaseError: kill_process() is an ACT verb; this Sut left ACT at 12:03:41Z.
#             Assert over Evidence instead: expect_restart_count(at('worker'), since=h)
```

The guard is small, and the error names the phase and the replacement instead of raising
`AttributeError: 'NoneType' object has no attribute 'get_pods'`.

## I4 Handles and proof

```python
@dataclass(frozen=True)
class Handle(Generic[T]):
    id: str; verb: str; sel: Sel | None; args: Mapping[str, JsonValue]; name: str | None
    started_at: datetime; ended_at: datetime | None
    outcome: Literal["done", "running", "failed"]
    resolved: Mapping[str, Any]          # what the Policy ACTUALLY selected
    value: T | None
    proof: Proof | None
    artifacts: tuple[ArtifactRef, ...]
    error: str | None
    def window(self, *, before=0.0, after=0.0) -> Window: ...

@dataclass(frozen=True)
class Proof:
    kind: str                 # "proc_state_T" | "restart_count" | "pod_replaced" | "uid_diff"
    verified: str             # "/proc/4711/stat state S->T->S"
    before: Mapping[str, Any]; after: Mapping[str, Any]
    landed: Landed            # PROVEN | UNPROVEN | REFUSED
```

Three consequences:

1. **Windowed assertions become expressible at all.** `expect_requests_failed_in(window=stall.window())`
   takes a handle rather than scanning a timeline for a string-matched event name. Many existing
   checks take a window or anchor parameter, and several anchor by name on an unvalidated,
   non-unique string field **[a — no count is claimed]**.
2. **`resolved` is recorded**, so a check can know which replica a random or "hottest" policy actually
   selected.
3. **Proof is mandatory at registry import**, not at the call site:
   `@verb("act", "stall_process", proves=("pids", "state_before", "state_after"))`. The registry
   validator raises `MissingProofContract` at import for any fault-class verb with an empty contract.
   An optional `prove=` kwarg is the wrong shape: omitting it yields `proof=None`, which is not
   `landed is UNPROVEN`, so the run never enters the unproven set and every dependent check runs on
   evidence from a fault that may never have landed.

## I5 The naming law — gating policy is the verb prefix

| Prefix | Receiver | Meaning | Contributes to |
|---|---|---|---|
| `expect_*` | CHECK | must hold | the system verdict |
| `observe_*` | CHECK | measured on purpose | `OBSERVED`; **never** gates |
| `require_valid_*` | CHECK | is this measurement admissible? | run validity, not the system |
| `report_*` | CHECK | produces an artifact | nothing; runs first; never gates |
| `require_*` | ACT | must hold **now**; raises now; is recorded | in-timeline assertion |
| everything else | ACT / PLAN | does a thing | the timeline |

The prefix set is **not prefix-free** — `require_` is a proper prefix of `require_valid_` — so the
matcher is specified as **longest match wins**. `require_valid_load` is a CHECK verb; `require_now`
is an ACT verb. The registry validator enforces the same rule, so the classification a reader
performs and the classification the validator performs are the same function.

Registry validators, run at import and in CI:

| Rule | Failure |
|---|---|
| a verb name exists on at most one receiver | `DuplicateVerb` |
| a CHECK verb matches one of the four prefixes by longest match, or is a pure reader (`series`, `logs`, `timeline`, `metrics`, `spans`, `requests`) | `NamingLaw` |
| a fault-class ACT verb declares a non-empty proof contract | `MissingProofContract` |
| every verb signature is JSON-expressible | `NotSerialisable` |
| every marker name `Requirements` can emit is registered in `pyproject.toml` | `UnregisteredMarker` |
| the harness imports no `dynamo.*`; only the plugin registers pytest hooks; the manifest and evidence subtrees import no cluster client | `ImportBoundary` |

An observe-don't-judge mode becomes a **different verb**, not a keyword argument. Checks with such a
mode currently report `PASSED`; under the naming law they report `OBSERVED` and cannot be mistaken
for a gate. Existing check names survive as thin aliases over a small set of primitive readers, so a
port stays a port rather than becoming a rewrite.

The accepted type of the `of=` parameter is declared once and is the same on every check:

```python
Of = Handle | str | tuple[str, ...] | ProducerQuery
#     a handle | its id or a producer name | an explicit tuple | a typed producer-index query
```

`ProducerQuery(kind="load")` is how a check addresses *every* producer of a kind. It is resolved
against the producer index at CHECK time and the resolution is recorded, so it satisfies REQ 9's
"no globbing": there is no filesystem pattern and the answer is auditable. A string containing `*`
is rejected at PLAN time.

## I6 The evidence bundle, v1

```
<out_root>/<run_id>/
  sealed/                 # everything below this line is written before SEAL and is IMMUTABLE
    run.json              #   schema version, harness version, plan digest, site, RoleTable,
                          #   ownership, grants, git/image provenance, seal
    resolved.yaml         #   the fully-interpolated plan — the RE-RUNNABLE input
    timeline.jsonl        #   one record per Handle: verb, sel, args, brackets, resolved, proof
    producers.json        #   artifact -> {handle_id, producer, tool, tool_version}
    declared.json         #   artifacts AND row-count promises made at declare time
    requests/<name>.jsonl #   one record per request (query | probe | load) — ONE failure definition
    series/<name>.jsonl   #   (ts_ns, source, labels, value)
    roles/<log_key>/<replica>/{current.log, previous.log, metrics/*.prom, inventory.json, processes.tsv}
    load/<name>/          #   workload.json, records.jsonl, server_metrics.jsonl, summary.json
    probes/<name>/        #   per-response worker attribution
    faults/<handle_id>/proof.json
    exec/<handle_id>.{out,err,rc}
    monitor/<log_key>/<replica>/samples.jsonl
    system/applied/*.yaml #   EVERY document applied
    raw/<tool>/…          #   verbatim third-party output, QUARANTINED
  verdict/                # written AFTER seal, versioned separately, rewritable by `judge`
    findings.json         #   CHECK output, so a run can be re-scored and diffed
    outcome.json          #   the RunOutcome and its precedence trace
    derived/<name>.{json,md,csv,svg}
```

Five deliberate properties: **one** canonical timeline, not three overlapping serialisations of the
same events; `raw/` quarantined and `requests/` normalised so there is one definition of "did this
request fail"; the producer index mandatory and **no check ever globs**, which is what makes a
check's read-set verifiable in PLAN rather than by an assertion hours later; promises carried
alongside names; and **the seal boundary is a directory boundary**, so "immutable after SEAL" is
enforceable by a permission bit and re-judging is not a mutation.

### Seal — declared promise versus measured delivery

```python
@dataclass(frozen=True)
class Promise:
    kind: EvidenceKind; key: str
    min_rows: int | None = None            # "the file exists and is empty" is a delta
    window: tuple[datetime, datetime] | None = None
    producer: str = ""

@dataclass(frozen=True)
class Seal:
    schema_version: str
    complete: bool
    missing: tuple[str, ...]               # promised, never produced
    short: tuple[ShortFall, ...]           # promised N rows, delivered M < N
    producer_errors: tuple[str, ...]
    unproven: tuple[str, ...]              # handles whose proof did not land
    unresolved: tuple[Role, ...]           # roles the RoleTable could not bind
    def admits(self, c: Check) -> bool: ...
```

`min_rows` catches the hardest false-green class: *the producer ran and produced nothing, or less
than it promised.* It correctly distinguishes "shed 99% of requests but ran its full window"
(`complete=True`; the SLO check may fail honestly) from "collapsed" (`complete=False`; no system
verdict is emitted). A wrong-but-valid role binding is then caught twice, independently: `unresolved`
(the binding was never proven) and `short` (the log producer promised at least one row and delivered
zero).

Per REQ 11 the **seal is the authority**. `Seal.admits` is evaluated first; a `require_valid_*` check
may add producer-specific criteria and may downgrade, but may not upgrade. If a `require_valid_*`
returns `PASS` for a producer the seal listed in `missing` or `short`, the run is still inadmissible
and the disagreement is recorded as a finding against the check.

## I7 Verdicts, precedence, exit codes

```python
class Verdict(str, Enum):
    PASS = "pass"; FAIL = "fail"; OBSERVED = "observed"; UNKNOWN = "unknown"

class RunOutcome(str, Enum):
    PASSED = "passed"; FAILED = "failed"; DATA = "data"
    INADMISSIBLE = "inadmissible"; HARNESS_ERROR = "harness_error"
```

Precedence, total: `HARNESS_ERROR > INADMISSIBLE > FAILED > DATA > PASSED`. A gate finding of
`UNKNOWN` yields `INADMISSIBLE` unless the check registered `unknown_is="observe"`, which is itself
recorded in the bundle. A check that *raises* becomes `UNKNOWN` plus a harness error — **never**
`FAIL`.

| `RunOutcome` | pytest | CI exit | may a suite's `expect: fail` invert it? |
|---|---|---|---|
| `PASSED` | pass | 0 | yes (→ FAILED) |
| `DATA` | pass + recorded property | 0 | no |
| `FAILED` | fail | 1 | **yes — and only this one** |
| `INADMISSIBLE` | fail (`RunInadmissible`) + recorded property | 1 | **no** |
| `HARNESS_ERROR` | error | 1 | **no** |

Inverting only `FAILED` is the structural fix for the reported suite defect where a case that died in
argument parsing was scored `PASS` because it carried `expect: fail` **[a — the incident is not
re-verified; the rule stands on the type distinction alone]**. The second half of that fix is in
PLAN: override keys are validated against declared parameters on **every** path, not only the dry-run
path.

There is no fourth pytest outcome and this design does not pretend otherwise. `INADMISSIBLE` is a
failed pytest test with a distinct exception type plus a recorded property, so a suite runner can
classify it as a harness problem rather than a product regression.

## I8 Multi-system runs

A minimal comparison layer is **in scope**, because three real requirements have no other home: a
native-engine arm beside a Dynamo arm, an accuracy A/B, and suite negative controls.

```python
@dataclass(frozen=True)
class Arm:
    name: str; plan: Plan; site: Site | None = None
    expect: Literal["pass", "fail"] | None = None

def run_arms(arms: Sequence[Arm], body, *, root: Path) -> Comparison: ...
def judge_comparison(c: Comparison, *checks: CompareCheck) -> Sequence[Finding]: ...
```

N sessions, N bundles, one comparison bundle referencing them, sealed like any other, with
`judge_comparison` as a CHECK-phase pure function — so an A/B re-scores offline exactly like a single
run. A non-Dynamo arm is a `Plan` whose base is `Reference`: a provider that exposes `Reach` and
reports every control action as absent. Comparison checks obey the same naming law: a gating
comparison is `expect_better(...)`, not `compare_better(...)`.

## I9 Requirements → markers

```python
@dataclass(frozen=True)
class Requirements:
    # PROJECTS: emitted as a real pytest marker at DECORATION time
    lifecycle: str | None; gpus: int | None; accelerator: str; vram: Vram | None
    backend: str | None; platform: str | None; modality: str | None
    lane: tuple[str, ...]; area: tuple[str, ...]
    test_class: TestClass | None; deadline_s: int | None      # carried, NEVER synthesised
    # RUNTIME: attributed skip from the fixture
    config: Need[ConfigReq] | None; topology: Need[TopologyReq] | None
    imports: tuple[Need[str], ...]; capability: tuple[Need[str], ...]
    # COLLECTION: deselect / precondition
    source_tree: Need[bool]; image_kind: Need[str] | None; tooling: tuple[Need[str], ...]
    # CONSTRUCTION: refused at bring-up
    grants: Grants

    @property
    def marks(self) -> tuple[pytest.MarkDecorator, ...]: ...
    @classmethod
    def of(cls, item) -> Requirements: ...
```

Every runtime and collection field is a `Need`, so REQ 12's requirement/precondition polarity is
carried by the value rather than by a comment.

| field | marker emitted (exactly) | consumed by |
|---|---|---|
| `vram=Vram(profiled_gib=3.8)` | `profiled_vram_gib(3.8)` — same name, single positional, **float identity preserved** | `tests/conftest.py:693-712` deselect; `write_test_meta`; `shared-test.yml:251` lane routing |
| `vram=Vram.SEQUENTIAL` | *(no vram marker — explicitly)* | the `and not profiled_vram_gib` sequential lane |
| `vram=Vram(..., vllm_kv_cache_bytes=N)` | `requested_vllm_kv_cache_bytes(N)` — int, no rounding | the engine memory-argument path |
| `vram=Vram(..., sglang_kv_tokens=N \| trtllm_kv_tokens=N \| sglang_vram_gib=N \| trtllm_vram_gib=N)` | markers of the same name | per-backend engine overrides |
| `gpus=N, accelerator="gpu"` (default) | `gpu_0` / `gpu_1` / `gpu_2` / `gpu_4` / `gpu_8` | CI `-m`; Hardware category |
| `gpus=N, accelerator="xpu"` | `xpu_1` / `xpu_2` | CI `-m`; Hardware category |
| `lifecycle="pre_merge"` | `pre_merge` | CI `-m`; Lifecycle category |
| `backend="vllm"` | `vllm` | image selection and the import-based auto-skip |
| `platform="k8s"\|"h100"` | markers of the same name | CI `-m`; Hardware category |
| `test_class` | `e2e` (functional, topology) · `unit` (library_unit) · `integration` (image_probe, artifact) · `benchmark` (experiment) · *no marker at all* (infra_helper — not collected as a test) | Test Type category |
| `modality="image"` | `multimodal` | CI `-m` |
| `lane=(...)`, `area=(...)` | the marker of the same name, verbatim | open passthroughs |
| `deadline_s=1800` | `timeout(1800)` | pytest-timeout; also read by `write_test_meta` (`vram_utils.py:171-173`) |
| `imports`, `capability`, `config`, `topology` | *no marker* | runtime three-valued evaluation |

There is **no `platform="xpu"`**. `pyproject.toml` at `0765d30ad8` registers `xpu_1` and `xpu_2` and
no bare `xpu` **[m]**, and `--strict-markers` is in `addopts`, so `pytest.mark.xpu` would be a
collection error in every lane rather than a skip. `xpu_1`/`xpu_2` are also count-bearing and live in
the `Hardware` category alongside `gpu_0…gpu_8` (`tests/marker_categories.py` **[m]**), so they
belong to the accelerator-count field, not to a platform string. This is what the `accelerator=`
field exists for.

Four rules keep this honest.

1. **Every field is bucketed** (PROJECTS / RUNTIME / COLLECTION / CONSTRUCTION). A field without a
   bucket is a collection error. This is what stops a new requirement from quietly not reaching the
   scheduler.
2. **Every emitted marker name is validated at import** against the registered list, not only `lane`
   and `area`. `pyproject.toml` registers **68 markers** **[m]**, several of which CI selects on and
   no closed enumeration would contain; a closed enumeration would strand them, and an unvalidated
   open one produces the `xpu` failure above.
3. **`timeout` is carried, never synthesised.** Generating it would hide cases where a test's timeout
   is set *below* its own readiness budget **[a]** — a defect the single threaded deadline is meant
   to surface, not paper over.
4. **Marks are applied at decoration and parametrization time, and the harness registers no
   marker-mutating collection hook** (REQ 13). The only collection hook the plugin registers is a
   `tryfirst` **read-only validator** that raises `UsageError` when a declaration conflicts with an
   authored marker, or when an emitted name is unregistered. It never calls `add_marker` or
   `pytest_deselected`.

### The parity gate

```bash
# 1. Recover every -m expression CI actually uses, including the sequential-lane transform.
#    37 distinct *_test_markers / pytest_marks values exist across .github/workflows/*.yml [m],
#    several still template-interpolated; the script resolves them.
python scripts/ci_marker_expressions.py .github/workflows/*.yml > exprs.txt

# 2. Base is an explicit worktree at the merge base. `git stash` on a clean CI tree is a NO-OP.
git worktree add "$BASE_DIR" "$(git merge-base HEAD origin/main)"

# 3. Collect the way CI collects. NOT --collect-only (REQ 13). Distinct TMPDIR per run,
#    or the base run overwrites head's write_test_meta file and the gate diffs it against itself.
while read -r EXPR; do
  for LIMIT in $(python scripts/vram_ladder.py); do
    TMPDIR="$RUN/head" pytest -m "$EXPR" --max-vram-gib="$LIMIT" --dry-run \
        --marker-dump="$RUN/head.json"
    (cd "$BASE_DIR" && TMPDIR="$RUN/base" pytest -m "$EXPR" --max-vram-gib="$LIMIT" --dry-run \
        --marker-dump="$RUN/base.json")
    python scripts/marker_parity.py "$RUN/base.json" "$RUN/head.json" || exit 1
  done
done < exprs.txt
```

`--marker-dump` is new; `--dry-run` is not (REQ 13). The dump emits, per item: node id, the sorted
`(marker_name, marker_args)` tuple set, the selected/deselected state, and the `write_test_meta`
record. It **must** assemble that from a `tryfirst` snapshot of the full collected list plus every
`pytest_deselected` callback plus the meta file, because `tests/conftest.py:761` clears `items` under
`--dry-run` — a dump taken at `pytest_collection_finish` sees zero items, and a `tryfirst` dump alone
sees the pre-deselection list, which is precisely not the deselection. It must also tolerate three
`pytest_collection_modifyitems` implementations (`tests/conftest.py:661` `trylast`,
`tests/fault_tolerance/deploy/conftest.py:65` unordered, and the plugin's own `tryfirst` validator).

The diff must be **empty in the loss direction and reviewed in the gain direction**; a superset test
alone never fails when markers grow, which is the direction the dangerous failure moves. The baseline
is keyed by `(image_tag, arch, marker_expr)` because collection is image-dependent: `pyproject.toml`
carries **12 `--ignore-glob` entries** **[m]**, plus per-package collection hooks, module-level skips,
and directories absent from some runtime images **[a]**.

The VRAM ladder is not hard-coded here. `scripts/vram_ladder.py` emits one limit per distinct GPU
VRAM envelope present in the runner fleet, plus one below the smallest declared `profiled_vram_gib`
value and one above the largest, so both boundary directions are exercised. Deriving it from the
fleet inventory rather than from a literal list is a phase-2 deliverable.

**Cost, and where the matrix runs.** The full cross-product (37 expressions × the ladder × each
`(image_tag, arch)` pair) is a nightly job, not a per-PR one; each collection imports the whole test
tree inside a backend image and the per-collection wall time is unmeasured today. Phase 2's gate
requires that time to be **measured and published**, and the per-PR contract to be set from it: the
per-PR gate runs the expressions of the pipelines the PR's changed paths can reach, at one limit per
lane. The nightly full matrix is the backstop.

### The collection-integrity guard

Landed in the same PR as the gate, or the gate certifies today's blindness as the baseline. The
guard as revision 1 stated it — "no `--ignore-glob` pattern may match any `test_*.py`" — is
**unimplementable**, and measuring it is what shows why **[m]**: `--ignore-glob=deploy/power-agent/tests/*`
(`pyproject.toml:228`) matches **16 files containing 319 test functions**, and the comment
immediately above it records the reason — power-agent "ships a standalone suite run in-image
(`deploy/power-agent` Dockerfile `test` stage + its own `pytest.ini`); its `tests/` package name
collides with the repo-root `tests/` package during repo-wide collection". `--ignore-glob=*model.py`
(`:218`) matches `components/src/dynamo/vllm/tests/multimodal_utils/test_vllm_model.py`, and the
block comment above `addopts` says these ignores exist "to avoid duplicate-module collection
errors". Enforcing the naive guard would newly collect over 320 tests **and**, by the repository's
own note, break collection outright.

The implementable guard is:

* **Re-anchor the two unanchored globs**: `*vllm_integration*` → the `lib/bindings/kvbm/python/kvbm/vllm_integration/`
  path it was written for, and `*trtllm_integration*` likewise. This is what makes the 5 KVBM tests
  appear.
* **No `--ignore-glob` may shadow a `test_*.py` outside an allowlist** of directories that ship their
  own `pytest.ini` and are run separately, each allowlist entry carrying a written rationale in the
  same file. `deploy/power-agent/tests/` is the first entry.
* **No import-skip at package `__init__` scope**, and a **per-tier minimum-collected-count** recorded
  in the bundle and compared in CI.

## I10 Packaging

| Decision | Value |
|---|---|
| Location | `ai-dynamo/dynamo`, directory `harness/`, its own `pyproject.toml` and version |
| Distribution / import root | `dynamo-test-harness` / `dynamo_test` |
| Consumed by `dynamo` | `pip install -e ./harness`; `tests/utils/*` become re-export shims |
| Consumed by the internal repo | a pinned git reference to the same subdirectory, graduating to a published package once the verb registry and bundle schema stabilise |
| Review routing | a `harness/` entry in the CODEOWNERS **source** (`.github/codeowners/areas.yaml`), never a hand edit of the generated file |

```
harness/dynamo_test/
  roles.py  argv.py  facts.py  engine_details.py  router_config.py  constants.py
  manifest/   schema.py  service_spec.py  plan.py       # tests/deploy loader + peer-harness mutators
  site.py  policies.py  grants.py
  evidence/   bundle.py  readers.py  seal.py  judge.py  # sync, pure, stdlib only
  checks/library/   reports/library/                    # sync, pure, stdlib only
  load/  workload.py  shapes.py  sweep.py  trace.py  inprocess.py  deployed.py
  reach/ reach.py  ingress.py  sink.py
  providers/  local/  k8s/  attached/  reference/  docker/
  sut.py  registry.py  arms.py
  requirements.py                                       # imports pytest ONLY for pytest.mark
  runner/  phases.py  signals.py  suite.py
  pytest_plugin.py                                      # the ONLY module registering pytest hooks
  cli/  plan  run  judge  replay  rerun  suite  list  verbs  stop
```

Three import rules, each with a test that walks the AST of every module:

1. **Nothing imports `dynamo.*`.** CI job: install the harness in a venv **without** the runtime and
   import every module **except `pytest_plugin.py`**, which by construction requires pytest and is
   covered by a second job in a venv that has pytest and not the runtime. Consequence, stated
   plainly: in-process PyO3 tests can never use the façade.
2. **Only `pytest_plugin.py` registers pytest hooks or fixtures; only it and `requirements.py`
   import pytest, and `requirements.py` may use no pytest name but `mark`** (REQ 17). Today
   `tests/deploy/dgd_utils.py` carries a single module-level `import pytest` at `:16` serving one
   `pytest.fail` at `:81` **[m]**, and that one line is what blocks the manifest model from being
   importable standalone.
3. **The manifest, evidence, checks and reports subtrees import stdlib plus a YAML parser and nothing
   else** — no cluster client, no HTTP client. This is what lets PLAN-time validation and offline
   judging run on a laptop, in CI's bare-checkout tier, and in a runner with no cluster credentials.
   Extras: `[k8s]`, `[local]`, `[load]`.

**The pre-commit marker hook is a deliverable, not a bystander.** The first
`from dynamo_test import requires` inside `tests/` runs in an isolated venv with 8 dependencies that
do not include the harness (§M2/R6), where `DependencyStubber` replaces the decorator with a stub
class and the decorated test **disappears from the report**. Phase 5a therefore lands the
`.pre-commit-config.yaml` edit that adds the harness as a local path dependency in the same PR as the
first migrated file. Note that pre-commit caches the hook environment keyed on the dependency list,
so a `./harness` path dependency does **not** reinstall when the harness changes; the phase must
either pin a version that is bumped, or add a `pre-commit clean` step to the harness's own CI.

**The public/internal cut: share the schema and the engine; do not share the corpus.** Public and
jointly owned: the bundle schema, the receiver protocols, the generic check and report libraries, the
plan/argv/dialect model, the site *schema*, the reach protocols, the providers, the pytest plugin, the
verb registry. Internal: site profile *instances*, scenario documents naming internal workloads,
suites carrying incident provenance, and any check whose *name* references an incident — the
mechanism moves out under a generic name; the incident identifier stays internal. Internal-only verbs
live in a reserved `x_` namespace, so an internal capability never blocks on a public review and
never silently becomes public API.

**One blocking unknown**: whether the internal CI can install from `github.com`. It must be answered
before the first step lands, because that step is where the internal repository starts depending on
the package. Fallback ladder: internal package mirror → vendored copy with a scheduled re-sync test.
Vendoring guarantees exactly the drift already measured between the two Kubernetes implementations
(§M2/R5), so it is a stopgap with an expiry date, not an option.

## I11 How each class is authored

**TC1 functional** — the portability target. The body is the existing assertion function; the
fixture's construction is decided by `--site`.

```python
@requires(lifecycle="pre_merge", gpus=1, vram=Vram(profiled_gib=24.0),
          backend="vllm", capability=(Need("tool_calling"),))
@testclass(FUNCTIONAL)
def test_tool_call_is_actually_executed(sut: Sut, model: str):
    assert_executes_real_tool_and_uses_output(sut.client(), model)   # tests/utils/, UNCHANGED
```

The **enforcement** is the construction-time grant check: `hasattr(sut, "kill_process")` is false for
a functional item, and `Evidence` grants only request records and the timeline, so reading worker
logs raises rather than quietly succeeding because the file happens to be there. The AST lint is
advisory only (REQ 15) — this very example delegates its whole body to a helper in `tests/utils/`
that no AST walk over the test body can see through.

Fleet depth is a **verb**, not a noun — there is no `Fleet` class:

```python
p = sut.probe("What is the capital of France?", n=12)   # unique prefix per request
judge(sut.evidence(), checks=[
    expect_workers_seen(of=p, at_least=2),
    expect_not_degenerate(of=p, max_share=0.8),         # one worker took >80%? affinity masked it
    expect_all_answered(of=p, contains="paris")])
```

Attribution is sourced from routing telemetry, not log scraping, retiring the most brittle mechanism
in the fault-tolerance area (backend-specific request-id log literals) **[a]**.

**TC2 config / topology** — arms, presets, semantic settings, no shared mutable spec:

```python
arms = [Arm("round_robin", plan.with_preset("router", "round_robin")),
        Arm("kv",          plan.with_preset("router", "kv_approx"))]
c = run_arms(arms, body, root=site.out_root)
judge_comparison(c, require_valid_load(of=ProducerQuery(kind="load")),
                    expect_series_above("dynamo_kv_cache_hit_rate", arm="kv", bound=0.25),
                    expect_better(metric="p99_ttft_ms", better="kv", worse="round_robin", by=0.10))
```

Note what is absent: no free-form `restart(router_mode="kv")`, no backend-locked flag
(`plan.set(at("decode"), context_length=4096)` renders per engine), and no per-arm mutation of a
shared spec object. `expect_better` carries a naming-law prefix (§I5); a name like `compare_better`
would raise `NamingLaw` at import.

**TC3 deployment artifact** — no `Sut` at all; PLAN and CHECK only. The cheapest large win in the
proposal: it runs on a bare checkout with no GPU, no cluster and no image.

```python
def _recipe_manifests() -> list[Path]:
    # Resolved from the harness's own installed location, NOT from the process CWD: a
    # Path("recipes") literal is evaluated at collection import, before source_tree is
    # ever checked, so from any other directory the parametrize list is silently empty
    # and the test reports a non-failure -- the same vacuous-pass shape M5 indicts.
    # ARC containers run from $GITHUB_WORKSPACE, not a fixed path, so CWD does vary here.
    return sorted(p for p in (repo_root() / "recipes").rglob("*.yaml") if Plan.looks_like_dgd(p))

@requires(lifecycle="pre_merge", gpus=0, source_tree=Need(True, Polarity.PRECONDITION))
@testclass(ARTIFACT)
@pytest.mark.parametrize("path", _recipe_manifests(), ids=str)
def test_recipe_bundle_is_self_contained(path: Path):
    ev = Evidence.of_plan(Plan.from_manifest(path))     # a PLAN-only bundle: no SUT, no ACT
    judge(ev, checks=[expect_documents_parsed(),
                      expect_configmaps_shipped(),      # mounts no ConfigMap it does not ship
                      expect_every_role_imaged(),
                      expect_argv_never_unparseable()]) # ArgV.form is never UNPARSEABLE
```

Two details that are load-bearing. `rglob("*.yaml")` filtered by `looks_like_dgd` reaches the **178**
`recipes/` DGD manifests **[m]**; `rglob("deploy.yaml")` would reach **136** **[m]**, missing all 25
`patch-dgd.yaml` files and both template families. And `source_tree` is a **precondition** (unmet ⇒
error, not skip), which fixes a whole directory module-skipping when its inputs are absent from the
image and reporting a quiet green.

**TC4 library unit / ABI pin** — deliberately outside the façade. No `Sut`, no reach, no bundle, and
these items never enter PLAN, so REQ 14's weak-verdict rule does not apply to them. The framework's
entire contribution is collection integrity, which is exactly what these tests needed and never got
(§M5).

**TC5 image / wheel probe** — `Plan.empty()` yields only the `self` role; `exec_in` writes stdout,
stderr and exit code into the bundle, so the assertion becomes a pure function over an artifact and,
more usefully, **diffable across images**. The vacuity guard is a first-class check rather than a
bare `assert`, because `image_probe` is a gating class (REQ 14):

```python
r = sut.exec_in(at("self"), ["python", "-c", "import cv2"])
f = sut.files(at("self"), "**/libavcodec.so*")
judge(sut.evidence(), checks=[
    expect_exec_failed(of=r, why="opencv must be absent from this image"),
    expect_files_present(of=f, why="the media stack is present, so the absence check above "
                                   "is measuring something rather than an empty directory")])
```

**TC6 benchmark / experiment** — `DATA` is an outcome; gating is optional. This is the only class
where an empty gate list is legal, and declaring the class is the opt-out from the weak-verdict
error.

```python
loads = ProducerQuery(kind="load")          # a producer-index query, not a glob (REQ 9)
judge(ev, reports=[report_sla_frontier(), report_per_worker_latency(), report_event_timeline()],
          checks=[require_valid_load(of=loads),                       # admissibility ONLY
                  require_valid_shape(of=loads, tolerance=0.1),
                  observe_goodput(of=loads, slo="ttft_p99:500")])     # measured, never gates
```

This is where the accuracy A/B currently sitting under `tests/` finally has a home: as an experiment
with two arms it becomes a real measurement with a real bundle instead of a printed line (§M5).

## Deferred to Implementation

* The concrete capability enumeration and per-backend fact extractors. The rule is fixed — a small
  number of extractors, each with a named source, each returning a three-valued fact whose **default
  is `UNKNOWN`** — but the set is not.
* Whether the standalone router warrants its own role binding or is only ever observable through the
  frontend.
* Whether `UNKNOWN` fails CI from day one or warns for one release. The published count and the
  triaged flip list come first either way.
* Compose and container-per-component topology definitions, if and when a test needs them.
* Whether comparison arms ever need to run *concurrently* rather than sequentially. If they do,
  `run_arms` grows a scheduler and §I8 is wrong.
* Cross-repository bundle-schema compatibility beyond "semver, forward-tolerant readers, a
  conformance suite shipped in the harness and run by both repositories, and the schema frozen for
  the duration of the migration".
* The share of `tests/` that is `infra_helper` rather than test. It is believed to be large, and the
  category exists regardless of the number; a measured share is a phase-2 by-product of the path map,
  not an input to this proposal.

# Implementation Phases

Each phase lands alone, is independently revertable, and carries a **falsifiable, mechanical gate**.
**Phases 1–4 change no test file.** The façade arrives in phase 5, not phase 1: a design whose first
deliverable is a façade cannot be ratcheted, because a façade's value only appears after N files
adopt it, and no file can adopt anything until the package exists, installs from both repositories,
and reproduces today's CI selection exactly.

Two notes on the per-phase fields:

* **Release Target** is stated as a position in the sequence rather than a date. This proposal fixes
  the *order*, which is the part that is argued from evidence; the calendar is set with the Sponsor
  at approval, and every phase's target is written as `TBD (sequence position N)` until then.
* **Effort Estimate** is tagged **[e]** — the author's estimate, not a measurement. The basis is
  stated per phase. The total across phases 1–12 is **55–76 engineer-weeks [e]**, excluding the
  per-area migration burn-down described at the end of this section.

## Phase 1 The Package, Seeded Only With Code That Is Already Portable

**Release Target**: TBD (sequence position 1)

**Effort Estimate**: 3–4 engineer-weeks, 1 engineer **[e]** — basis: the module list below is
existing code being moved, plus the registry and its six validators, which are new but small.

**Work Item(s):** TBD

**Dependencies:** none.

**Supported API / Behavior:**

* `harness/` with its own `pyproject.toml` and version; `pip install ./harness` in a venv with
  neither the runtime nor a cluster client nor pytest.
* Moved, already-portable modules: engine process tables, router presets, fabric facts, constants,
  output paths, the cluster interface.
* New values: the `Role` vocabulary, `Sel`/`at()`, `RoleBinding`/`RoleTable`, `Fact[T]`, `Need`/`Polarity`,
  `ArgV`/`ArgForm`/`EngineDialect`, the verb registry and its six validators, and `dynamo-test verbs`.
* A pre-commit rule forbidding **new** `https?://(localhost|127\.0\.0\.1|0\.0\.0\.0)` literals under
  `tests/*.py`. This is one grep, needs none of the rest of the harness, and stops the M1 pattern
  from growing for the five quarters the burn-down takes.

**Not Supported:**

* No provider, no `Sut`, no `Evidence`, no marker interaction. Nothing under `tests/` imports it.

> **Gate.** In a venv with neither the runtime nor a cluster client nor pytest:
> `pip install ./harness && python -c "import dynamo_test"` succeeds, and an AST walk confirms no
> module outside `pytest_plugin.py`/`requirements.py` imports pytest. The internal repository adds
> the pinned dependency **behind an import shim, keeping its own copies**, and its full scenario
> dry-run set is unchanged. **And: prove the internal CI can install from `github.com`.** If it
> cannot, the packaging decision changes, and it is better to know in week one than in month three.
>
> *Deleting the internal copies is a separate, later step, gated on the pinned reference having been
> green for one internal release cycle.* Revision 1 made the deletion part of this gate, which made
> phase 1 a two-sided cross-repository atomic landing and therefore not revertable.

*Why first:* nothing else can start until the package exists and both consumers can install it.

## Phase 2 The Marker Bridge, The Real Parity Gate, And Collection Integrity

**Release Target**: TBD (sequence position 2)

**Effort Estimate**: 4–6 engineer-weeks, 1 engineer **[e]** — basis: the bridge itself is small; the
gate matrix, its cost measurement and the triage of its first diff dominate.

**Work Item(s):** TBD

**Dependencies:** phase 1.

**Supported API / Behavior:**

* `Requirements.of(item)` (derives from **authored** markers; treats a category marker on a
  `defaulted` item as `UNKNOWN`), `@requires` applying real marks **at decoration time**,
  `Requirements.marks` usable inside `pytest.param(marks=…)` and appendable to a computed marks list.
* The three-way VRAM rule (profiled value → parallel lane; `SEQUENTIAL` sentinel → sequential lane;
  neither → import-time error).
* The `accelerator=` field emitting `gpu_N` / `xpu_N`, and import-time validation of **every**
  emitted marker name against the registered list.
* `--marker-dump`, `scripts/ci_marker_expressions.py`, `scripts/vram_ladder.py`,
  `scripts/marker_parity.py`.
* The collection-integrity guard of §I9, including re-anchoring `*vllm_integration*` and
  `*trtllm_integration*` and the `pytest.ini`-allowlist rule, **in the same PR**.

**Not Supported:**

* **Zero test files adopt the decorator.** No `pytest_collection_modifyitems` hook that mutates
  markers is added, now or ever (REQ 13).

> **Gate.** For every `-m` expression across all four pipelines × the derived VRAM ladder, the
> collected `(nodeid, marker_name, marker_args)` tuple set, the selected/deselected split, and the
> `write_test_meta` output are byte-identical before and after — **except** the 5 KVBM tests the
> re-anchored `--ignore-glob` newly collects **[m]**, which is the one reviewed diff. Head and base
> runs use distinct `TMPDIR` values. *(The gate is stated over the collected tuple set, not over a
> source-grep count: the cardinality of that set is not the 106 source applications, because
> `tests/utils/multimodal.py:612` emits one marker per topology-matrix case from a computed value and
> the `pytest.param` tables fan out further. A source-grep number inside a collection-time gate is
> exactly the category error the rest of this proposal argues against.)*
>
> **And: publish the measured per-collection wall time**, and set the per-PR subset from it. The full
> matrix runs nightly.

*Why second:* this is the launch blocker and the most likely way the work breaks CI. Landing the gate
before anything consumes it means the gate exists before the risk does.

## Phase 3 The Manifest Layer, Shell Form, And Apply-Every-Document

**Release Target**: TBD (sequence position 3)

**Effort Estimate**: 5–7 engineer-weeks, 1–2 engineers **[e]** — basis: the spec model is a verbatim
move; `ArgV`'s splice-on-write representation and the deprecation shims are new.

**Work Item(s):** TBD

**Dependencies:** phase 1. **Unlanded work this phase builds on**: PR #13979 (multi-document
`safe_load_all` at `dgd_utils.py:496`) and branch `dtokarev/dgd-shell-style-lc` (the widened
short-option-cluster predicate). Both must land first or be absorbed into this phase.

**Supported API / Behavior:**

* `dgd_utils.py`'s spec model moves into `harness/manifest/` verbatim; `tests/deploy/dgd_utils.py`
  keeps its name and becomes a re-export shim, so its importers do not change.
* The peer harness's mutators are grafted on.
* The module-level `import pytest` at `dgd_utils.py:16` is deleted.
* `ArgV` replaces the `cmd[-1] != "-c"` predicate: any short-option cluster ending in `c` is `SHELL`;
  `raw` is retained and writes splice it; an untokenisable script is `UNPARSEABLE`, not empty.
* Reads become non-mutating. `api_version` is carried in the shared core, which collapses several
  merge conflicts to nothing. Deprecation shims ship for the four collided names, and the pod view
  gains a real `previous=` parameter.
* The TC3 recipe lint of §I11 lands in the same PR.

**Not Supported:**

* No provider, no `Sut`, no `Evidence`. No test file changes except the new TC3 lint.

> **Gate.** Three numbers, because the phase is only meaningful as a delta:
> today's loader ceilings are **67 of 295** (peer-harness shape) and **162 of 295**
> (`tests/deploy` at `0765d30ad8`) **[m]**; PR #13979 alone reaches **278 of 295** **[m]**. The gate
> is therefore *not* "constructs 278" — that arrives with the dependency — but the delta the move
> does not automatically produce:
> 1. every one of the **100** `-lc` containers across **51** files flips from argv to `SHELL`, and a
>    write to each is byte-verified against the original script (operators, `#` comments and line
>    continuations unchanged) **[m]**;
> 2. every non-DGD document in a manifest reaches `system/applied/`, closing the **76 of 178** recipe
>    ConfigMap loss **[m]**;
> 3. `import pytest` no longer appears in the manifest subtree;
> 4. node-ID diff on `tests/deploy` is empty; `logs(previous=True)` no longer silently returns
>    current logs.
>
> The construction ceiling is stated as **278 of 295**, not 279: the 279th is
> `deploy/operator/config/samples/nvidia.com_v1alpha1_dynamographdeployment.yaml`, whose `spec:` is
> comment-only and parses to `None`, so it carries a DGD document with no roles to construct **[m]**.
> It is a named exception, not a residue.
>
> **Named risk, to be verified against the operator's Go source rather than reasoned about:** a read
> that materialises the container sub-document, combined with the operator's override-merge slice
> replacement, can silently clobber a mount declared at service level. Read the source before landing.

*Why third, and why it is the highest-value phase:* it is the difference between "run against any
shipped recipe" being a slogan and a command, for the price of two import lines and one value type,
and it unlocks the recipe corpus on a bare checkout with no GPU and no cluster.

## Phase 4 `Evidence` And `judge()`, As A Reader Of What Already Exists

**Release Target**: TBD (sequence position 4)

**Effort Estimate**: 6–8 engineer-weeks, 1–2 engineers **[e]** — basis: the bundle contract and
readers are new but bounded; the flip-list triage is the long pole and is a people problem.

**Work Item(s):** TBD

**Dependencies:** phase 1. Independent of phases 2 and 3.

**Supported API / Behavior:**

* The bundle contract of §I6 including the `sealed/` versus `verdict/` split, the producer index,
  promises, `Seal`, typed readers, the five ordering rules, the five-valued run outcome, and the
  naming law with its registry validator.
* `Evidence.open()` points at the peer harness's *existing* output directories through a
  compatibility sealer that synthesises the run document from the conventions already on disk.
* Existing checks and reports are repointed mechanically. Fixed on the way through: checks whose
  `description` is a method rather than a property (they render as a bound-method repr in the run log
  and the failure summary), the bare `assert`s inside checks (stripped by `PYTHONOPTIMIZE`, which
  would silently turn every check into a pass), the multiple definitions of "did this request fail",
  and the unanchored substring match in load-directory resolution.

**Not Supported:**

* **Execution does not change.** No provider, no `Sut`, no new run mode.

> **Gate.** `judge(Evidence.open(d), …)` reproduces byte-identical findings against archived runs,
> **except** where a check previously passed vacuously — those flip to `UNKNOWN`, and **the flip list
> is the deliverable**: it is the first measurement of how much of the existing verdict layer is
> reading nothing. The three unguarded-`ctx.deployment` checks (`RankProcessCount`, `CliffContained`,
> `PinningContained` **[p]**) **fail to compile** — `Evidence` has no deployment attribute — and are
> rewritten against `ev.pods()` / `ev.artifact(...)`.

Absence-implies-inadmissible and short-delivery ship behind `--strict-evidence` for one release; the
**count** is published from day one and the flip list is triaged before the flag flips. Otherwise a
correct change arrives as a wave of new red on a suite people believe is green.

## Phase 5a Local Provider, A Sync-Native `Sut`, And One Duplicated Pair Collapsed To One

**Release Target**: TBD (sequence position 5)

**Effort Estimate**: 6–8 engineer-weeks, 2 engineers **[e]** — basis: the provider wraps existing
synchronous code; the verb surface, grants and conformance suite are new.

**Work Item(s):** TBD

**Dependencies:** phases 1, 2 and 4. Phase 4 **must** land before this phase, and a revert of phase 4
after this phase lands is forbidden by the gate below (§P4). Also depends on
`tests/deploy/test_recipe_tool_execution.py` from PR #13979, which is the Kubernetes half of the
migrated pair.

**Supported API / Behavior:**

* `providers/local/` as `Plan.render(LOCAL)` → `ManagedProcess`, wrapping the existing readiness
  model verbatim. It is **synchronous**, so the `Sut` shipped here is sync-native and needs no loop
  thread; that is why the bridge is phase 5b.
* The `Sut` verb surface of §I3, the phase guard, grants, and mandatory fault proof.
* The role-scope fix in the `Control` conformance suite: `stop`/`kill` take a `Sel`, and a provider
  that cannot scope raises `Refused` rather than widening (§P4). The process-name kill sweep defaults
  to **off**.
* REQ 2's **Attached** provider (query-only; `Ingress` raises `Unreachable`) and **Reference**
  provider (exposes `Reach`, reports every control action as absent) ship here, since both are small
  and both are needed by the conformance suite.
* REQ 14's path map and the weak-verdict PLAN-time error land here, scoped to items that enter PLAN.
* The `.pre-commit-config.yaml` edit that makes the marker hook see the harness (§I10).
* `.ai/pytest-guidelines.md` drops its recommendation of `runtime_services_session`, a session fixture
  defined at `tests/conftest.py:1223`, recommended at `.ai/pytest-guidelines.md:469`, with **zero**
  consumers repo-wide **[m]**.
* Exactly one migration: the local-subprocess tool-calling tests and the Kubernetes recipe
  tool-execution test — the same assertions over two bring-ups — collapse onto one scenario function
  taking `(client, model)`.

**Not Supported:**

* No loop thread, no `sut.aio`, no async providers, no Kubernetes provider.

> **Gate.** One function, two `--site` values, both green. `-m pre_merge` marker-dump diff empty.
> `tests/utils/test_managed_process_teardown.py` **[m — the file exists]** becomes the
> backend-agnostic `Control` conformance suite and passes against the Local provider unmodified.
> Pointed at a Kubernetes site with no grants, the migrated test fails with a named
> `NotGranted`/`Unreachable`, not a timeout. **And: a CI check asserts that `harness/dynamo_test/sut.py`
> cannot be imported unless `harness/dynamo_test/evidence/` exists**, so a revert of phase 4 that
> leaves phase 5a in place fails to build rather than silently recreating the landmine §P4 describes.

*Why fifth:* it is the first phase that proves REQ 2 with the corpus's own evidence rather than with
a new test written to succeed, and choosing the already-duplicated pair makes the payoff a
**deletion**.

## Phase 5b The Sync-Over-Async Bridge

**Release Target**: TBD (sequence position 6; lands with phase 6)

**Effort Estimate**: 2–3 engineer-weeks, 1 engineer **[e]** — basis: the bridge is small; its hazard
handling is most of it.

**Work Item(s):** TBD

**Dependencies:** phase 5a. **Lands with phase 6**, which is the first phase that has an async
provider to bridge to.

**Supported API / Behavior:**

* The runner's dedicated loop thread and the mechanically generated `sut.aio` view.
* REQ 18's day-one long-run hazards, which are dependencies of the loop thread and not hardening:
  abandoned-run cleanup, a **signal-escalation ladder that does not depend on process-group
  delivery**, and an **interpreter-exit escape for hung non-daemon threads**.

**Not Supported:**

* Nothing async is exposed to test authors; `Sut` stays synchronous.

> **Gate.** A provider call deliberately wedged in the loop thread is escaped within a bounded,
> published timeout and the interpreter exits; the run leaves a judgeable, sealed bundle rather than
> hanging. `SIGINT` and `SIGTERM` each escalate through the full ladder against a child that ignores
> the first two signals and against one detached from the process group.

*Why split from 5a:* this is the design's self-declared riskiest mechanism, the Local provider does
not need it, and bundling it with the first migration would mean a wedge cannot be reverted without
also reverting the migration — which contradicts "each phase lands alone".

## Phase 6 Kubernetes Runtime As Opt-In Policies

**Release Target**: TBD (sequence position 7)

**Effort Estimate**: 10–14 engineer-weeks, 2 engineers **[e]** — the largest single merge; basis: the
capability set enumerated in REQ 16.

**Work Item(s):** TBD

**Dependencies:** phases 3, 5a, 5b.

**Supported API / Behavior:** the REQ 16 capability set behind `Policy` values that are **default
off**: namespace lifecycle including scrub (additionally requiring `namespace_ownership: dedicated`),
log-collection storage with in-pod capture, model prefetch, run provenance, signal handling with
emergency delete, per-pod metrics capture, in-pod exec transport, and `Reach` via port-forward with a
lifetime and a budget.

**Not Supported:** `Ingress`, load generation, `Sink`, suites.

> **Gate.** Every `Policy` defaults off and a scrub against a `namespace_ownership: shared` site
> raises rather than deleting. The `Control` conformance suite from 5a passes unmodified against the
> Kubernetes provider. One already-migrated functional test runs on Local and Kubernetes from the
> same function with no test edit.

## Phase 7 `Workload` / `Generate` With `Ingress`

**Release Target**: TBD (sequence position 8)

**Effort Estimate**: 5–7 engineer-weeks, 1–2 engineers **[e]**.

**Work Item(s):** TBD

**Dependencies:** phases 4, 6.

**Supported API / Behavior:** REQ 8's workload model, in-process caller first (`vantage=CALLER`,
binds `Reach`), deployed generator second (`vantage=WORKLOAD`, binds `Ingress`); open- and
closed-loop shapes, rate staircases, bursts, sequence-length mixtures, trace replay; third-party CLI
passthrough quarantined in one field.

**Not Supported:** `Sink`, suites, arms.

> **Gate.** The same workload declaration runs from both vantages against the same deployment and
> produces `load/<name>/` bundles whose `workload.json` is identical apart from vantage and
> placement. A generator whose placement the site cannot satisfy raises `Unreachable` at PLAN time.

## Phase 8 `Site` Plus The Bleed Check

**Release Target**: TBD (sequence position 9)

**Effort Estimate**: 2–3 engineer-weeks, 1 engineer **[e]**.

**Work Item(s):** TBD

**Dependencies:** phase 1.

**Supported API / Behavior:** REQ 18's environment profile as an opaque validated object; auth with a
remediation string; provisioning recipe; per-cluster workarounds.

**Not Supported:** shipping site *instances* in the public repository.

> **Gate.** A test-scoped key placed in a site profile, and a site-scoped key placed in a run, each
> raise a named error naming the key and both scopes. No site instance appears under `harness/`.

## Phase 9 The Scenario Document And Generated Registry

**Release Target**: TBD (sequence position 10)

**Effort Estimate**: 4–5 engineer-weeks, 1 engineer **[e]**.

**Work Item(s):** TBD

**Dependencies:** phases 4, 5a.

**Supported API / Behavior:** a test serialisable to a document, and a primitive registry
**generated** from the same definitions the imperative surface uses.

**Not Supported:** hand-maintained registry entries of any kind.

> **Gate.** Round-trip: every imperative test in the migrated set serialises to a document, the
> document runs, and the two bundles' `timeline.jsonl` are identical apart from run id and clock. A
> registry entry that is not generated fails a CI check.

## Phase 10 Arms And Suites

**Release Target**: TBD (sequence position 11)

**Effort Estimate**: 3–4 engineer-weeks, 1 engineer **[e]**.

**Work Item(s):** TBD

**Dependencies:** phases 4, 7.

**Supported API / Behavior:** §I8's `Arm`, `run_arms`, `judge_comparison`; suites with `expect: fail`
arms and the REQ 18 inversion rule.

**Not Supported:** concurrent arms (see *Deferred to Implementation*).

> **Gate.** An arm that dies in argument parsing is scored `HARNESS_ERROR` and is **not** inverted by
> `expect: fail`; an arm that is `INADMISSIBLE` is likewise not inverted; only a genuine `FAILED` is.
> Each is a distinct test in the suite-runner's own suite.

## Phase 11 `Sink`

**Release Target**: TBD (sequence position 12)

**Effort Estimate**: 2–3 engineer-weeks, 1 engineer **[e]**.

**Work Item(s):** TBD

**Dependencies:** phases 6, 8.

**Supported API / Behavior:** `publish(blob, reachable_from=sel)` and `listener(kind, reachable_from=sel)`
for media fixtures, an OTLP collector, an SSRF canary and a trace sink, reachable from inside the
SUT's network.

**Not Supported:** anything requiring inbound connectivity the site does not declare.

> **Gate.** A fixture published from the test process is fetched by a worker pod and the fetch
> appears in the bundle; on a site that cannot route inbound, `publish` raises `Unreachable` at PLAN
> time rather than timing out.

## Phase 12 The Declarative Deployment-Request Mode

**Release Target**: TBD (sequence position 13)

**Effort Estimate**: 3–4 engineer-weeks, 1 engineer **[e]**.

**Work Item(s):** TBD

**Dependencies:** phases 3, 6.

**Supported API / Behavior:** `Plan.request(model=, sla=)` as a third plan constructor, rendered to a
DGDR the operator synthesises.

**Not Supported:** the shared utility module this touches is a **separate merge with a separate
owner**, because it is a rewrite of the shared ancestor rather than a divergence from it.

> **Gate.** A `Plan.request(...)` and the DGD the operator synthesises from it round-trip through
> `Plan.from_manifest` to the same resolved plan, and every role binds by name.

## The Burn-Down

The hostable corpus is **89 files** **[m]** — the union of the three signatures a façade can host:

```bash
# tests/*.py at 0765d30ad8; union of the three sets = 89
git grep -lE 'ManagedProcess|DynamoFrontendProcess' 0765d30ad8 -- 'tests/*.py'   # 56
git grep -lE 'kubernetes|kr8s|kubectl'              0765d30ad8 -- 'tests/*.py'   # 18
git grep -lE 'examples/'                            0765d30ad8 -- 'tests/*.py'   # 18
```

Target: **20 files per quarter [e]**, ordered by area (fault-tolerance cancellation → frontend →
router → serve → deploy → mm_router → kvbm_integration), each area landing with its bespoke process
subclasses collapsed onto `Plan`, its share of the **78 localhost-building files** **[m]** converted
to `sut.url()`, and its log-pattern polling converted to structured `logs()`. At that rate "when does
`tests/` stop having two frameworks in it" has an answer — **roughly five quarters [e]** from the
start of phase 6 — and the burn-down is published per PR by the marker-dump gate. The rate is an
estimate; the denominator is not.

# Related Proposals

* **[DEP 0008 — Test Strategy](0008-testing-strategy.md)** defines the shared vocabulary, the test
  taxonomy (linting, unit, integration, end-to-end, benchmark, stress), directory structure, coverage
  expectations, and how tests map onto the development and release life-cycle. It states as a goal
  that tests "should be easy to write and run both locally and for CI" and that the strategy must fit
  "multiple programming languages and deployment targets", but does not specify the mechanism by
  which a single test achieves that. This proposal supplies the mechanism.

  It does **not** change 0008's taxonomy: the six classes here are an orthogonal *authoring* axis
  (what a test needs in order to run) that projects onto 0008's types through the `test_class`
  mapping in §I9. The projection is partial in both directions and this proposal says so rather than
  claiming a clean fit: 0008's `stress` is a shape of a `functional` or `experiment` item rather than
  a class of its own, and its `linting` type is not a runtime test at all, so **5 of 0008's 7 types**
  have a source class here. `infra_helper` deliberately projects to no marker, because it is not a
  test.

  **Marker names.** 0008 §*Test segmentation* publishes `gpus_needed_0/1/2`, `premerge`, `postmerge`
  and `tensorrt_llm`; DEP 0009 publishes `gpu_0…gpu_8`, `pre_merge` and `trtllm`. The repository
  implements 0009's spelling and **none** of 0008's exists among the 68 registered markers at
  `0765d30ad8` **[m]**. A proposal whose REQ 13 makes markers the wire format cannot leave that
  ambiguous: **this proposal normatively adopts the implemented (0009) names**, and records that
  0008's published list is stale and should be corrected there.

* **[DEP 0009 — Testing in CI Strategy](0009-testing-in-ci-strategy.md)** defines CI workflows, which
  pipeline runs which tests, segmentation via pytest markers and cargo groupings, coverage metrics,
  and quality gates. It governs *when and where* tests execute. This proposal governs *how a test is
  written*, and is deliberately constrained by 0009: marker-based selection and the VRAM-aware GPU
  scheduler must keep working, so declared requirements are applied as markers (REQ 13) rather than
  replacing them, and the parity gate in §I9 exists to prove that constraint mechanically. The two
  additive gates this proposal introduces are named in *Non Goals*.

Neither proposal addresses test-framework *design*. That gap is what this proposal fills.

# Alternate Solutions

## Alt 1 Keep The Current Payload-And-Marker Approach

Already delivers deployment agnosticism for the payload and verification path, and is strictly less
work.

**Reason Rejected:** insufficient. There is no home for readiness policy, capability negotiation,
lifecycle control, evidence or fleet semantics, so every test re-implements them — which is how the
shallowness patterns in §M6 arose — and it does nothing for the platform matrix or the manifest
corpus.

## Alt 2 Fixture-Per-Endpoint, No Framework

A `frontend_url` fixture plus free functions. Simpler.

**Reason Rejected:** behaviour that belongs to a role (waiting, retry, restart, protocol quirks) ends
up in ownerless helper modules, and there is no natural place for capabilities, probes, lifecycle or
an evidence bundle.

## Alt 3 Revision 1's Component-Oriented Façade

`dynamo.frontend.query(...)`, `dynamo.worker.kill()`.

**Reason Rejected:** withdrawn on measurement. The component object implies a scope the provider
never promised and cannot express, which is a live defect in the prototype (§P4, `deployment.py:45`,
`:51` versus `:57`, and `Docker.stop`/`Docker.kill` at `:191`/`:196` **[p]**); two surfaces means two
sources of truth for readiness, retry and scope; and a component-first scenario engine must re-derive
dispatch from a role list, which is the code path that produced a reported silent no-op in a green
run **[a]**. Replaced by verbs plus a selector, with `sut.roles[role]` as a read-only typed view.

## Alt 4 A Single Receiver For The Whole Test

One object carrying actions and assertions.

**Reason Rejected:** it invites the torn-down-deployment `AttributeError` at corpus scale (three live
instances, §M2/R1), and it makes offline re-scoring impossible because assertions can reach the live
system.

## Alt 5 Adopt The Internal Peer Harness Wholesale

It is the more mature system on the operations axis and its verdict layer is already the right shape.

**Reason Rejected:** as the *whole* answer. It has no transport axis at all (it cannot run a scenario
against a local process), no capability or requirement model, and a manifest loader that reaches at
most **67 of 295** in-tree manifests **[m]**. Adopted *by layer* instead (§M2/R5), which is what
REQ 16 states.

## Alt 6 Make Declared Requirements The Sole Marker Carrier

Cleaner as a declaration surface.

**Reason Rejected:** two measurements. **71 of 106** `profiled_vram_gib` applications are
non-decorator **[m]**, 64 of them through a `pytest.param(..., marks=…)` table and one *computed at
collection time* from a topology matrix, which a decorator cannot express at all. And the pre-commit
marker hook runs a real collection in an isolated 8-dependency venv the harness cannot enter, where
its stubber would make decorated tests **vanish** from the report rather than report as marker-less
**[m]**. A red or blind pre-commit hook affects every PR in the repository.

## Alt 7 An Async-Native Harness

Matches the Kubernetes runtime and the load path.

**Reason Rejected:** the 56-file synchronous subprocess census **[m]** and the fact that every
existing check is synchronous **[p]**: converting either direction is thousands of lines of churn for
zero coverage. The reverse trade is named as this design's first falsifier.

## Alt 8 Adopt An Off-The-Shelf Container-Lifecycle Harness

**Reason Rejected:** solves container lifecycle, not Dynamo's roles, capabilities, evidence or
multi-replica semantics. Suitable *underneath* the Docker provider rather than as a replacement.

## Alt 9 Generate Per-Platform Suites From A Shared Core

**Reason Rejected:** code generation preserves the per-cell structure and its multiplicative cost, and
generated tests are harder to debug than a runtime-configured object.

# Background

## Risks, Honest Weaknesses And Falsifiers

Each item names what would show it wrong. This section is deliberately falsifiable rather than
reassuring.

1. **The sync-over-async bridge is the single riskiest mechanism** (§I3, phase 5b). *Falsifier:* if
   the loop thread wedges twice in the first quarter after phase 6, the trade was wrong and the
   answer is an async-native `Sut` with a sync adapter for legacy call sites — the reverse of what is
   built here (Alt 7).
2. **Phases 1–5 deliver no façade to almost the whole corpus.** That is on purpose and it is also the
   design's weakest quarter. *Falsifier:* if the burn-down misses 20 files per quarter twice
   consecutively, the ratchet is producing infrastructure nobody adopts.
3. **Markers-as-wire-format makes the requirement DSL second-class for a long time.** *Falsifier:* if
   `Requirements.of(item)` marker-derivation proves lossy for any of the 68 registered markers, the
   read-view model is wrong and the DSL must become authoritative behind a much larger phase 2.
4. **Phase 4's `UNKNOWN` flip will arrive as a wave of red on a suite people believe is green.** That
   is the point, and it will still be argued about. *Falsifier:* if the flip list exceeds a third of
   findings — a threshold chosen by the author, not measured — the checks are worse than believed and
   phase 4 becomes a checks rewrite rather than a reader.
5. **The Kubernetes composition (phase 6) is the largest single merge and no organising idea makes it
   smaller.** Phases, protocols and documents all give it a home; none shrinks it.
6. **`Sink` is the least-evidenced protocol.** The evidence is a small number of loopback-bound sinks
   in the current corpus **[a — the count is not re-derived]**, which is a thin basis for a third
   addressing direction. It is scheduled last for that reason. *Falsifier:* if no test needs it by
   phase 11, drop it.
7. **The bundle schema becomes a cross-repository ABI and will skew.** The conformance suite and the
   migration-duration freeze are the mitigation; both are stated in *Deferred to Implementation*
   rather than solved.
8. **Namespace-scrub default drift.** Correct for dedicated namespaces, destructive in shared ones.
   It is exactly the shape that gets copy-pasted wrong; the `namespace_ownership` requirement is the
   guard, and guards get bypassed.
9. **Internal CI egress is unverified** and is the only unknown that can invalidate the packaging
   decision outright. It is phase 1's gate for exactly that reason.

**The single measurement that would falsify the whole design:** if, after phase 3, `Plan.from_manifest`
does not construct essentially the entire manifest corpus — i.e. if the manifest merge is not
mechanical — then the "one manifest model, several renderers" premise this design rests on is false,
and the correct answer is two manifest models with an explicit conversion boundary, not a converged
framework.

## References

* [DEP 0008 — Test Strategy](0008-testing-strategy.md)
* [DEP 0009 — Testing in CI Strategy](0009-testing-in-ci-strategy.md)
* [ai-dynamo/dynamo#12690](https://github.com/ai-dynamo/dynamo/pull/12690) — makes pytests agnostic
  of how Dynamo is deployed; introduces the shared payload runner and the deployment-coupling
  predicate this proposal builds on.
* [ai-dynamo/dynamo#13979](https://github.com/ai-dynamo/dynamo/pull/13979) — multi-document DGD
  loading in `DeploymentSpec` (`dgd_utils.py:496`) and the first recipe-driven functional test
  (`tests/deploy/test_recipe_tool_execution.py`); a stated dependency of phases 3 and 5a.
* `ai-dynamo/dynamo` branch `dtokarev/dgd-shell-style-lc` — widens the shell-form predicate to any
  short-option cluster ending in `c`; a stated dependency of phase 3.
* `ai-dynamo/dynamo` branch `dtokarev/tests-v2-harness` at **`2fd6eb2586`** — the prototype cited as
  **[p]** throughout §P4, §M3 and Alt 3. Pinned because the branch can be rebased.
* `ai-dynamo/dynamo` at **`0765d30ad8`** — the commit every **[m]** measurement is taken at. Use the
  literal SHA; `origin/main` has moved since.
* The internal peer harness (`dyntest`) at **`e77b0d3`** — the source of the operations capability
  set REQ 16 enumerates, and of every **[p]** claim marked *internal*. It is **not publicly
  readable**; REQ 16's normative content is the enumerated capability set in this document, not
  conformance to that repository.
* `tests/README.md` and `.ai/pytest-guidelines.md` in `ai-dynamo/dynamo` — the authoring conventions
  added alongside #12690.
* The four CI mechanisms this proposal must not break, all at `0765d30ad8`: `conftest.py:91-122`
  (repository-root category defaults) with `tests/marker_categories.py`; `tests/conftest.py:153`,
  `:661`, `:693-712`, `:717`, `:761` (the `--dry-run` flag, the `trylast` collection hook, the VRAM
  deselect, `write_test_meta`, and `items.clear()`); `.github/workflows/shared-test.yml:251` (the
  sequential-lane transform); and `.pre-commit-config.yaml:170-183` with
  `tests/report_pytest_markers.py:683` (the isolated-venv real collection).

## Terminology & Definitions

| Term | Definition |
|---|---|
| Role | A named participant in a deployment — frontend, worker, prefill, decode, router, planner, operator, KVBM, load generator, self. |
| `Sel` | The single selector value: role plus optional replica, policy, fraction, rank, process and port. |
| `Plan` | The desired-state document: per-role image, environment, arguments, replicas, resources, placement, probes and mounts, plus every document of the source manifest. |
| `Site` | The environment profile: cluster or host facts, credentials with remediation, output roots, per-environment workarounds. |
| Provider | A platform implementation of the seven capability protocols. |
| Reach / Ingress / Sink | The three addressing directions: test process → SUT, deployed generator → SUT, SUT → test-owned service. |
| `Sut` | The ACT-phase receiver. Holds the live system; every verb returns a `Handle`. |
| `Handle` | The record of one verb invocation: brackets, arguments, resolved selection, proof, artifacts. Frozen; a verb that completes a running handle returns a new one. |
| `ArgV` | The argument value type: form (`ARGV` / `SHELL` / `UNPARSEABLE`), the retained original script, its tokens and their byte spans. Writes splice the original. |
| `Need` | A declared dependency plus its polarity (`REQUIREMENT` ⇒ skip when unmet, `PRECONDITION` ⇒ error when unmet). |
| Evidence bundle | The sealed on-disk directory a run produces; the sole input to CHECK. `sealed/` is immutable after SEAL; `verdict/` is written after and is rewritable. |
| `Seal` | The comparison of declared producer promises against measured delivery. The single authority on admissibility. |
| Admissibility | Whether the evidence supports a verdict at all, evaluated before any system verdict. |
| Grants | The powers a test declared it needs; absent grants make the verb raise rather than widen. |
| Prefix affinity | KV-aware routing sending identical prompt prefixes to the same worker; the reason a fixed probe cannot detect a bad replica. |
| Peer harness (`dyntest`) | NVIDIA's internal scenario-based Dynamo test harness, pinned at `e77b0d3`. The mature implementation of the Kubernetes operations capability set REQ 16 enumerates, and the source of every **[p]** claim marked *internal*. Not publicly readable. |
| Prototype (`tests-v2`) | The public prototype on branch `dtokarev/tests-v2-harness` at `2fd6eb2586` that revision 1's façade was drafted against. |

## Acronyms & Abbreviations

| Acronym | Meaning |
|---|---|
| DGD | DynamoGraphDeployment (Kubernetes custom resource) |
| DGDR | DynamoGraphDeploymentRequest (the declarative, operator-synthesised deployment mode) |
| SUT | System Under Test |
| TP / EP | Tensor Parallel / Expert Parallel size |
| MoE | Mixture of Experts |
| MNNVL | Multi-Node NVLink |
| KVBM | KV Block Manager |
| FT | Fault Tolerance |
| ABI | Application Binary Interface (here: an upstream engine's internal Python API) |
| AST | Abstract Syntax Tree (the parsing instrument several §M measurements use) |

## Revision History

**Revision 2** (this document). The object model in revision 1 — a `Dynamo` façade with per-component
attributes, `transport=` × `deployment=` as two independent constructor arguments, and three test
classes — is withdrawn and replaced. *Goals* through *Implementation Phases* are rewritten; the
header metadata and the relationship to DEP 0008/0009 are unchanged. Six corrections to revision 1's
own claims are listed in [§M2](#m2-six-corrections-carried-in-the-open) rather than applied silently,
and the `profiled_vram_gib` correction there is itself a correction of an earlier correction —
re-derived with `ast` rather than with a line-oriented regex, which is what produced both wrong
answers.

Revising in place rather than superseding is deliberate and permitted: DEP 0000 §*Significant Changes
After Review* requires a new proposal for significant changes made **after** review, and this
proposal's `Review Date` is TBD with no review having taken place. Revision 1 remains readable in this
file's git history and in the diff of
[ai-dynamo/enhancements#100](https://github.com/ai-dynamo/enhancements/pull/100).

**Revision 1**. Initial proposal: component-oriented façade, three test classes, `transport=` ×
`deployment=` axes, five implementation phases. Measured at `4c3e61f107`.
