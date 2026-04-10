# DynamoGraphDeployment API Graduation to v1beta1

**Status**: Draft

**Authors**: [jmancuso](https://github.com/jmancuso)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Graduate the DynamoGraphDeployment (DGD) and DynamoComponentDeployment (DCD) CRD APIs from `v1alpha1` to `v1beta1`. The new version replaces ten per-service fields with a single `podTemplate` (`corev1.PodTemplateSpec`), converts `spec.services` from a map to a list, removes deprecated and redundant fields (`dynamoNamespace`, `autoscaling`, `ingress`, `subComponentType`), adopts Kubernetes naming conventions (`envs` to `env`), and simplifies custom types (`Resources` to native `corev1.ResourceRequirements`, `SharedMemorySpec` to `*resource.Quantity`). A hub-and-spoke conversion webhook ensures lossless round-trip between `v1alpha1` and `v1beta1`, following the pattern already established by the DynamoGraphDeploymentRequest (DGDR) CRD. DCD graduates alongside DGD since both share `DynamoComponentDeploymentSharedSpec` and the DGD controller creates DCD objects directly.

# Motivation

The current `v1alpha1` DGD API was designed for rapid iteration during early development. As the operator has matured, several API pain points have emerged:

1. **Fragmented pod configuration.** Users must scatter pod-level settings across `resources`, `envs`, `envFromSecret`, `livenessProbe`, `readinessProbe`, `volumeMounts`, `extraPodMetadata`, `extraPodSpec`, `annotations`, and `labels`. The `extraPodSpec` field uses an unusual inline `*corev1.PodSpec` plus `MainContainer` pattern with fragile `mergo.WithOverride` merge semantics. Capabilities like `startupProbe`, `volumeDevices`, `subPath`, sidecars, and init containers are only reachable through this fragile `extraPodSpec` escape hatch.

2. **Custom types that duplicate Kubernetes-native types.** The `Resources` type uses string fields (`cpu`, `memory`, `gpu`) with a custom `GPUType` field, requiring a ~90-line `GetResourcesConfig()` conversion layer to produce `corev1.ResourceRequirements`. The `VolumeMount` type exposes only 3 of 7+ native fields. The `SharedMemorySpec` struct uses an awkward `Disabled: true` pattern. These custom types increase the learning curve for Kubernetes-experienced users and bloat the operator's conversion logic.

3. **Deprecated fields still present in the schema.** `dynamoNamespace` is always overwritten by `ComputeDynamoNamespace()` and was never user-settable. `autoscaling` is ignored at runtime, replaced by DGDSA + HPA/KEDA/Planner. Both generate validation warnings but remain in the CRD schema, confusing users.

4. **Map-based services.** `spec.services` is `map[string]*DynamoComponentDeploymentSharedSpec`. Go map iteration is non-deterministic, which complicates spec-hash computation for change detection (requiring explicit `SortKeys()` workarounds), makes test assertions fragile, and makes debugging diffs harder to reason about. The `serviceName` field on the shared spec is redundant with the map key and is always overwritten when generating DCDs.

5. **Naming inconsistencies.** The field `envs` diverges from the Kubernetes convention of singular `env` used universally in `pod.spec.containers[].env`.

Graduating to `v1beta1` addresses all of the above while establishing the multi-version CRD infrastructure (conversion webhooks, hub/spoke model) needed for future API evolution.

## Goals

* Replace fragmented pod configuration fields with a single `podTemplate` (`corev1.PodTemplateSpec`) that gives users access to the full Kubernetes pod API surface.

* Eliminate custom resource types (`Resources`, `VolumeMount`, `SharedMemorySpec`) in favor of Kubernetes-native equivalents, removing conversion layers.

* Remove deprecated and unused fields (`dynamoNamespace`, `autoscaling`, `ingress`, `subComponentType`) from the API surface.

* Convert `spec.services` from a map to a list with an explicit `name` field for deterministic ordering.

* Adopt Kubernetes naming conventions (`envs` to `env`).

* Implement hub-and-spoke conversion webhooks for lossless v1alpha1/v1beta1 round-trip, following the existing DGDR pattern.

* Graduate DCD alongside DGD to v1beta1 so both CRDs share the same `DynamoComponentDeploymentSharedSpec` type and the DGD controller can create DCD objects without a translation layer.

* Migrate all controller logic to operate on v1beta1 types (the hub).

### Non Goals

* Graduating the DynamoGraphDeploymentRequest (DGDR) CRD -- this is already at v1beta1.

* Operator-internal refactoring (directory structure, file decomposition, debug mode removal, config-vs-flags) -- these are orthogonal improvements tracked separately.

* Non-GPU E2E test infrastructure -- tracked separately.

## Requirements

### REQ 1 Lossless Round-Trip Conversion

Conversion between `v1alpha1` and `v1beta1` **MUST** be lossless. Converting `v1alpha1` -> `v1beta1` -> `v1alpha1` **MUST** return an object semantically equivalent to the original. Fields removed in `v1beta1` **MUST** be preserved via annotations on the `v1beta1` object so they can be restored during `ConvertFrom`. Fields present in `v1beta1` with no `v1alpha1` equivalent (e.g., `podTemplate.metadata` fields beyond `annotations` and `labels`, such as `finalizers` or `ownerReferences`) **MUST** be preserved via annotations on the `v1alpha1` object.

### REQ 2 Hub and Spoke Model

`v1beta1` **MUST** be the hub (storage version). All controller logic **MUST** operate on `v1beta1` types. `v1alpha1` **MUST** be the spoke, implementing `ConvertTo()` and `ConvertFrom()` against the `v1beta1` hub. Both versions **MUST** be served (`served: true`).

### REQ 3 Backward Compatibility

Existing `v1alpha1` DGD resources **MUST** continue to work without user intervention after an operator upgrade. The conversion webhook is invoked by the API server on any access (GET, LIST, WATCH, UPDATE) where the requested version differs from the storage version; since the controller watches v1beta1 (storage), conversion happens transparently when any client (kubectl, CI, another controller) accesses the resource via v1alpha1.

### REQ 4 PodTemplate Strategic Merge

The operator **MUST** inject its defaults into the container named `"dynamo"` within `podTemplate`. User-provided overrides in `podTemplate` **MUST** be merged using strategic merge patch semantics (merge by container name). Users **MUST** be able to add sidecars, init containers, volumes, and other pod-level configuration via `podTemplate` without any `extraPodSpec`-style escape hatch.

### REQ 5 v1alpha1 Deprecation Policy

`v1alpha1` **MUST** remain served (`served: true`) for at least one minor release cycle after `v1beta1` is introduced, following the project's N-1 upgrade policy. `v1alpha1` **SHOULD** emit a deprecation warning (via the API server's `+kubebuilder:deprecatedversion` marker) directing users to migrate to `v1beta1`. The timeline for setting `served: false` on `v1alpha1` will be determined based on adoption metrics and user feedback.

### REQ 6 Deterministic Service Ordering

`spec.services` **MUST** be a list (`[]DynamoComponentDeploymentService`) with an explicit `name` field. During `v1alpha1` -> `v1beta1` conversion, map entries **MUST** be sorted alphabetically by key to produce a deterministic list order.

# Proposal

## Overview

The proposal introduces a `v1beta1` version of both the `DynamoGraphDeployment` (DGD) and `DynamoComponentDeployment` (DCD) CRDs with a cleaned-up API surface, following the Kubernetes [multi-version CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/) pattern and the [kubebuilder multi-version guide](https://book.kubebuilder.io/multiversion-tutorial/tutorial). The `v1beta1` version becomes the storage version and the hub for conversion. The `v1alpha1` version remains served for backward compatibility and implements spoke conversion logic. DCD graduates alongside DGD because both embed `DynamoComponentDeploymentSharedSpec` and the DGD controller creates DCD objects directly -- keeping them on different API versions would require a translation layer that reintroduces the complexity this proposal eliminates.

## v1beta1 DGD Spec

### `DynamoGraphDeploymentSpec`

| Field | Type | Change |
| :---- | :---- | :---- |
| `services` | `[]DynamoComponentDeploymentService` | **Changed** from `map[string]*DynamoComponentDeploymentSharedSpec`. Each entry has an explicit `name` field. |
| `env` | `[]corev1.EnvVar` | **Renamed** from `envs` to match k8s convention (`pod.spec.containers[].env`). |
| `pvcs` | `[]PVC` | Unchanged |
| `backendFramework` | `string` (enum) | Unchanged |
| `restart` | `*Restart` | Unchanged |
| `topologyConstraint` | `*SpecTopologyConstraint` | Unchanged |
| `annotations` | `map[string]string` | Unchanged. Propagated by the controller to all child resources (DCDs, PodCliqueSets, and pod templates). In v1beta1 the controller merges these into each service's `podTemplate.metadata.annotations`; service-level (podTemplate) values take precedence on conflict. |
| `labels` | `map[string]string` | Unchanged. Same propagation behavior as `annotations`. |

### `DynamoComponentDeploymentService`

A thin wrapper that pairs a service name with its shared spec:

```go
type DynamoComponentDeploymentService struct {
    // Name identifies the service within the graph (e.g., "vllm-worker", "frontend").
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinLength=1
    Name string `json:"name"`

    DynamoComponentDeploymentSharedSpec `json:",inline"`
}
```

Service name uniqueness within the `services` list is enforced via webhook validation (or a `+kubebuilder:validation:XValidation` CEL rule on the `services` field). In v1alpha1, the map key enforced uniqueness by construction; in v1beta1, explicit validation is required.

```go
// CEL rule on the DynamoGraphDeploymentSpec.services field:
// +kubebuilder:validation:XValidation:rule="self.all(x, self.exists_one(y, x.name == y.name))",message="service names must be unique"
```

### `DynamoComponentDeploymentSharedSpec` Field Changes

#### Removed (subsumed by `podTemplate`)

| Field | v1alpha1 Type | Reason |
| :---- | :---- | :---- |
| `annotations` | `map[string]string` | Subsumed by `podTemplate.metadata.annotations` |
| `labels` | `map[string]string` | Subsumed by `podTemplate.metadata.labels` |
| `resources` | `*Resources` | Subsumed by `podTemplate.spec.containers[].resources` using native `corev1.ResourceRequirements`. Eliminates the custom `ResourceItem` string-parsing and the ~90-line `GetResourcesConfig()` conversion layer. GPU types expressed as native resource names (e.g., `nvidia.com/gpu`). |
| `envs` | `[]corev1.EnvVar` | Subsumed by `podTemplate.spec.containers[].env` |
| `envFromSecret` | `*string` | Subsumed by `podTemplate.spec.containers[].envFrom`. Users gain support for multiple secrets, ConfigMaps, and prefix filtering via native `[]corev1.EnvFromSource`. |
| `livenessProbe` | `*corev1.Probe` | Subsumed by `podTemplate.spec.containers[].livenessProbe` |
| `readinessProbe` | `*corev1.Probe` | Subsumed by `podTemplate.spec.containers[].readinessProbe` |
| `volumeMounts` | `[]VolumeMount` | Plain mounts subsumed by `podTemplate.spec.volumes` + `podTemplate.spec.containers[].volumeMounts` using native types. Compilation cache semantics extracted to dedicated `compilationCache` field. |
| `extraPodMetadata` | `*ExtraPodMetadata` | Subsumed by `podTemplate.metadata` |
| `extraPodSpec` | `*ExtraPodSpec` | Subsumed by `podTemplate`. Eliminates the fragile `mergo.WithOverride` merge chain and the unusual inline `*corev1.PodSpec` + `MainContainer` pattern. |

#### Removed (dropped from API)

| Field | v1alpha1 Type | Reason |
| :---- | :---- | :---- |
| `dynamoNamespace` | `*string` | Deprecated. Always overwritten by `ComputeDynamoNamespace()` from k8s namespace + DGD name. Never user-settable. |
| `autoscaling` | `*Autoscaling` | Deprecated and ignored at runtime. Replaced by DGDSA + HPA/KEDA/Planner. |
| `ingress` | `*IngressSpec` | Removed from per-service config. Ingress/VirtualService creation is driven by operator-level config and auto-defaulted for frontend components. |
| `subComponentType` | `string` | Removed. Only used to set the pod label `nvidia.com/dynamo-sub-component-type`. Users who need this label can set it directly via `podTemplate.metadata.labels`. |
| `serviceName` | `string` | Removed from shared spec. Only meaningful for standalone DCDs. In DGD context, always overwritten from the service list entry's `Name` field. |

#### Added

| Field | Type | Reason |
| :---- | :---- | :---- |
| `podTemplate` | `*corev1.PodTemplateSpec` | Single Kubernetes-native field replacing the 10 removed fields above. Operator injects defaults into the container named `"dynamo"` and merges user overrides via strategic merge by name. Also gives users access to capabilities previously only reachable via the fragile `extraPodSpec` escape hatch: `startupProbe`, `volumeDevices`, `subPath`, sidecars, init containers. |
| `compilationCache` | `*CompilationCacheConfig` | Extracted from `VolumeMount.UseAsCompilationCache`. References a PVC by name; operator handles backend-specific mount paths and env vars. Clean separation of Dynamo-specific semantics from native volume config. |

#### Modified

| Field | v1alpha1 | v1beta1 | Reason |
| :---- | :---- | :---- | :---- |
| `sharedMemory` -> `sharedMemorySize` | `*SharedMemorySpec` (struct with `Disabled bool` + `Size`) | `*resource.Quantity` | Simpler API. `nil` = default 8Gi, set value = custom size, `"0"` = disabled. Eliminates the awkward `Disabled: true` pattern where the rest of the struct is ignored. |

#### Kept (unchanged)

| Field | Type | Reason |
| :---- | :---- | :---- |
| `componentType` | `string` | User-specified per service. Drives port mapping, frontend detection, planner RBAC. No native k8s equivalent. Whether to add strict kubebuilder enum validation in v1beta1 is deferred to implementation. |
| `globalDynamoNamespace` | `bool` | Per-service opt-in to global Dynamo namespace. |
| `modelRef` | `*ModelReference` | Per-service model reference for headless service discovery. |
| `replicas` | `*int32` | Per-service, optionally managed by DGDSA. |
| `multinode` | `*MultinodeSpec` | Per-service multinode configuration. |
| `scalingAdapter` | `*ScalingAdapter` | Per-service DGDSA toggle. |
| `eppConfig` | `*EPPConfig` | Per-service EPP configuration. |
| `checkpoint` | `*ServiceCheckpointConfig` | Per-service checkpoint configuration. |
| `topologyConstraint` | `*TopologyConstraint` | Per-service topology constraint. |
| `frontendSidecar` | `*FrontendSidecarSpec` | Per-service frontend sidecar configuration. Kept for now; may be subsumed by `podTemplate` in a future revision (see Deferred to Implementation). |

### New Types

```go
// CompilationCacheConfig configures a PVC-backed compilation cache for a service.
// The operator handles backend-specific mount paths and environment variables.
type CompilationCacheConfig struct {
    // PVCName references a PVC defined in the top-level pvcs list.
    // +kubebuilder:validation:Required
    PVCName string `json:"pvcName"`

    // MountPath overrides the backend-specific default mount path.
    // +optional
    MountPath string `json:"mountPath,omitempty"`
}
```

### Status Changes

`DynamoGraphDeploymentStatus` is unchanged between `v1alpha1` and `v1beta1`. The `State` field is already typed as `DGDState` with `+kubebuilder:validation:Enum=initializing;pending;successful;failed` and `observedGeneration` is already present.

`DynamoComponentDeploymentStatus` is also unchanged between `v1alpha1` and `v1beta1`.

## Conversion Strategy

The conversion follows the [hub-and-spoke model](https://book.kubebuilder.io/multiversion-tutorial/conversion-concepts). `v1beta1` is the hub. `v1alpha1` implements `ConvertTo()` (spoke -> hub) and `ConvertFrom()` (hub -> spoke) on `DynamoGraphDeployment`.

This is the same pattern used by `DynamoGraphDeploymentRequest`, where `v1beta1` is the hub (`api/v1beta1/dynamographdeploymentrequest_conversion.go`) and `v1alpha1` is the spoke (`api/v1alpha1/dynamographdeploymentrequest_conversion.go`).

### v1alpha1 -> v1beta1 (ConvertTo)

| v1alpha1 field | v1beta1 destination | Notes |
| :---- | :---- | :---- |
| `spec.services` (map) | `spec.services` (list) | `name` from map key; sorted alphabetically |
| `spec.envs` | `spec.env` | Rename |
| service `annotations`/`labels` | `podTemplate.metadata.annotations`/`labels` | |
| service `resources` | `podTemplate.spec.containers["dynamo"].resources` | Convert `ResourceItem` strings to `resource.Quantity` |
| service `envs` | `podTemplate.spec.containers["dynamo"].env` | |
| service `envFromSecret` | `podTemplate.spec.containers["dynamo"].envFrom` | Wrap in `EnvFromSource` |
| service `livenessProbe`/`readinessProbe` | `podTemplate.spec.containers["dynamo"]` probes | Direct copy |
| service `volumeMounts` (non-compilation-cache) | `podTemplate.spec.volumes` + `podTemplate.spec.containers["dynamo"].volumeMounts` | Convert custom to native |
| service `volumeMounts` (compilation cache) | `compilationCache` | Extract |
| service `sharedMemory` | `sharedMemorySize` | `Disabled=true` -> `"0"`, else copy `Size` |
| service `extraPodMetadata` | merge into `podTemplate.metadata` | |
| service `extraPodSpec` | merge into `podTemplate.spec` / containers | |
| service `dynamoNamespace` | annotation `nvidia.com/dgd-dynamo-namespace` | Round-trip only |
| service `autoscaling` | annotation `nvidia.com/dgd-autoscaling` | Round-trip only |
| service `ingress` | annotation `nvidia.com/dgd-ingress` | Round-trip only |
| service `subComponentType` | annotation `nvidia.com/dgd-sub-component-type` | Round-trip only |
| service `serviceName` | annotation `nvidia.com/dgd-service-name` | Round-trip only |

### `extraPodSpec` Conversion Detail

The v1alpha1 `ExtraPodSpec` type is an inline `*corev1.PodSpec` (carrying pod-level fields like `tolerations`, `serviceAccountName`, `nodeSelector`, etc.) plus a separate `MainContainer` (`*corev1.Container`). During `ConvertTo`:

* Pod-level fields from `ExtraPodSpec.PodSpec` map to `podTemplate.spec` (e.g., `tolerations`, `nodeSelector`).
* `MainContainer` fields map to `podTemplate.spec.containers["dynamo"]` (e.g., `startupProbe`, `volumeDevices`).
* Additional containers in `ExtraPodSpec.PodSpec.Containers` map to additional entries in `podTemplate.spec.containers`.

When both a dedicated v1alpha1 field and `extraPodSpec` provide the same data (e.g., `envs` and `extraPodSpec.mainContainer.env`), the converter merges them into the single `podTemplate` field.

**Reverse direction (v1beta1 -> v1alpha1):** The converter cannot distinguish which `podTemplate` fields originated from `envs` vs. `extraPodSpec.mainContainer.env`, or `readinessProbe` vs. `extraPodSpec.mainContainer.readinessProbe`. The convention is: all fields from the `"dynamo"` container that have a dedicated v1alpha1 field (probes, env, resources, etc.) are placed into that dedicated field. Any remaining fields on the `"dynamo"` container that have no dedicated v1alpha1 field (e.g., `startupProbe`, `volumeDevices`) go into `extraPodSpec.mainContainer`. Pod-level fields not mapped by the converter go into `extraPodSpec.PodSpec`. This means `extraPodSpec.mainContainer.env` will be empty on the round-trip back -- all env vars end up in `envs`. This is an acceptable lossy rewrite of the v1alpha1 shape because the semantic result is identical. Most v1beta1 `podTemplate` content can be decomposed into v1alpha1 fields -- sidecars map to `extraPodSpec.PodSpec.Containers`, init containers to `extraPodSpec.PodSpec.InitContainers`, and `startupProbe` to `extraPodSpec.mainContainer.startupProbe`. The annotation `nvidia.com/dgd-pod-template-extra` is a safety net for `podTemplate` content that v1alpha1 cannot represent, such as `podTemplate.metadata` fields beyond `annotations` and `labels` (which are all that `ExtraPodMetadata` supports -- e.g., `finalizers`, `ownerReferences`).

### v1beta1 -> v1alpha1 (ConvertFrom)

Reverse of the above. Structured `v1beta1` fields are decomposed back into the `v1alpha1` shape as described in the `extraPodSpec` section. Fields present in `v1beta1` with no `v1alpha1` equivalent are preserved via annotation `nvidia.com/dgd-pod-template-extra` for round-trip.

### DCD Conversion

DCD conversion follows the same shared-spec field mapping but is simpler -- each DCD represents a single service, so round-trip annotations store values directly rather than as per-service JSON maps. The `BackendFramework` field is unchanged between versions. `ServiceName` is preserved via annotation in v1beta1 (removed from the shared spec) for round-trip.

### Volume Mount Handling

In `v1alpha1`, the custom `VolumeMount` type lets users write `name: model-store, mountPoint: /models` and the operator auto-creates both the `corev1.Volume` (with `persistentVolumeClaim.claimName` resolved from the top-level `pvcs` list) and the `corev1.VolumeMount`. In `v1beta1`, users specify both the `podTemplate.spec.volumes` entry and the `podTemplate.spec.containers[].volumeMounts` entry using native Kubernetes types. This is more verbose for simple PVC mounts but is the standard Kubernetes pattern and gives users full control over volume configuration (subPath, readOnly, mountPropagation, etc.) without escape hatches. The top-level `pvcs` list continues to handle PVC creation; users reference the created PVCs by name in their `podTemplate`.

**PVC reference validation:** In v1alpha1, the operator implicitly coupled volume mount names to the `pvcs` list during graph generation -- there was no admission-time cross-validation. In v1beta1, `podTemplate.spec.volumes` can reference any PVC (including pre-existing PVCs not managed by the `pvcs` list), so no cross-validation is imposed. However, `compilationCache.pvcName` is tightly coupled to the `pvcs` list (the operator needs to manage backend-specific mount paths), so webhook validation **MUST** verify that `compilationCache.pvcName` references a PVC defined in the top-level `pvcs` list.

## Controller Migration

All controller logic (DGD reconciler, DCD reconciler, graph generation, webhooks) migrates from `v1alpha1` to `v1beta1` types. Because DCD also graduates, `graph.go` generates v1beta1 DCD objects from v1beta1 DGD specs using the same `DynamoComponentDeploymentSharedSpec` type -- no cross-version translation is needed.

* **Service iteration** changes from `for name, component := range spec.Services` (map) to `for _, svc := range spec.Services` (list) with `svc.Name`.

* **`GenerateBasePodSpec`** in `graph.go` changes from assembling a pod spec from individual fields to merging the user's `podTemplate` with operator defaults using strategic merge. The operator injects its defaults into the container named `"dynamo"`.

* **`GetResourcesConfig()`** (~90 lines in `controller_common/resource.go`) is eliminated. Resources come directly from `podTemplate.spec.containers[].resources` as native `corev1.ResourceRequirements`.

* **Shared memory** changes from `ApplySharedMemoryVolumeAndMount(&podSpec, &container, component.SharedMemory)` to reading `*resource.Quantity` directly.

* **Ingress** logic in the DGD controller (`reconcileGroveResources`) drops the `component.Ingress != nil` override path. Ingress is driven solely by operator config (`r.Config.Ingress`).

## User Migration Guide

**Existing v1alpha1 users: no action required.** Your deployed DGD resources continue to work without modification. The conversion webhook transparently converts between v1alpha1 and v1beta1 on every API access, and all controller logic operates on the v1beta1 storage version. Existing resources stored in etcd are migrated to v1beta1 format on their next write (e.g., when the controller reconciles them or a user updates them).

**New deployments should use v1beta1.** The v1alpha1 API is deprecated and will emit a warning when used. v1alpha1 remains fully functional for at least one minor release cycle (see REQ 5).

**Migrating YAML from v1alpha1 to v1beta1** -- the primary user-visible changes are:

1. `spec.services` changes from a map to a list with an explicit `name` field.
2. `spec.envs` is renamed to `spec.env`.
3. Per-service pod configuration (`resources`, `envs`, `envFromSecret`, probes, `volumeMounts`, `annotations`, `labels`, `extraPodSpec`, `extraPodMetadata`) is consolidated into a single `podTemplate` field using native Kubernetes types.
4. `sharedMemory: { size: 16Gi }` becomes `sharedMemorySize: 16Gi`.
5. Compilation cache volume mounts become `compilationCache: { pvcName: <name> }`.
6. Volume mounts use native Kubernetes `podTemplate.spec.volumes` + `podTemplate.spec.containers[].volumeMounts` instead of the custom shorthand.

See the [v1alpha1 vs v1beta1 example](#example-v1alpha1-vs-v1beta1) for a complete side-by-side comparison.

## Example: v1alpha1 vs v1beta1

The following examples show the same DGD expressed in both API versions. This is a disaggregated vLLM deployment with a frontend, a prefill worker, and a decode worker.

### v1alpha1

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
  namespace: inference
spec:
  backendFramework: vllm
  envs:
    - name: HF_TOKEN
      valueFrom:
        secretKeyRef:
          name: hf-secret
          key: token
  pvcs:
    - name: model-store
      create: true
      size: 100Gi
      storageClass: fast-ssd
      volumeAccessMode: ReadWriteOnce
    - name: compile-cache
      create: true
      size: 50Gi
      storageClass: fast-ssd
      volumeAccessMode: ReadWriteOnce
  services:
    frontend:
      componentType: frontend
      replicas: 2
      resources:
        requests:
          cpu: "4"
          memory: 8Gi
        limits:
          cpu: "8"
          memory: 16Gi
      envs:
        - name: LOG_LEVEL
          value: info
      livenessProbe:
        httpGet:
          path: /health
          port: 8000
        initialDelaySeconds: 30
      readinessProbe:
        httpGet:
          path: /ready
          port: 8000
      ingress:
        enabled: true
        host: vllm.example.com
        ingressControllerClassName: nginx
      annotations:
        prometheus.io/scrape: "true"
    prefill-worker:
      componentType: worker
      subComponentType: prefill
      replicas: 2
      resources:
        requests:
          cpu: "16"
          memory: 64Gi
          gpu: "4"
        limits:
          cpu: "32"
          memory: 128Gi
          gpu: "4"
      envs:
        - name: WORKER_MODE
          value: prefill
      envFromSecret: worker-config
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-store
          mountPoint: /models
        - name: compile-cache
          useAsCompilationCache: true
      extraPodMetadata:
        labels:
          app.kubernetes.io/component: prefill
      extraPodSpec:
        tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
        mainContainer:
          startupProbe:
            httpGet:
              path: /health
              port: 8001
            failureThreshold: 30
            periodSeconds: 10
      scalingAdapter:
        enabled: true
    decode-worker:
      componentType: worker
      subComponentType: decode
      replicas: 4
      resources:
        requests:
          cpu: "16"
          memory: 64Gi
          gpu: "2"
        limits:
          cpu: "32"
          memory: 128Gi
          gpu: "2"
      envs:
        - name: WORKER_MODE
          value: decode
      envFromSecret: worker-config
      sharedMemory:
        size: 16Gi
      volumeMounts:
        - name: model-store
          mountPoint: /models
        - name: compile-cache
          useAsCompilationCache: true
      extraPodSpec:
        tolerations:
          - key: nvidia.com/gpu
            operator: Exists
            effect: NoSchedule
      scalingAdapter:
        enabled: true
```

### v1beta1

```yaml
apiVersion: nvidia.com/v1beta1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
  namespace: inference
spec:
  backendFramework: vllm
  env:
    - name: HF_TOKEN
      valueFrom:
        secretKeyRef:
          name: hf-secret
          key: token
  pvcs:
    - name: model-store
      create: true
      size: 100Gi
      storageClass: fast-ssd
      volumeAccessMode: ReadWriteOnce
    - name: compile-cache
      create: true
      size: 50Gi
      storageClass: fast-ssd
      volumeAccessMode: ReadWriteOnce
  services:
    - name: frontend
      componentType: frontend
      replicas: 2
      podTemplate:
        metadata:
          annotations:
            prometheus.io/scrape: "true"
        spec:
          containers:
            - name: dynamo
              resources:
                requests:
                  cpu: "4"
                  memory: 8Gi
                limits:
                  cpu: "8"
                  memory: 16Gi
              env:
                - name: LOG_LEVEL
                  value: info
              livenessProbe:
                httpGet:
                  path: /health
                  port: 8000
                initialDelaySeconds: 30
              readinessProbe:
                httpGet:
                  path: /ready
                  port: 8000

    - name: prefill-worker
      componentType: worker
      replicas: 2
      sharedMemorySize: 16Gi
      compilationCache:
        pvcName: compile-cache
      scalingAdapter:
        enabled: true
      podTemplate:
        metadata:
          labels:
            app.kubernetes.io/component: prefill
        spec:
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
          volumes:
            - name: model-store
              persistentVolumeClaim:
                claimName: model-store
          containers:
            - name: dynamo
              resources:
                requests:
                  cpu: "16"
                  memory: 64Gi
                  nvidia.com/gpu: "4"
                limits:
                  cpu: "32"
                  memory: 128Gi
                  nvidia.com/gpu: "4"
              env:
                - name: WORKER_MODE
                  value: prefill
              envFrom:
                - secretRef:
                    name: worker-config
              volumeMounts:
                - name: model-store
                  mountPath: /models
              startupProbe:
                httpGet:
                  path: /health
                  port: 8001
                failureThreshold: 30
                periodSeconds: 10

    - name: decode-worker
      componentType: worker
      replicas: 4
      sharedMemorySize: 16Gi
      compilationCache:
        pvcName: compile-cache
      scalingAdapter:
        enabled: true
      podTemplate:
        spec:
          tolerations:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
          volumes:
            - name: model-store
              persistentVolumeClaim:
                claimName: model-store
          containers:
            - name: dynamo
              resources:
                requests:
                  cpu: "16"
                  memory: 64Gi
                  nvidia.com/gpu: "2"
                limits:
                  cpu: "32"
                  memory: 128Gi
                  nvidia.com/gpu: "2"
              env:
                - name: WORKER_MODE
                  value: decode
              envFrom:
                - secretRef:
                    name: worker-config
              volumeMounts:
                - name: model-store
                  mountPath: /models
```

### Key Differences Highlighted

| Aspect | v1alpha1 | v1beta1 |
| :---- | :---- | :---- |
| Services | `services:` map keyed by name | `services:` list with explicit `name` field |
| Graph-level env | `envs:` | `env:` |
| Resources | Custom `resources.requests.gpu: "4"` with string values | Native `nvidia.com/gpu: "4"` in `podTemplate.spec.containers[].resources` |
| Per-service env | `envs:` on shared spec | `podTemplate.spec.containers["dynamo"].env` |
| Secret env | `envFromSecret: worker-config` (single string) | `podTemplate.spec.containers["dynamo"].envFrom` (native list, supports multiple sources) |
| Probes | Top-level `livenessProbe` / `readinessProbe` | `podTemplate.spec.containers["dynamo"]` probes (including `startupProbe` natively) |
| Shared memory | `sharedMemory: { size: 16Gi }` | `sharedMemorySize: 16Gi` |
| Compilation cache | `volumeMounts: [{ name: compile-cache, useAsCompilationCache: true }]` | `compilationCache: { pvcName: compile-cache }` |
| Volume mounts | Custom `volumeMounts: [{ name: model-store, mountPoint: /models }]` | Native `podTemplate.spec.volumes` + `podTemplate.spec.containers[].volumeMounts` |
| Tolerations | `extraPodSpec: { tolerations: [...] }` | `podTemplate.spec.tolerations` |
| Startup probe | `extraPodSpec: { mainContainer: { startupProbe: {...} } }` | `podTemplate.spec.containers["dynamo"].startupProbe` |
| Pod labels | `extraPodMetadata: { labels: {...} }` | `podTemplate.metadata.labels` |
| Pod annotations | Top-level `annotations:` | `podTemplate.metadata.annotations` |
| Ingress | `ingress: { enabled: true, host: ... }` | Removed -- driven by operator config |
| SubComponentType | `subComponentType: prefill` | Removed -- use `podTemplate.metadata.labels` if needed |

# Implementation Details

## CRD Versioning Configuration

The generated CRD will declare both versions:

```yaml
spec:
  versions:
    - name: v1alpha1
      served: true
      storage: false
      schema: ...
    - name: v1beta1
      served: true
      storage: true
      schema: ...
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        service:
          name: dynamo-operator-webhook
          namespace: dynamo-operator-system
          path: /convert
```

## Annotation Keys for Round-Trip

All annotations use the `nvidia.com/dgd-` prefix to avoid collisions. Because a single DGD object can contain multiple services, per-service annotations store a JSON object keyed by service name. For example, if two services had `ingress` config in v1alpha1, the annotation would be:

```json
{"frontend": {"enabled": true, "host": "vllm.example.com"}, "worker": {"enabled": false}}
```

| Annotation Key | Value Format | Direction |
| :---- | :---- | :---- |
| `nvidia.com/dgd-dynamo-namespace` | `{"<svc>": "<ns>", ...}` | v1alpha1 -> v1beta1 |
| `nvidia.com/dgd-autoscaling` | `{"<svc>": {<autoscaling JSON>}, ...}` | v1alpha1 -> v1beta1 |
| `nvidia.com/dgd-ingress` | `{"<svc>": {<ingress JSON>}, ...}` | v1alpha1 -> v1beta1 |
| `nvidia.com/dgd-sub-component-type` | `{"<svc>": "<type>", ...}` | v1alpha1 -> v1beta1 |
| `nvidia.com/dgd-service-name` | `{"<svc>": "<name>", ...}` | v1alpha1 -> v1beta1 |
| `nvidia.com/dgd-pod-template-extra` | `{"<svc>": {<extra fields JSON>}, ...}` | v1beta1 -> v1alpha1 |

Only services that actually had the field set are included in the JSON map. Services without the field are omitted. If no service had the field set, the annotation is not created.

## Operational Readiness

### Webhook Infrastructure

The DGD/DCD conversion webhook reuses the existing webhook infrastructure established by DGDR:

* **TLS certificates**: Managed by the operator's `CertManager`, which uses the OPA cert-controller (`certrotator`) for auto-provisioning and rotation. Manual mode (externally provided certs) is also supported. No new certificate infrastructure is needed.
* **CA bundle injection**: The existing `CABundleInjector.ensureCRDConversion()` is extended to patch the DGD and DCD CRDs in addition to DGDR. The injector reads the CA bundle from the operator's webhook secret and patches the CRD conversion config at startup.
* **Webhook serving**: The conversion webhook handler is registered on the same webhook server (same service, same `/convert` path) that already serves DGDR conversion and admission webhooks.

### Failure Modes

Conversion webhooks are in the API server's critical path for all requests to the affected CRDs. If the webhook is unreachable:

* **`failurePolicy`**: Conversion webhooks do not support `failurePolicy` (unlike admission webhooks). If the webhook is down, all API access (GET, LIST, WATCH, UPDATE) for DGD/DCD resources returns an error. This is the same failure mode that already exists for DGDR.
* **Mitigation**: The webhook server starts only after certificates are confirmed ready (`CertManager.WaitReady()`). The operator pod exposes health and readiness probes so Kubernetes restarts it on failure. This is the same single-replica availability model already in production for DGDR conversion.

### Rollback Procedure

If a conversion bug is discovered after deployment:

1. **Fix forward** (preferred): Deploy a corrected operator image. Since the webhook is stateless and the conversion logic is deterministic, a fixed webhook immediately resolves the issue for all subsequent API accesses.
2. **Revert to v1alpha1-only**: If a fix-forward is not immediately possible, revert the CRD to v1alpha1-only (`storage: true`, remove v1beta1 version). Objects already stored as v1beta1 in etcd must be migrated back -- this requires a storage version migration (via the `StorageVersionMigrator` or manual `kubectl get/replace` loop). This is a last-resort option because of the etcd migration cost.

### Monitoring

Conversion webhook health is observable via standard API server metrics for CRD conversion and controller-runtime webhook server metrics (request latency, error counters). The existing operator alerting infrastructure applies without changes.

## Deferred to Implementation

* Exact strategic merge behavior for `podTemplate` -- whether to use `apimachinery` strategic merge patch or a simpler name-based container merge.

* Whether `componentType` should be a strict kubebuilder enum (`+kubebuilder:validation:Enum=frontend;worker;epp;prefill;decode`) or an extensible string with recommended values.

* Whether `frontendSidecar` should be subsumed by `podTemplate` (users can add sidecar containers natively) or kept as a convenience field.

* Whether `DYN_DEPLOYMENT_CONFIG` removal should be scoped into this effort or tracked separately.

* Semantics of list ordering in `spec.services` -- whether order is significant beyond determinism (the restart spec already has its own `order` field).

# Implementation Phases

TBD

# Alternate Solutions

TBD

# Background

## CRD Versioning in Kubernetes

Kubernetes supports [multiple versions of a CRD](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/) through conversion webhooks. Key concepts:

* **Served vs Storage**: Multiple versions can be served (accepted by the API server), but only one version is the storage version (persisted in etcd). When a client requests a non-storage version, the API server invokes the conversion webhook.

* **Round-Trip Requirement**: Conversions must be lossless. Converting `v1alpha1` -> `v1beta1` -> `v1alpha1` must return the original object. Fields that exist in one version but not the other must be preserved (typically via annotations).

* **Hub and Spoke Model**: One version is chosen as the hub (typically the newest/storage version). All other versions implement `ConvertTo` (spoke -> hub) and `ConvertFrom` (hub -> spoke). The hub type only needs a `Hub()` marker method. This is the pattern recommended by [kubebuilder](https://book.kubebuilder.io/multiversion-tutorial/conversion-concepts).

## Existing Conversion Precedent in Dynamo

The `DynamoGraphDeploymentRequest` (DGDR) CRD already has a `v1beta1` hub and `v1alpha1` spoke:

* Hub: `api/v1beta1/dynamographdeploymentrequest_conversion.go` -- `Hub()` marker
* Spoke: `api/v1alpha1/dynamographdeploymentrequest_conversion.go` -- `ConvertTo()` / `ConvertFrom()` with annotation-based round-trip preservation

The DGD graduation follows this exact pattern.

## References

* [Kubernetes: Versions in CRDs](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)
* [Kubebuilder: Multi-version Tutorial](https://book.kubebuilder.io/multiversion-tutorial/tutorial)
* [Kubebuilder: Conversion Concepts](https://book.kubebuilder.io/multiversion-tutorial/conversion-concepts)
* [RFC 2119: Key Words for Use in RFCs](https://datatracker.ietf.org/doc/html/rfc2119)

## Acronyms & Abbreviations

**CRD:** Custom Resource Definition

**DCD:** DynamoComponentDeployment

**DGD:** DynamoGraphDeployment

**DGDR:** DynamoGraphDeploymentRequest

**DGDSA:** DynamoGraphDeploymentScalingAdapter
