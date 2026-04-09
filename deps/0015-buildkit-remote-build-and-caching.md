# Remote BuildKit Build Infrastructure and Caching Strategy

**Status**: Published

**Authors**: [@ranrubin](https://github.com/ranrubin)

**Category**: Architecture - CI/CD

**Sponsor**: [@saturley-hall](https://github.com/saturley-hall)

**Required Reviewers**: [@saturley-hall](https://github.com/saturley-hall), [@dmitry-tokarev-nv](https://github.com/dmitry-tokarev-nv)

**Review Date**: 2026-02-03

**Implementation PR / Tracking Issue**: [OPS-2569](https://linear.app/nvidia/issue/OPS-2569/accelerating-container-builds-with-remote-buildkit)

# Summary

This document describes the design and implementation of Dynamo's remote BuildKit build infrastructure — a Kubernetes-native distributed build system that offloads container image builds from GitHub Actions runners to dedicated, persistent BuildKit pods. The system provides deterministic layer-cache affinity across parallel builds, pull-through image mirroring via ECR, and architecture-aware routing for simultaneous amd64 and arm64 builds. A controlled proof-of-concept demonstrated a ~40% reduction in build time on warm-cache runs (22 min vs. 40 min baseline). The result is dramatically shorter CI build times, elimination of Docker-in-Docker, reduced network egress costs, and centralized observability over the entire container build pipeline.

# Motivation

Dynamo's CI/CD pipeline builds container images for three primary flavors — vLLM, TRT-LLM, and SGLang — across two architectures (amd64 and arm64). The legacy approach used Docker-in-Docker (DinD) on ephemeral GitHub Actions runners. This created several compounding problems:

1. **Cache volatility.** Ephemeral runners destroy their local Docker layer cache on termination. Every CI run paid the full cold-build cost. Remote registry caching (ECR `--cache-from`/`--cache-to`) mitigated this partially but introduced significant push overhead and required pulling all layers again on cache hit, negating much of the benefit.

2. **No native multi-architecture support.** DinD runners could not perform native multi-arch builds. Architecture variants required separate runners with distinct image tags (`:latest-arm64`, `:latest-amd64`). This forced users and downstream systems to manage architecture-specific tags rather than pulling a single unified manifest.

3. **Resource inefficiency.** Each build job provisioned a dedicated ephemeral runner for a single-use task. The provisioning overhead and node churn added cost and latency even before any build work began.

4. **Technical debt and security.** DinD is deprecated upstream and requires privileged containers or Docker socket bind-mounts, degrading the cluster's security posture. Long-term maintainability of DinD-based pipelines is poor.

5. **No visibility into build performance.** There was no centralized tracing or metrics for build times, cache hit rates, or layer pull latencies across the fleet.

6. **NAT Gateway egress costs.** Runners in private subnets pulled large base images from public registries through NAT Gateways, incurring high hourly data-processing charges.

## Goals

* Provide persistent, shared layer-cache storage across all CI builds
* Eliminate Docker-in-Docker for all image operations in CI
* Route builds to specific BuildKit pods to maximize Docker layer-cache reuse
* Support simultaneous multi-architecture builds (amd64 and arm64) without interference
* Reduce public-registry pull volume and egress costs via ECR mirrors and public subnet deployment
* Provide centralized observability (metrics, traces, dashboards) for the build fleet
* Scale to zero during off-hours and scale up on demand

### Non Goals

* Local developer builds

# Proof of Concept

Before committing to the full infrastructure build-out, a controlled experiment was conducted using the BuildKit Kubernetes driver to validate the performance hypothesis.

**Setup**: Two persistent BuildKit pods (one ARM64, one AMD64) provisioned dynamically via the Kubernetes driver. Standard multi-architecture build workload (vLLM + TRT-LLM + SGLang).

| Scenario | Description | Build Time | vs. Baseline |
|---|---|---|---|
| Scenario A — Cold Start | Fresh pods, no cached layers (equivalent to current ephemeral runner) | ~40 min | baseline |
| Scenario B — Warm Cache | Same pods reused on a subsequent run with populated local cache | ~22 min | **~40% faster** |

Cold-start performance matched the existing baseline — confirming the approach carries no regression risk. Warm-cache performance showed a 40% reduction in build time for the longest-running flavor. This validated the core hypothesis: persistent pods with retained layer cache deliver meaningful build acceleration at no extra cold-start cost.

# Proposal

## Architecture Overview

The build infrastructure is split across two repositories:

- **[ai-dynamo/velonix](https://github.com/ai-dynamo/velonix)** (private): Kubernetes IaC — the `dynamo-builder` Helm chart, Terraform for IAM/ECR, Flux GitOps overlays, and registry mirror configuration.
- **[ai-dynamo/dynamo](https://github.com/ai-dynamo/dynamo)** (public): CI workflows and composite actions that connect GitHub Actions jobs to the remote BuildKit fleet.

```
GitHub Actions Runner
        │
        │  docker buildx build --builder dynamo-builder-xxx
        ▼
  init-dynamo-builder action
  (route_buildkit.sh → nslookup)
        │
        ▼
  BuildKit StatefulSet (Kubernetes)
  ┌──────────────────────────────────────────────┐
  │  dynamo-builder-amd64-{0,1,2,...}            │
  │  dynamo-builder-arm64-{0,1,2,...}            │
  │                                              │
  │  buildkitd (v0.28.0-custom)                  │
  │  ├── Layer cache  (emptyDir, up to 1.3TB)    │
  │  ├── ECR credential helper                   │
  │  └── Mirror config (docker.io, quay.io,      │
  │                      nvcr.io → ECR)          │
  │                                              │
  │  OpenTelemetry Collector sidecar             │
  └──────────────────────────────────────────────┘
        │
        │  layer pulls (cache miss)
        ▼
  ECR Pull-Through Cache
  (docker.io, quay.io, nvcr.io → ECR mirrors)
```


## Key Components

### `dynamo-builder` Helm Chart

Deployed via Flux to the `buildkit` namespace in the Velonix AWS EKS cluster. The chart provisions:

- **Separate StatefulSets per architecture** — `dynamo-builder-amd64` and `dynamo-builder-arm64`. Each pod mounts a large `emptyDir` volume for its BuildKit layer cache. StatefulSets provide stable pod names (e.g., `dynamo-builder-amd64-0`) so the routing algorithm can deterministically target specific pods.

- **Headless Service** — allows `nslookup` from runners to enumerate live pod IPs without a load balancer in the path.

- **KEDA Autoscaling** — pods scale based on a cron schedule (7AM–7PM PST Mon–Fri) and Prometheus queue depth. Outside work hours the fleet scales to zero, eliminating idle node costs. KEDA was chosen over Kubernetes HPA to allow proactive, event-driven scaling before CPU spikes, rather than reacting after build latency has already increased.

- **Graceful Termination** — a `preStop` hook waits up to 60 minutes for in-flight builds to complete before allowing pod eviction. This prevents mid-build disconnects during node rollovers.

- **PodDisruptionBudget CronJobs** — during work hours, PDBs are created to block node evictions from disrupting active pods. Outside work hours the PDBs are removed so KEDA can scale down freely.

- **Metrics and Tracing** — each pod runs an OpenTelemetry Collector sidecar. Metrics are scraped by Prometheus (ServiceMonitor); traces are emitted to Jaeger. Two Grafana dashboards provide build-fleet visibility: pod utilization, cache size, build durations, and GC activity.

**Velonix PR**: [#147](https://github.com/ai-dynamo/velonix/pull/147) — `dynamo-builder` Helm chart

### `init-dynamo-builder` GitHub Actions Composite Action

The primary entry point for CI workflows. It:

1. Calls `route_buildkit.sh` to determine which BuildKit pod(s) to use for the current build flavor.
2. Calls `bootstrap-buildkit` to register the remote BuildKit endpoint with the local `docker buildx` context on the runner.
3. Exposes the builder name for subsequent `docker buildx build` calls.

All CI steps that build images call `init-dynamo-builder` first. No workflow runs `docker build` directly.

**Fallback mechanism**: If the remote builder is unreachable, the system automatically falls back to the BuildKit Kubernetes driver, which creates BuildKit pods dynamically. This is less cache-efficient (cold start equivalent to baseline) but ensures the CI pipeline is never fully blocked by BuildKit fleet availability.

**Related dynamo PRs**:
- [#7398](https://github.com/ai-dynamo/dynamo/pull/7398) — `builder-refresher` action and `docker-test-image-build` composite action
- [#7522](https://github.com/ai-dynamo/dynamo/pull/7522) — address refresh between builds

## Cache Routing Algorithm

### Problem Statement

Dynamo builds multiple container flavors in parallel — vLLM (CUDA 12.x and 13.x), SGLang, TRT-LLM, and general tooling. Each flavor shares significant base layers (the CUDA deep-learning base image, Dynamo core libraries). For maximum cache reuse, builds sharing common base layers should consistently target the same BuildKit pod. Naive round-robin routing would scatter these builds across the pod pool, defeating the purpose of a shared cache.

Additionally, at non-integer multiples of the number of cache groups, a modulo-based scheme leaves some pods permanently idle.

### Solution: Coverage-Aware Ranked Rendezvous Hashing

**Dynamo PR**: [#6480](https://github.com/ai-dynamo/dynamo/pull/6480)

The routing script (`.github/scripts/route_buildkit.sh`) implements a two-stage algorithm:

**Stage 1 — SHA-256 Scoring**

For every `(group_key, arch, pod_index)` triple, a SHA-256 hash produces a uniformly distributed, deterministic score. The `arch` dimension is included so amd64 and arm64 pod pools are ranked independently.

**Stage 2 — Coverage-Aware Pool Construction**

Pods are assigned to groups round-by-round. In each round, every group picks its highest-scoring unclaimed pod. This guarantees:
- Every active pod appears in at least one group's candidate pool (100% utilization at any pod count).
- Pool size is always `ceil(N/3)`, evenly distributing capacity.
- Determinism: the same pod count always produces the same assignment, so there are no spurious cache misses from routing instability.

**Cache Groups**

| Group | Flavors | Shared Base Layers |
|---|---|---|
| 0 | vLLM, SGLang (CUDA 13.x) | `cuda-dl-base` (13.x) |
| 1 | vLLM, SGLang (CUDA 12.x) | `cuda-dl-base` (12.x) |
| 2 | TRT-LLM, General | TRT-LLM base + common tooling |

**Tradeoff**: On KEDA scale events, coverage rebalancing can cascade and shift pool assignments for groups that didn't directly gain or lose a pod. This produces ~1 cache miss per affected group per scale event. Since KEDA scale events happen at most twice per day (business-hours start and end), this is an acceptable tradeoff for guaranteed full pod utilization.

> **Note**: The original design proposed a dedicated NGINX load balancer with consistent hashing by branch name. The implemented solution achieves the same cache-affinity goal entirely client-side via the routing script, eliminating the load balancer as a dependency and single point of failure.

## ECR Pull-Through Cache and Image Mirroring

**Velonix PR**: [#282](https://github.com/ai-dynamo/velonix/pull/282) — OPS-3435

### Problem

Docker Hub rate-limits, `quay.io` availability, and `nvcr.io` NVIDIA container pull latency were causing intermittent CI failures and adding significant layer-pull time. Each BuildKit pod was pulling large NVIDIA base images (several GB) directly from upstream registries on every cache miss.

### Solution

BuildKit v0.28.0 introduced native mirror configuration in `buildkitd.toml`. The custom `dynamo-builder` image bakes in the [Amazon ECR credential helper](https://github.com/awslabs/amazon-ecr-credential-helper) so BuildKit daemons can authenticate directly to ECR without external secret injection.

**Registry Mirror Mapping**

| Upstream Registry | Mirror | Mechanism |
|---|---|---|
| `docker.io` | ECR pull-through cache endpoint | Automatic (AWS-managed, on first miss) |
| `quay.io` | ECR pull-through cache endpoint | Automatic (AWS-managed, on first miss) |
| `nvcr.io` | ECR repository (`nvcr/` prefix) | Manual sync via `skopeo.sh` script |

For `docker.io` and `quay.io`, AWS ECR's pull-through cache feature fetches and caches layers automatically on first access. For `nvcr.io`, images are explicitly mirrored using `skopeo.sh --context-yaml container/context.yaml`, which parses Dynamo's `context.yaml` to derive the full image list. BuildKit falls back transparently to the upstream registry if a layer is not present in the mirror.

**IAM / EKS Pod Identity**: A Terraform-provisioned IAM policy grants BuildKit pods ECR pull/push and `BatchImportUpstreamImage` permissions, bound to the `dynamo-builder-buildkit` service account via EKS Pod Identity. No secrets are mounted in pods.

## BuildKit Storage and GC Management

BuildKit uses an `emptyDir` volume on the node for its layer cache. Mismatched GC thresholds and kubelet eviction thresholds caused pods to be evicted (OPS-3862) before BuildKit GC could reclaim space.

**Velonix PRs**: [#259](https://github.com/ai-dynamo/velonix/pull/259), [#268](https://github.com/ai-dynamo/velonix/pull/268), [#269](https://github.com/ai-dynamo/velonix/pull/269)

### Configuration

| Parameter | Value | Purpose |
|---|---|---|
| `reservedSpace` | 1100 GB | Hard floor: BuildKit always keeps this space available for active builds |
| `maxUsedSpace` | 1300 GB | GC kicks in before this limit to reclaim old layers |
| `minFreeSpace` | 25% | Trigger GC early when disk headroom narrows |
| `requests.ephemeral-storage` | 1200 Gi | Kubernetes scheduling guarantee |
| `limits.ephemeral-storage` | 1400 Gi | Hard cap enforced by kubelet before eviction |

The key invariant is `maxUsedSpace < limits.ephemeral-storage`, so BuildKit's own GC always triggers before kubelet's eviction threshold is reached. `max-parallelism` is also set to prevent a single large build from starving concurrent builds on the same pod.

## Nightly CI: Fresh Builder Isolation

**Dynamo PR**: [#7678](https://github.com/ai-dynamo/dynamo/pull/7678) — OPS-4120

Nightly CI runs the full build matrix (vLLM, SGLang, TRT-LLM) against the latest main branch. Unlike PR builds, nightly runs must not benefit from stale cached layers that may hide regressions.

The previous approach used `no_cache: true`, which forced all layers to be rebuilt from scratch but did not clear stale mirror configuration on reused BuildKit pods. When the ECR mirror was introduced, stale mirror configs on reused pods caused build failures that `no_cache` did not prevent.

**Solution**: A `fresh_builder: true` flag propagates through the action call stack. When set:
1. `init-dynamo-builder` skips the pod-routing step, forcing the Kubernetes fallback path.
2. `bootstrap-buildkit` bypasses the "already exists" guard and always creates a new builder.

This guarantees a completely fresh pod environment per nightly run: no stale layer cache, no stale mirror config, and no cross-run state leakage.

**Nightly job structure**:
```
create-fresh-builder ──┐
                       ├─→ vllm-pipeline  ──┐
                       ├─→ sglang-pipeline ──┼─→ clean-k8s-builder
                       └─→ trtllm-pipeline ──┘       │
                                                      └─→ notify-slack (on failure)
```

`create-fresh-builder` runs first to pre-warm the builder pod. All pipeline jobs inherit the pre-warmed builder, amortizing bootstrap overhead across the parallel matrix. `clean-k8s-builder` runs with `if: always()` to ensure cleanup regardless of build outcome.

## Compliance Scan: BuildKit Filesystem Extraction

**Dynamo PR**: [#7397](https://github.com/ai-dynamo/dynamo/pull/7397)

The compliance scan (SBOM generation) previously used `docker run` to execute helper scripts inside target containers, requiring a local Docker daemon (DinD). This was replaced with a BuildKit-native approach using `FROM scratch` + `--output type=local`:

```dockerfile
# Dockerfile.extract
FROM ${TARGET_IMAGE} AS target
FROM ubuntu AS extractor
RUN --mount=type=bind,from=target,target=/target \
    python3 helpers/dpkg_helper.py --root /target > /out/dpkg.tsv && \
    python3 helpers/python_helper.py --root /target > /out/python.tsv
FROM scratch
COPY --from=extractor /out /
```

The extractor stage mounts the target image's filesystem read-only via a `bind` mount. Output files are exported directly to the runner filesystem via `--output type=local`. This eliminates DinD entirely and routes all image operations through the existing `init-dynamo-builder` factory.

# Pitfalls and Mitigations

## BuildKit as a Single Point of Failure

**Problem**: If the remote BuildKit deployment fails or becomes unreachable, the entire CI pipeline could be blocked.

**Mitigation**:
- **High Availability**: The fleet is distributed across multiple Availability Zones with KEDA autoscaling to handle load spikes.
- **Automatic Fallback**: If the remote builder is unreachable, `init-dynamo-builder` falls back to the BuildKit Kubernetes driver, which creates pods dynamically. This is cold-cache equivalent to the old baseline but keeps the pipeline unblocked. The fallback is controlled by environment variable toggles without requiring workflow edits.

## Cache Distribution in HA Setup

**Problem**: Parallel builds spread across multiple pods may miss each other's cached layers.

**Mitigation**:
- Worst-case performance equals the current baseline (cold cache) — no regression risk.
- During high-demand business hours, the statistical probability of a warm-cache hit increases significantly as the fleet processes shared base layers concurrently across many builds.
- The client-side rendezvous hashing routing (see above) ensures builds sharing base layers consistently target the same pod, converting statistical probability to near-determinism.

## Disk Pressure and Pod Eviction

**Problem**: Long-lived BuildKit pods accumulate cache data over time. If disk fills before GC triggers, Kubernetes may evict the pod mid-build.

**Mitigation**:
- GC policies in `buildkitd.toml` prune old layers when storage usage approaches the configured threshold (see Storage and GC Management above).
- `ephemeral-storage` pod requests and limits create a scheduling guarantee and a hard eviction ceiling above the GC trigger, ensuring GC always fires first.
- `max-parallelism` prevents a single large build from consuming the full disk while other builds are in-flight.

## Autoscaling Latency

**Problem**: KEDA creates new pods in response to load, but there is a lag between a build queue spike and a new pod becoming ready. During this window, builds queue up on existing pods.

**Mitigation**:
- KEDA is configured with custom metrics exported from the BuildKit Prometheus endpoint, allowing proactive scaling on queue depth before CPU saturates.
- Cron-based pre-scaling at business-hours start (7AM PST) ensures a minimum warm pod count is ready before engineers begin work, minimizing cold-start lag for the first builds of the day.

## Security Risks of Public Subnets

**Problem**: Deploying builders in public subnets to avoid NAT Gateway costs theoretically increases the attack surface.

**Mitigation**:
- All network configurations (Security Groups, subnet rules) are managed strictly via Terraform IaC. This provides an auditable source of truth and prevents manual misconfiguration.
- Security Groups permit inbound traffic from the internal EKS cluster only. No public internet inbound access is allowed to BuildKit pods.
- NetworkPolicy enforces the same restriction at the Kubernetes layer (defense in depth).

# Implementation Phases

## Phase 1 — Infrastructure Preparation (Oct 2025)

**Linear**: [OPS-1494](https://linear.app/nvidia/issue/OPS-1494)
**Velonix PR**: [#52](https://github.com/ai-dynamo/velonix/pull/52)

- Deployed AWS EBS CSI Driver with `gp3` StorageClass for high-performance I/O.
- Provisioned dedicated node groups in public subnets with `workload=buildkit` taints/labels to isolate the build fleet from general workloads.
- Added Azure Container Registry (ACR) pull-through cache for Docker Hub images (Azure cluster) with a Kyverno mutating webhook to rewrite image references. This was the initial caching layer.

## Phase 2 — BuildKit Cluster Deployment (Jan 2026)

**Linear**: [OPS-2576](https://linear.app/nvidia/issue/OPS-2576)
**Velonix PR**: [#147](https://github.com/ai-dynamo/velonix/pull/147)

- Deployed the `dynamo-builder` Helm chart to the AWS CI cluster via Flux.
- Established StatefulSet topology, KEDA autoscaling, graceful termination, PDB CronJobs, Prometheus/Grafana/Jaeger observability, and NetworkPolicy.
- CI workflows updated to use `init-dynamo-builder` for all image builds. Canary routing allowed shadowing builds to the new infrastructure before full cutover.

## Phase 3 — Cache Routing (Feb 2026)

**Dynamo PR**: [#6480](https://github.com/ai-dynamo/dynamo/pull/6480)

- Replaced modulo-3 routing (wasted capacity at non-multiple-of-3 pod counts) with Coverage-Aware Ranked Rendezvous Hashing.
- Introduced flavor-to-group mapping and 100% pod utilization guarantee.
- Comprehensive test suite (`test_route_buildkit.sh`) covering hash quality, pool sizing, cache affinity, stability, utilization, and edge cases.

## Phase 4 — BuildKit Reliability Fixes (Mar 2026)

**Dynamo PRs**: [#7397](https://github.com/ai-dynamo/dynamo/pull/7397), [#7398](https://github.com/ai-dynamo/dynamo/pull/7398), [#7522](https://github.com/ai-dynamo/dynamo/pull/7522)

- Replaced DinD in compliance scan with BuildKit filesystem extraction (PR #7397).
- Added `builder-refresher` action to detect and recover stale BuildKit connections between the framework build and test image build steps (PR #7398).
- Added address refresh between builds to avoid stale DNS entries (PR #7522).

## Phase 5 — ECR Mirror Registry (Mar 2026)

**Linear**: [OPS-3435](https://linear.app/nvidia/issue/OPS-3435)
**Velonix PR**: [#282](https://github.com/ai-dynamo/velonix/pull/282)

- Upgraded BuildKit to v0.28.0-custom with the ECR credential helper baked in.
- Configured native BuildKit mirror support for `docker.io`, `quay.io`, and `nvcr.io`.
- Added EKS Pod Identity IAM bindings and `skopeo.sh` tooling for `nvcr.io` image synchronization.

## Phase 6 — Nightly CI Isolation (Mar 2026)

**Linear**: [OPS-4120](https://linear.app/nvidia/issue/OPS-4120)
**Dynamo PR**: [#7678](https://github.com/ai-dynamo/dynamo/pull/7678)

- Replaced `no_cache: true` with `fresh_builder: true` in nightly CI.
- Introduced the `create-fresh-builder` / `clean-k8s-builder` job frame.

## Phase 7 — GC Tuning (Mar–Apr 2026)

**Linear**: [OPS-3862](https://linear.app/nvidia/issue/OPS-3862)
**Velonix PRs**: [#259](https://github.com/ai-dynamo/velonix/pull/259), [#268](https://github.com/ai-dynamo/velonix/pull/268), [#269](https://github.com/ai-dynamo/velonix/pull/269) (in review)

- Resolved arm64 pod evictions caused by BuildKit's GC thresholds being too close to kubelet's eviction watermarks.
- Added `ephemeral-storage` requests/limits to pods and re-tuned `reservedSpace`, `maxUsedSpace`, and `minFreeSpace`.

## Future Work

The following optimizations were identified during the original design process and remain on the roadmap:

- **Zstd layer compression**: Enable `compression=zstd,force-compression=true` in BuildKit for reduced layer transfer times during push/pull.
- **mTLS with cert-manager**: Add automated mutual TLS between GitHub Actions runners and BuildKit pods for in-transit encryption of build contexts and layer data.
- **Karpenter integration**: Migrate node group autoscaling from AWS ASGs to Karpenter for faster node provisioning and more granular instance-type selection for BuildKit workloads.
- **Native multi-arch manifests**: Fully transition production artifacts to unified multi-arch manifests (ARM64/AMD64), removing the need for separate architecture-specific image tags.

# Related Proposals

* [DEP-0007](./0007-velonix-cicd-solution.md) — Velonix K8s CI/CD infrastructure (ARC, EKS, External Secrets)
* [DEP-0010](./0010-container-strategy.md) — Dynamo container build process and Dockerfile structure
* [DEP-0012](./0012-docker-templating.md) — Docker templating system for Dynamo Dockerfiles

# Alternate Solutions

## Alt 1 — BuildKit DinD per Runner

Run BuildKit as a Docker-in-Docker daemon on each GitHub Actions runner.

**Pros:**
* Simple setup — no Kubernetes infrastructure required
* No routing or pod-management complexity

**Cons:**
* Every runner has an isolated cache; no sharing across parallel jobs or workflows
* DinD has security implications (privileged containers or bind-mounts of the Docker socket)
* Cache is lost when the runner pod terminates

**Reason Rejected:** Cache isolation negates the primary benefit. Each CI run would pay the full cold-build penalty. DinD is also deprecated upstream.

## Alt 2 — Registry-Based Caching Only (no remote BuildKit)

Use `--cache-from type=registry` and `--cache-to type=registry` with a dedicated ECR cache repository instead of persistent BuildKit pods.

**Pros:**
* Simpler infrastructure — no StatefulSets or pod routing
* Survives runner restarts natively (cache lives in the registry)
* Works with any Docker client

**Cons:**
* Registry-based cache requires pushing the full layer manifest on every build, adding significant push overhead to build time
* No control over which layers are actually reused — cannot guarantee affinity between builds sharing base layers
* Does not eliminate DinD

**Reason Rejected:** Push overhead offsets a significant portion of the cache benefit, especially for large CUDA-based images. The remote daemon model gives faster cache reuse with no push cost. This was also validated empirically — ECR registry caching was already in use before this project and provided only marginal improvement.

## Alt 3 — NGINX Load Balancer with Consistent Hashing

Deploy an NGINX reverse proxy in front of the BuildKit StatefulSet, configured with consistent hashing by branch name and container flavor.

**Pros:**
* Standard load-balancer pattern
* Branch-to-pod affinity is enforced at the network layer, not the application layer

**Cons:**
* Adds another infrastructure component and potential single point of failure
* Requires managing TLS termination and NGINX configuration in IaC
* Consistent hashing at the LB layer cannot encode the flavor-to-group mapping (shared base layer awareness) that the routing script implements

**Reason Rejected:** The client-side rendezvous hashing routing script achieves the same cache-affinity goal without the additional infrastructure. The routing script also encodes semantic knowledge of which flavors share base layers, which an LB cannot.

## Alt 4 — GitHub Actions Cache (`actions/cache`)

Cache the Docker layer tarball to GitHub's built-in cache storage.

**Pros:**
* Zero additional infrastructure
* Natively integrated with GitHub Actions

**Cons:**
* 10 GB cache limit per repository branch — far below the size of a single CUDA framework image
* Cache is not shared across branches or PRs
* No multi-architecture cache coordination

**Reason Rejected:** Cache size limit is a hard blocker for Dynamo's image sizes (typically 15–40 GB per build).

# Background

## References

**Design documents (internal)**

* [Dynamo CI/CD: Accelerating Container Builds with Remote BuildKit](https://docs.google.com/document/d/1YETEYPE_wBPpfFjPH1xV_qiEqIPQysyQe8_92HJM72E) — original design doc and POC results
* [Dynamo CI: Centralized BuildKit Builders](https://docs.google.com/presentation/d/1llqJzfzEFnXj76e7j3LF14CLBaVCWKFwIJJxPHlwtyg) — rollout presentation
* [Dynamo CI: Dynamo-Builder Rollout](https://docs.google.com/presentation/d/1y_mWJ327UTEJ-H4WRUoIbNE03n8YlvGX2CEUwIXjXYA) — rollout presentation (updated)
* [Dynamo Builder Postmortem](https://docs.google.com/document/d/1Lhsmo0lGXIR4Oumm2FfMS6l5s5pO50OMUXa0Wv16N9A) — incident postmortem for arm64 pod evictions (OPS-3862)

**External**

* [BuildKit documentation](https://github.com/moby/buildkit)
* [KEDA documentation](https://keda.sh)
* [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
* [Amazon ECR pull-through cache](https://docs.aws.amazon.com/AmazonECR/latest/userguide/pull-through-cache.html)
* [EKS Pod Identities](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
* [ECR credential helper](https://github.com/awslabs/amazon-ecr-credential-helper)
* [Karpenter](https://karpenter.sh)

## Terminology & Definitions

**BuildKit**: A next-generation container image builder backend (`moby/buildkit`). Replaces the legacy `docker build` daemon with a daemonized `buildkitd` that supports concurrent builds, cross-architecture builds, and advanced cache management.

**Cache Affinity**: The property of consistently routing builds that share common base layers to the same BuildKit pod, maximizing Docker layer-cache reuse.

**Coverage-Aware Ranked Rendezvous Hashing**: A pod-routing algorithm that combines SHA-256 consistent hashing with a round-robin coverage pass to guarantee 100% pod utilization and flavor-to-pod affinity simultaneously.

**dynamo-builder**: The Helm chart (and deployed StatefulSet workload) that provides the remote BuildKit fleet in the Velonix Kubernetes cluster.

**ECR Pull-Through Cache**: An AWS ECR feature that transparently proxies and caches layers from upstream public registries (Docker Hub, Quay) on first access.

**KEDA**: Kubernetes-based Event Driven Autoscaler. Used to scale the BuildKit pod fleet based on cron schedules and Prometheus queue metrics.

**Rendezvous Hashing**: A consistent hashing scheme where each item is assigned to the candidate that produces the highest hash score, minimizing reassignments on pool size changes.

**StatefulSet**: A Kubernetes workload controller that provides stable, ordered pod names and persistent identity — used here to give BuildKit pods stable hostnames for the routing algorithm.

## Acronyms & Abbreviations

**ARC**: Actions Runner Controller

**ASG**: Auto Scaling Group (AWS)

**CI**: Continuous Integration

**DEP**: Design Enhancement Proposal

**DinD**: Docker-in-Docker

**ECR**: Elastic Container Registry (AWS)

**EKS**: Elastic Kubernetes Service (AWS)

**GC**: Garbage Collection

**HA**: High Availability

**IAM**: Identity and Access Management

**IaC**: Infrastructure as Code

**KEDA**: Kubernetes-based Event Driven Autoscaler

**mTLS**: mutual Transport Layer Security

**NAT**: Network Address Translation

**PDB**: PodDisruptionBudget

**PVC**: PersistentVolumeClaim

**RBAC**: Role-Based Access Control

**SBOM**: Software Bill of Materials
