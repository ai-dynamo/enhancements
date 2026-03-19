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

* Upgrading recipe YAML files (these reference Dynamo release versions, not upstream framework versions)

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
| `deploy/pre-deployment/nixl/README.md` | — | — | — | NIXL version references |
| `docs/reference/support-matrix.md` | `main (ToT)` row | `main (ToT)` row | `main (ToT)` row | `main (ToT)` row |

**Files NOT auto-modified** (they track Dynamo release versions, not upstream framework versions):
- `docs/reference/release-artifacts.md` — documents what shipped in each Dynamo release
- Recipe YAML files — reference Dynamo release versions and minimum requirements
- `container/deps/requirements.common.txt` — shared constraints like `transformers>=` that may require cross-framework compatibility analysis
- trtllm `torch_version`, `pytorch_triton_ver`, etc. — tied to the NGC PyTorch base image

### REQ 3 Trial Build and Test Validation

The system **MUST** run the full post-merge CI pipeline for the target framework before creating a PR. This **MUST** reuse the existing `post-merge-ci.yml` workflow directly, dispatched on the upgrade branch, to ensure the validation is identical to what runs on `main` after merge.

### REQ 4 Pull Request Creation

On successful build, the system **MUST** create a pull request with:
- Title: `chore: bump {framework} to {version}`
- Labels: `dep-upgrade`, `backend::{framework}`
- Body containing: version change summary, build log link, manual review checklist

If a prior open PR exists for the same framework and version, the system **SHOULD** skip creating a duplicate.

### REQ 5 Failure Notification

On build failure, the system **MUST** send a Slack notification including the framework name, attempted version, and a link to the build logs.

### REQ 6 Independent Framework Scheduling

Each framework **MUST** be independently schedulable. A failure in one framework's upgrade pipeline **MUST NOT** block or delay other frameworks.

# Proposal

## Overview

The solution consists of four components:

1. **Version detection script** (`detect_latest_versions.py`) — queries upstream sources for latest releases
2. **Version bump script** (`bump_dependency.py`) — performs coordinated multi-file version updates
3. **`auto-dep-upgrade-trigger.yml`** — detects new versions, creates upgrade branch, dispatches post-merge CI. Exits immediately.
4. **`auto-dep-upgrade-complete.yml`** — triggered automatically via `workflow_run` when post-merge CI finishes on a `deps/upgrade-*` branch; creates PR on success, notifies Slack on failure. No polling, no idle runners.

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

Validation reuses `post-merge-ci.yml` directly rather than duplicating pipeline configuration. This requires a small modification to `post-merge-ci.yml`:

**Changes to `post-merge-ci.yml`:**

1. Add `workflow_dispatch` trigger with a `framework` filter input
2. Add conditionals so each framework pipeline only runs when matching the filter (or when no filter is set, preserving existing behavior)
3. Skip dev/EFA/operator/deploy jobs when triggered via `workflow_dispatch` (not relevant for dependency upgrades)

```yaml
# Addition to post-merge-ci.yml triggers
on:
  push:
    branches:
      - main
      - 'release/*.*.*'
  workflow_dispatch:
    inputs:
      framework:
        description: 'Run only this framework pipeline (empty = all)'
        required: false
        type: choice
        options: ['', vllm, sglang, trtllm]

# Each framework pipeline gets a conditional:
# github.event_name == 'push' preserves existing behavior on main/release branches
vllm-pipeline:
    if: github.event_name == 'push' || inputs.framework == '' || inputs.framework == 'vllm'
    ...

# Dev/EFA/operator/deploy jobs skip on workflow_dispatch:
vllm-dev-pipeline:
    if: github.event_name == 'push'
    ...
```

The two upgrade workflows are connected via GitHub's `workflow_run` trigger:

```
auto-dep-upgrade-trigger.yml    ┊  post-merge-ci.yml           ┊  auto-dep-upgrade-complete.yml
────────────────────────────────┊──────────────────────────────┊───────────────────────────────
                                ┊                              ┊
  schedule / dispatch           ┊                              ┊
          │                     ┊                              ┊
          ▼                     ┊                              ┊
   detect latest version        ┊                              ┊
          │                     ┊                              ┊
    no update? ──→ exit         ┊                              ┊
          │                     ┊                              ┊
          ▼                     ┊                              ┊
    PR exists? ──→ exit         ┊                              ┊
          │                     ┊                              ┊
          ▼                     ┊                              ┊
   create branch + bump         ┊                              ┊
          │                     ┊                              ┊
          ▼                     ┊                              ┊
   gh workflow run ──────────────→  runs on upgrade branch     ┊
   --ref <branch>               ┊  (full build + test on       ┊
          │                     ┊   all platforms + CUDA)      ┊
        exit                    ┊         │                    ┊
                                ┊     completes                ┊
                                ┊         │                    ┊
                                ┊  workflow_run trigger ────────→  check conclusion
                                ┊                              ┊        │
                                ┊                              ┊  ┌─────┴─────┐
                                ┊                              ┊  success  failure
                                ┊                              ┊     │        │
                                ┊                              ┊     ▼        ▼
                                ┊                              ┊  create PR  Slack notify
```

### `auto-dep-upgrade-trigger.yml`

```yaml
name: Auto Dependency Upgrade Trigger

on:
  schedule:
    # Wed: all frameworks
    - cron: '0 10 * * 3'
  workflow_dispatch:
    inputs:
      framework:
        description: 'Framework to upgrade'
        required: true
        type: choice
        options: [vllm, sglang, trtllm, nixl]

permissions:
  contents: write
  actions: write

jobs:
  resolve-frameworks:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.resolve.outputs.matrix }}
    steps:
      - id: resolve
        run: |
          if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
            echo 'matrix=["${{ inputs.framework }}"]' >> $GITHUB_OUTPUT
          else
            echo 'matrix=["vllm","sglang","trtllm","nixl"]' >> $GITHUB_OUTPUT
          fi

  upgrade:
    needs: [resolve-frameworks]
    if: needs.resolve-frameworks.outputs.matrix != '[]'
    strategy:
      fail-fast: false
      matrix:
        framework: ${{ fromJson(needs.resolve-frameworks.outputs.matrix) }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Detect latest version
        id: detect
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          RESULT=$(python .github/scripts/detect_latest_versions.py \
            --framework ${{ matrix.framework }})
          echo "latest_version=$(echo $RESULT | jq -r '.latest')" >> $GITHUB_OUTPUT
          echo "needs_update=$(echo $RESULT | jq -r '.needs_update')" >> $GITHUB_OUTPUT

      - name: Check for existing PR
        if: steps.detect.outputs.needs_update == 'true'
        id: check-pr
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          VERSION="${{ steps.detect.outputs.latest_version }}"
          BRANCH="deps/upgrade-${{ matrix.framework }}-${VERSION}"
          EXISTING=$(gh pr list --head "$BRANCH" --json title --jq 'length')
          if [ "$EXISTING" -gt 0 ]; then
            echo "skip=true" >> $GITHUB_OUTPUT
          else
            echo "skip=false" >> $GITHUB_OUTPUT
          fi

      - name: Create upgrade branch
        if: steps.detect.outputs.needs_update == 'true' && steps.check-pr.outputs.skip != 'true'
        run: |
          VERSION="${{ steps.detect.outputs.latest_version }}"
          BRANCH="deps/upgrade-${{ matrix.framework }}-${VERSION}"

          git checkout -b $BRANCH
          python .github/scripts/bump_dependency.py \
            --framework ${{ matrix.framework }} --version $VERSION
          git add -A
          git commit -s -m "chore: bump ${{ matrix.framework }} to ${VERSION}"
          git push origin $BRANCH

      - name: Dispatch post-merge CI on upgrade branch
        if: steps.detect.outputs.needs_update == 'true' && steps.check-pr.outputs.skip != 'true'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          VERSION="${{ steps.detect.outputs.latest_version }}"
          BRANCH="deps/upgrade-${{ matrix.framework }}-${VERSION}"

          # For nixl, run all frameworks; otherwise filter
          FRAMEWORK_ARG=""
          if [ "${{ matrix.framework }}" != "nixl" ]; then
            FRAMEWORK_ARG="-f framework=${{ matrix.framework }}"
          fi

          gh workflow run post-merge-ci.yml --ref $BRANCH $FRAMEWORK_ARG
```

### `auto-dep-upgrade-complete.yml`

Triggered automatically by GitHub when post-merge CI finishes on an upgrade branch. No polling needed.

```yaml
name: Auto Dependency Upgrade Complete

on:
  workflow_run:
    # TODO: fragile name coupling — consider repository_dispatch as alternative
    workflows: ["Post-Merge CI Pipeline"]
    types: [completed]
    branches:
      - 'deps/upgrade-**'

permissions:
  contents: read
  pull-requests: write

jobs:
  handle-result:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.workflow_run.head_branch }}

      - name: Extract framework and version from branch
        id: parse
        run: |
          # deps/upgrade-vllm-v0.18.0 → framework=vllm, version=v0.18.0
          BRANCH="${{ github.event.workflow_run.head_branch }}"
          SUFFIX="${BRANCH#deps/upgrade-}"
          FRAMEWORK="${SUFFIX%%-*}"
          VERSION="${SUFFIX#*-}"
          echo "framework=$FRAMEWORK" >> $GITHUB_OUTPUT
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      - name: Create PR
        if: github.event.workflow_run.conclusion == 'success'
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr create \
            --head "${{ github.event.workflow_run.head_branch }}" \
            --title "chore: bump ${{ steps.parse.outputs.framework }} to ${{ steps.parse.outputs.version }}" \
            --label "dep-upgrade" \
            --label "backend::${{ steps.parse.outputs.framework }}" \
            --body "$(cat <<EOF
          ## Dependency Upgrade

          **Framework:** ${{ steps.parse.outputs.framework }}
          **Version:** ${{ steps.parse.outputs.version }}
          **Validation:** [Post-merge CI run](${{ github.event.workflow_run.html_url }})
          EOF
          )"

      - name: Notify Slack on failure
        if: github.event.workflow_run.conclusion == 'failure'
        uses: slackapi/slack-github-action@v2
        with:
          webhook: ${{ secrets.SLACK_NOTIFY_NIGHTLY_WEBHOOK_URL }}
          webhook-type: incoming-webhook
          payload: |
            blocks:
              - type: "section"
                text:
                  type: mrkdwn
                  text: >-
                    :alert: *Dependency upgrade failed*
                    Framework: ${{ steps.parse.outputs.framework }}
                    Version: ${{ steps.parse.outputs.version }}
                    <${{ github.event.workflow_run.html_url }}|Build Logs>
```

### Schedule

Staggered from nightly CI (08:00 UTC) and from each other to avoid BuildKit worker contention:

| Day | Frameworks | Schedule |
|-----|-----------|----------|
| Wednesday | All (vLLM, SGLang, TRTLLM, NIXL) | 10:00 UTC |

Single mid-week schedule. All frameworks run in parallel via matrix strategy with `fail-fast: false`. The workflow also supports `workflow_dispatch` for manual triggers when someone wants to upgrade a specific framework outside of the regular cadence.

### Build and Test Configuration

Since phase 2 reuses `post-merge-ci.yml` directly, all CUDA versions, platforms, test markers, and timeouts are inherited from the existing post-merge configuration. No separate configuration is needed.

For single-framework upgrades (vllm, sglang, trtllm), only that framework's pipeline runs. For NIXL upgrades, all three framework pipelines run since NIXL is a shared dependency.

### Permissions

- `auto-dep-upgrade-trigger.yml`: `contents: write` (to push branches), `actions: write` (to dispatch post-merge CI)
- `auto-dep-upgrade-complete.yml`: `contents: read`, `pull-requests: write` (to create PRs)

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
- `.github/workflows/auto-dep-upgrade-trigger.yml` — detect, branch, bump, dispatch post-merge CI
- `.github/workflows/auto-dep-upgrade-complete.yml` — react to CI result, create PR or notify Slack
- `.github/workflows/post-merge-ci.yml` — **modified**: add `workflow_dispatch` trigger with framework filter

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

### Switch Validation to Nightly Pipeline

Once the nightly CI pipeline is stable and reliable, switch the validation step from dispatching `post-merge-ci.yml` to dispatching `nightly-ci.yml` instead. The nightly pipeline runs a broader test suite (including tests not in post-merge) and would provide higher confidence in upgrade PRs.

# Background

## Current Version Pinning Locations

| Source | File | Example |
|--------|------|---------|
| Container builds | `container/context.yaml` | `vllm_ref: v0.17.1` |
| Python packages | `pyproject.toml` | `vllm[flashinfer,runai]==0.17.1` |
| vLLM install | `container/deps/vllm/install_vllm.sh` | `VLLM_VER="0.17.1"` |
| NIXL install (container) | `container/deps/trtllm/install_nixl.sh` | `NIXL_COMMIT="0.10.1"` |
| NIXL install (deploy) | `deploy/pre-deployment/nixl/build_and_deploy.sh` | `NIXL_VERSION="0.10.1"` |
| NIXL deploy docs | `deploy/pre-deployment/nixl/README.md` | `version 0.10.1` |
| Support matrix | `docs/reference/support-matrix.md` | `main (ToT)` row in Backend Dependencies table |

## Current CI Build Infrastructure

The existing CI infrastructure is reused directly:
- `post-merge-ci.yml` runs full build + test for all frameworks — dispatched on the upgrade branch via `workflow_dispatch`
- `workflow_run` trigger enables the completion workflow to react when post-merge CI finishes
- Slack notifications use `SLACK_NOTIFY_NIGHTLY_WEBHOOK_URL`

## Upstream Release Sources

| Framework | Release Source | Tag Pattern |
|-----------|--------------|-------------|
| vLLM | [github.com/vllm-project/vllm/releases](https://github.com/vllm-project/vllm/releases) | `v0.17.1` |
| SGLang | [github.com/sgl-project/sglang/releases](https://github.com/sgl-project/sglang/releases) | `v0.5.9` |
| TRTLLM | [pypi.nvidia.com/tensorrt-llm](https://pypi.nvidia.com/tensorrt-llm/) | `1.3.0rc7` |
| NIXL | [github.com/ai-dynamo/nixl/releases](https://github.com/ai-dynamo/nixl/releases) | `0.10.1` |
| SGLang Docker | [hub.docker.com/r/lmsysorg/sglang](https://hub.docker.com/r/lmsysorg/sglang/tags) | `v0.5.9-runtime`, `v0.5.9-cu130-runtime` |
