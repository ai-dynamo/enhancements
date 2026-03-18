# Automated Dependency Upgrade Pipelines

**Status**: Draft

**Authors**: Anant

**Category**: Process - CI/CD

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Introduce automated CI pipelines that detect new releases of Dynamo's core framework dependencies (vLLM, TensorRT-LLM, SGLang, NIXL), validate compatibility by running trial container builds, and create upgrade pull requests when builds succeed. This replaces the current fully manual upgrade process and reduces the time between an upstream release and a Dynamo upgrade.

# Motivation

Dynamo depends on four fast-moving upstream frameworks: vLLM, TensorRT-LLM (TRTLLM), SGLang, and NIXL. Each upgrade currently requires manual, coordinated changes across 6+ files (`container/context.yaml`, `pyproject.toml`, `install_vllm.sh`, `install_nixl.sh`, documentation). This process is:

- **Slow**: Engineers must discover new releases, understand what changed, and manually update multiple files.
- **Error-prone**: Version references are scattered across YAML, TOML, and shell scripts with no single-command update path.
- **Reactive**: The team only learns about breaking changes after attempting an upgrade, rather than catching them early through continuous integration.

Recent examples of manual upgrades include `chore: bump trtllm to 1.3.0rc7` and `chore: Advance deepseek wideep and qwen-235b recipes to 1.0.1 TRTLLM version` — each touching multiple files with framework-specific update logic.

## Goals

* Automatically detect when a new stable release is available for vLLM, TRTLLM, SGLang, or NIXL

* Validate that Dynamo's container images build successfully with the new version before involving a human

* Create draft pull requests with all necessary file changes when a build succeeds

* Notify the team via Slack when an upgrade build fails so breaking changes are surfaced early

* Reduce the manual effort for dependency upgrades to reviewing and merging a pre-built PR

### Non Goals

* Fully automated merging of upgrade PRs (human review is still required)

* Upgrading transitive dependencies (e.g., `transformers`, `torch`) — these require manual analysis

* Upgrading recipe YAML files or documentation (flagged in PR for manual review)

## Requirements

### REQ 1 Version Detection

The system **MUST** detect the latest stable release for each framework from its canonical source:
- vLLM: GitHub releases (`vllm-project/vllm`)
- SGLang: GitHub releases (`sgl-project/sglang`) + Docker Hub image tag verification (`lmsysorg/sglang`)
- TRTLLM: NVIDIA PyPI index (`pypi.nvidia.com/tensorrt-llm`)
- NIXL: GitHub releases (`ai-dynamo/nixl`)

The system **MUST** compare the detected version against the currently pinned version in `container/context.yaml` and **MUST** skip the workflow if no upgrade is available.

### REQ 2 Coordinated Multi-File Version Bump

The system **MUST** update version references consistently across all files where a framework version is pinned. At minimum:

| File | vLLM | SGLang | TRTLLM | NIXL |
|------|------|--------|--------|------|
| `container/context.yaml` | `vllm_ref` (per CUDA ver) | `runtime_image_tag` (per CUDA ver) | `pip_wheel`, `github_trtllm_commit` | `nixl_ref` |
| `pyproject.toml` | `vllm==X.Y.Z` | `sglang==X.Y.Z` | `tensorrt-llm==X.Y.Z` | `nixl<=X.Y.Z` |
| `container/deps/vllm/install_vllm.sh` | `VLLM_VER` (line 15) | — | — | — |
| `container/deps/trtllm/install_nixl.sh` | — | — | — | `NIXL_COMMIT` (line 30) |
| `deploy/pre-deployment/nixl/build_and_deploy.sh` | — | — | — | `NIXL_VERSION` (line 9) |

The system **SHOULD** flag files requiring manual review in the PR body (docs, recipes, `requirements.common.txt`, trtllm torch versions).

### REQ 3 Trial Build and Test Validation

The system **MUST** run a full build and test cycle for the target framework on at least one platform (amd64) and one CUDA version before creating a PR. This **MUST** include CPU-only, single GPU, and multi-GPU tests matching the post-merge CI test markers. The system **MUST** reuse the existing `build-test-distribute-flavor-matrix.yml` workflow with tests enabled (NOT `build_only`).

### REQ 4 Pull Request Creation

On successful build, the system **MUST** create a pull request with:
- Title: `chore: bump {framework} to {version}`
- Labels: `dep-upgrade`, `{framework}`
- Body containing: version change summary, build log link, manual review checklist

If a prior open PR exists for the same framework and version, the system **SHOULD** skip creating a duplicate.

### REQ 5 Failure Notification

On build failure, the system **MUST** send a Slack notification including the framework name, attempted version, and a link to the build logs.

### REQ 6 Independent Framework Scheduling

Each framework **MUST** be independently schedulable. A failure in one framework's upgrade pipeline **MUST NOT** block or delay other frameworks.

# Proposal

## Overview

The solution consists of three components:

1. **Version detection script** (`detect_latest_versions.py`) — queries upstream sources for latest releases
2. **Version bump script** (`bump_dependency.py`) — performs coordinated multi-file version updates
3. **GitHub Actions workflow** (`dep-upgrade-release.yml`) — orchestrates detection → branch → build → PR/notify

The workflow is branch-based: it creates a temporary branch with the version bump changes, runs the existing build infrastructure against it, and either opens a PR (on success) or cleans up and notifies (on failure).

## Version Detection Script

**Location**: `.github/scripts/detect_latest_versions.py`

A standalone Python script (stdlib only + `urllib`) that queries upstream package sources.

```
$ python detect_latest_versions.py --framework vllm
{"framework": "vllm", "current": "v0.17.1", "latest": "v0.18.0", "needs_update": true}
```

| Framework | Source | API |
|-----------|--------|-----|
| vLLM | GitHub | `GET /repos/vllm-project/vllm/releases` → filter non-prerelease → highest semver |
| SGLang | GitHub + Docker Hub | `GET /repos/sgl-project/sglang/releases` + verify `lmsysorg/sglang:v{ver}-runtime` tag exists |
| TRTLLM | NVIDIA PyPI | Parse simple repository index at `pypi.nvidia.com/tensorrt-llm/` |
| NIXL | GitHub | `GET /repos/ai-dynamo/nixl/releases` → latest |

The script reads the current pinned version from `container/context.yaml` for comparison. It uses `GITHUB_TOKEN` for API authentication to avoid rate limits.

## Version Bump Script

**Location**: `.github/scripts/bump_dependency.py`

A Python script that modifies version strings across all relevant files for a given framework.

```
$ python bump_dependency.py --framework vllm --version v0.18.0
Updated container/context.yaml: vllm_ref v0.17.1 -> v0.18.0
Updated pyproject.toml: vllm==0.17.1 -> vllm==0.18.0
Updated container/deps/vllm/install_vllm.sh: VLLM_VER 0.17.1 -> 0.18.0
```

Implementation approach:
- YAML files: parsed with `pyyaml` (available in CI runners)
- TOML files: regex-based substitution (avoids needing `tomli_w` and preserves formatting/comments)
- Shell scripts: regex substitution on known variable assignment patterns

### SGLang-Specific Bump Logic

SGLang's `runtime_image_tag` in `context.yaml` has CUDA-dependent suffixes:
- CUDA 12.9: `v{version}-runtime`
- CUDA 13.0: `v{version}-cu130-runtime`

The bump script **MUST** verify the Docker image tags exist on Docker Hub before writing changes.

### TRTLLM-Specific Constraints

Only `pip_wheel` and `github_trtllm_commit` are auto-bumped. The torch ecosystem versions (`torch_version`, `pytorch_triton_ver`, `torchao_ver`, etc.) are tied to the NGC PyTorch base image, not to the TRTLLM version, and **MUST NOT** be automatically modified.

### NIXL Cross-Reference

NIXL appears in both the `vllm` and `sglang` optional dependency sections of `pyproject.toml`. The bump script **MUST** update both references.

## Workflow

The workflow is split into two phases across two workflow files. This is necessary because GitHub Actions reusable workflows (`workflow_call`) check out code from the calling workflow's ref. If a single workflow running on `main` calls the build pipeline, it would build `main`'s code — not the upgrade branch with version bumps. By dispatching phase 2 on the upgrade branch, `actions/checkout` naturally picks up the bumped versions.

### Phase 1: Detect and Prepare

**Location**: `.github/workflows/dep-upgrade-release.yml`

Triggered by schedule or `workflow_dispatch` on `main`. Detects new versions, creates the upgrade branch, and dispatches phase 2.

```
┌─────────────────────────────────────────────────────────┐
│  Trigger: cron (per-framework) or workflow_dispatch      │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   detect-release    │
              │  (latest version?)  │
              └────────┬────────────┘
                       │
              ┌────────┴────────┐
              │ no new release  │──→ exit(0)
              └─────────────────┘
                       │ new release found
                       ▼
              ┌─────────────────────┐
              │   prepare-branch    │
              │  (create branch,    │
              │   bump versions,    │
              │   commit & push)    │
              └────────┬────────────┘
                       │
                       ▼
              ┌─────────────────────┐
              │   dispatch phase 2  │
              │  gh workflow run    │
              │  dep-upgrade-       │
              │  test.yml           │
              │  --ref <branch>     │
              └─────────────────────┘
```

### Phase 2: Build, Test, and PR

**Location**: `.github/workflows/dep-upgrade-test.yml`

Triggered via `workflow_dispatch` on the upgrade branch. Since it runs in the context of the upgrade branch, all `actions/checkout` calls naturally pick up the bumped version files.

```
┌─────────────────────────────────────────────────────────┐
│  Trigger: workflow_dispatch on upgrade branch            │
│  Inputs: framework, version                              │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   setup             │
              │  (resolve framework │
              │   → CUDA versions,  │
              │   test markers)     │
              └────────┬────────────┘
                       │
                       ▼
              ┌─────────────────────┐
              │   build-and-test    │
              │  (build-test-       │
              │   distribute-       │
              │   flavor-matrix)    │
              │  per framework      │
              │  (nixl → all 3)     │
              └────────┬────────────┘
                       │
              ┌────────┴────────┐
              │                 │
         success            failure
              │                 │
              ▼                 ▼
     ┌────────────┐    ┌──────────────┐
     │ create-pr  │    │ notify-slack │
     │ (gh pr     │    │ + cleanup    │
     │  create)   │    │   branch     │
     └────────────┘    └──────────────┘
```

**Implementation**: The `setup` job resolves framework-specific configuration because `workflow_call` inputs cannot use conditionals. For NIXL upgrades, the workflow runs all three framework pipelines (vLLM, SGLang, TRTLLM) in parallel since NIXL is a shared dependency built into each framework's container.

```yaml
# dep-upgrade-test.yml (sketch)
name: Dependency Upgrade Test

on:
  workflow_dispatch:
    inputs:
      framework:
        description: 'Framework being upgraded'
        required: true
        type: choice
        options: [vllm, sglang, trtllm, nixl]
      version:
        description: 'Version being upgraded to'
        required: true
        type: string

jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      # Which framework builds to run (nixl → all three)
      run_vllm: ${{ steps.config.outputs.run_vllm }}
      run_sglang: ${{ steps.config.outputs.run_sglang }}
      run_trtllm: ${{ steps.config.outputs.run_trtllm }}
    steps:
      - id: config
        run: |
          case "${{ inputs.framework }}" in
            vllm)
              echo "run_vllm=true"   >> $GITHUB_OUTPUT
              echo "run_sglang=false" >> $GITHUB_OUTPUT
              echo "run_trtllm=false" >> $GITHUB_OUTPUT
              ;;
            sglang)
              echo "run_vllm=false"  >> $GITHUB_OUTPUT
              echo "run_sglang=true" >> $GITHUB_OUTPUT
              echo "run_trtllm=false" >> $GITHUB_OUTPUT
              ;;
            trtllm)
              echo "run_vllm=false"   >> $GITHUB_OUTPUT
              echo "run_sglang=false" >> $GITHUB_OUTPUT
              echo "run_trtllm=true"  >> $GITHUB_OUTPUT
              ;;
            nixl)
              # NIXL is built into all frameworks — validate all three
              echo "run_vllm=true"   >> $GITHUB_OUTPUT
              echo "run_sglang=true" >> $GITHUB_OUTPUT
              echo "run_trtllm=true" >> $GITHUB_OUTPUT
              ;;
          esac

  vllm-pipeline:
    needs: [setup]
    if: needs.setup.outputs.run_vllm == 'true'
    uses: ./.github/workflows/build-test-distribute-flavor-matrix.yml
    with:
      framework: vllm
      target: runtime
      platforms: '["amd64"]'
      cuda_versions: '["12.9"]'
      builder_name: b-${{ github.run_id }}-${{ github.run_attempt }}
      build_timeout_minutes: 180
      copy_to_acr: false
      cpu_only_test_markers: '(pre_merge or post_merge) and vllm and gpu_0'
      single_gpu_test_markers: '(pre_merge or post_merge) and vllm and gpu_1'
      multi_gpu_test_markers: '(pre_merge or post_merge) and vllm and (gpu_2 or gpu_4)'
      cpu_only_test_timeout_minutes: 60
      single_gpu_test_timeout_minutes: 60
      multi_gpu_test_timeout_minutes: 60
    secrets: inherit

  sglang-pipeline:
    needs: [setup]
    if: needs.setup.outputs.run_sglang == 'true'
    uses: ./.github/workflows/build-test-distribute-flavor-matrix.yml
    with:
      framework: sglang
      target: runtime
      platforms: '["amd64"]'
      cuda_versions: '["12.9"]'
      builder_name: b-${{ github.run_id }}-${{ github.run_attempt }}
      build_timeout_minutes: 180
      copy_to_acr: false
      cpu_only_test_markers: '(pre_merge or post_merge) and sglang and gpu_0'
      single_gpu_test_markers: '(pre_merge or post_merge) and sglang and gpu_1'
      multi_gpu_test_markers: '(pre_merge or post_merge) and sglang and (gpu_2 or gpu_4)'
      cpu_only_test_timeout_minutes: 60
      single_gpu_test_timeout_minutes: 60
      multi_gpu_test_timeout_minutes: 60
    secrets: inherit

  trtllm-pipeline:
    needs: [setup]
    if: needs.setup.outputs.run_trtllm == 'true'
    uses: ./.github/workflows/build-test-distribute-flavor-matrix.yml
    with:
      framework: trtllm
      target: runtime
      platforms: '["amd64"]'
      cuda_versions: '["13.1"]'
      builder_name: b-${{ github.run_id }}-${{ github.run_attempt }}
      build_timeout_minutes: 180
      copy_to_acr: false
      cpu_only_test_markers: '(pre_merge or post_merge) and trtllm and gpu_0'
      single_gpu_test_markers: '(pre_merge or post_merge) and trtllm and gpu_1'
      multi_gpu_test_markers: '(pre_merge or post_merge) and trtllm and (gpu_2 or gpu_4)'
      cpu_only_test_timeout_minutes: 60
      single_gpu_test_timeout_minutes: 60
      multi_gpu_test_timeout_minutes: 60
    secrets: inherit

  create-pr:
    needs: [setup, vllm-pipeline, sglang-pipeline, trtllm-pipeline]
    if: |
      always() &&
      (needs.vllm-pipeline.result == 'success' || needs.vllm-pipeline.result == 'skipped') &&
      (needs.sglang-pipeline.result == 'success' || needs.sglang-pipeline.result == 'skipped') &&
      (needs.trtllm-pipeline.result == 'success' || needs.trtllm-pipeline.result == 'skipped') &&
      !contains(needs.*.result, 'failure')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Create PR
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Create PR with version summary and build link

  notify-failure:
    needs: [setup, vllm-pipeline, sglang-pipeline, trtllm-pipeline]
    if: always() && contains(needs.*.result, 'failure')
    runs-on: ubuntu-latest
    steps:
      - name: Notify Slack + cleanup branch
        # Slack webhook + delete branch
```

### Schedule

Staggered from nightly CI (08:00 UTC) and from each other to avoid BuildKit worker contention:

| Framework | Schedule | Rationale |
|-----------|----------|-----------|
| vLLM | Mon + Thu 10:00 UTC | High upstream velocity |
| SGLang | Tue 10:00 UTC | Moderate velocity |
| TRTLLM | Wed 10:00 UTC | NVIDIA release cadence |
| NIXL | Fri 10:00 UTC | Internal project |

### Build and Test Configuration Per Framework

The trial run uses the same test markers as post-merge CI but on a single platform (amd64) and primary CUDA version to balance coverage with resource cost.

| Framework | CUDA Version | CPU Test Markers | Single GPU Markers | Multi GPU Markers |
|-----------|-------------|------------------|--------------------|--------------------|
| vLLM | `["12.9"]` | `(pre_merge or post_merge) and vllm and gpu_0` | `(pre_merge or post_merge) and vllm and gpu_1` | `(pre_merge or post_merge) and vllm and (gpu_2 or gpu_4)` |
| SGLang | `["12.9"]` | `(pre_merge or post_merge) and sglang and gpu_0` | `(pre_merge or post_merge) and sglang and gpu_1` | `(pre_merge or post_merge) and sglang and (gpu_2 or gpu_4)` |
| TRTLLM | `["13.1"]` | `(pre_merge or post_merge) and trtllm and gpu_0` | `(pre_merge or post_merge) and trtllm and gpu_1` | `(pre_merge or post_merge) and trtllm and (gpu_2 or gpu_4)` |
| NIXL | N/A | Built as part of other framework builds; triggers vLLM, SGLang, and TRTLLM build+test for validation | | |

Test timeouts follow post-merge CI: 60 minutes for CPU, single GPU, and multi-GPU test tiers. Images are NOT copied to ACR (`copy_to_acr: false`) since these are trial builds.

### Permissions

- **Phase 1** (`dep-upgrade-release.yml`): `contents: write` (to push branches) and `actions: write` (to dispatch phase 2 via `gh workflow run`)
- **Phase 2** (`dep-upgrade-test.yml`): `contents: write` (to delete branch on failure) and `pull-requests: write` (to create PRs)

The `GITHUB_TOKEN` is sufficient for same-repository operations.

### PR Lifecycle

- Branch naming: `deps/upgrade-{framework}-{version}`
- If an open PR already exists for the same framework and version: skip (already in progress)

# Implementation Phases

## Phase 1: Release Upgrade Pipelines

Implement the full release detection and upgrade PR pipeline for all 4 frameworks.

**Deliverables:**
- `.github/scripts/detect_latest_versions.py` — version detection from upstream sources
- `.github/scripts/bump_dependency.py` — coordinated multi-file version bump
- `.github/workflows/dep-upgrade-release.yml` — phase 1: detect, branch, bump, dispatch
- `.github/workflows/dep-upgrade-test.yml` — phase 2: build, test, PR/notify (runs on upgrade branch)
- `.github/workflows/dep-upgrade-all.yml` — optional orchestrator for manual triggers

**Rollout:**
1. Scripts developed and tested locally
2. Workflow deployed with `workflow_dispatch` trigger only (no cron)
3. Manual validation for each framework
4. Cron schedules enabled after validation

## Phase 2: Top-of-Main Integration Testing (Future)

Once release tracking is proven stable, add a separate workflow that builds against upstream `main` branches to catch breaking changes before they ship in a release.

**vLLM**: Add `--build-from-source` flag to `install_vllm.sh` for CUDA source builds (currently only CPU/XPU paths build from source). Add `VLLM_BUILD_FROM_SOURCE` ARG to `vllm_framework.Dockerfile`.

**NIXL**: Already builds from source — point `NIXL_COMMIT` to `main`.

**SGLang**: Use `lmsysorg/sglang:nightly-dev-*` Docker images (published daily on Docker Hub). Modify `sglang_runtime.Dockerfile` to detect nightly tags and skip version-pinned `pip install sglang==` (sglang is pre-installed in nightly images).

**TRTLLM**: Test against latest PyPI pre-release/RC versions. Source builds are impractical due to the massive C++ compilation required.

This workflow would run on a separate schedule, report pass/fail to Slack, and would NOT create PRs since `main` is a moving target.

# Background

## Current Version Pinning Locations

| Source | File | Example |
|--------|------|---------|
| Container builds | `container/context.yaml` | `vllm_ref: v0.17.1` |
| Python packages | `pyproject.toml` | `vllm[flashinfer,runai]==0.17.1` |
| vLLM install | `container/deps/vllm/install_vllm.sh` | `VLLM_VER="0.17.1"` |
| NIXL install (container) | `container/deps/trtllm/install_nixl.sh` | `NIXL_COMMIT="0.10.1"` |
| NIXL install (deploy) | `deploy/pre-deployment/nixl/build_and_deploy.sh` | `NIXL_VERSION="0.10.1"` |
| Documentation | `docs/reference/support-matrix.md` | Version compatibility table |

## Current CI Build Infrastructure

The existing CI already supports the building blocks needed:
- `build-test-distribute-flavor-matrix.yml` accepts `build_only: true` for build-without-test runs
- `build-flavor` composite action supports `extra_build_args` and `sanitized_ref_name` for branch-tagged builds
- BuildKit workers are routed by framework flavor
- Slack notifications use `SLACK_NOTIFY_NIGHTLY_WEBHOOK_URL`

## Upstream Release Sources

| Framework | Release Source | Tag Pattern |
|-----------|--------------|-------------|
| vLLM | [github.com/vllm-project/vllm/releases](https://github.com/vllm-project/vllm/releases) | `v0.17.1` |
| SGLang | [github.com/sgl-project/sglang/releases](https://github.com/sgl-project/sglang/releases) | `v0.5.9` |
| TRTLLM | [pypi.nvidia.com/tensorrt-llm](https://pypi.nvidia.com/tensorrt-llm/) | `1.3.0rc7` |
| NIXL | [github.com/ai-dynamo/nixl/releases](https://github.com/ai-dynamo/nixl/releases) | `0.10.1` |
| SGLang Docker | [hub.docker.com/r/lmsysorg/sglang](https://hub.docker.com/r/lmsysorg/sglang/tags) | `v0.5.9-runtime`, `v0.5.9-cu130-runtime` |

## Acronyms & Abbreviations

**TRTLLM**: TensorRT Large Language Model (NVIDIA's LLM inference engine)

**NIXL**: NVIDIA Inference eXchange Library (GPU-to-GPU data transfer library)

**DEP**: Dynamo Enhancement Proposal
