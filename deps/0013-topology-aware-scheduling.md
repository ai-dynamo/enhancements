# Topology Aware Scheduling Integration for Dynamo Operator

**Status**: Draft

**Authors**: [TBD]

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [TBD]

**Required Reviewers**: [TBD]

**Review Date**: [TBD]

**Pull Request**: [TBD]

**Implementation PR / Tracking Issue**: [TBD]

# Summary

Add topology-aware scheduling (TAS) support to DynamoGraphDeployment (DGD) so that users can express topology placement preferences (e.g., pack pods within a block or rack) at the deployment and service level. Users select a named topology profile (a `ClusterTopology` CR) and specify free-form topology domain names defined by the cluster admin — there is no fixed set of allowed domains. The Dynamo operator injects these user-specified constraints into the generated PodCliqueSet (PCS) spec, passing through the topology profile name so the framework resolves domains to infrastructure-specific node labels. The underlying framework (e.g., Grove) owns topology validation and enforcement; Dynamo remains framework-agnostic.

# Motivation

AI inference workloads such as disaggregated serving benefit significantly from locality-aware placement. Packing related pods within the same network block or rack reduces inter-node latency and improves throughput. Today, the Dynamo operator generates Grove PodCliqueSet resources from DGDs with no topology constraints, leaving users unable to express placement preferences through the DGD API.

Grove already provides Topology Aware Scheduling via its ClusterTopology CRD and TopologyConstraint fields on PCS, PCSG, and PodClique resources (GREP-244). Cluster admins define one or more `ClusterTopology` CRs, each mapping free-form domain names (e.g., `rack`, `block`, `zone`) to infrastructure-specific node labels. The missing piece is a user-facing API on DGD that lets users select a topology profile and specify domain-level placement preferences, which the operator translates into the correct PCS fields.

## Goals

* Allow DGD users to opt in to topology-aware scheduling via a simple, framework-agnostic API.

* Topology constraints are independently optional at the deployment level and at each service level. TAS is in effect when any topology constraint is set at any level. Services without an explicit constraint inherit from the deployment-level constraint (if set).

* Maintain full backward compatibility with existing DGDs (no TAS unless topology constraints are provided).

* Validate topology at DGD creation time (webhook) so users get immediate feedback; when any topology constraint is set, the webhook must validate against the framework's topology CRD and reject the DGD with an error if validation fails or the CRD cannot be read.

### Non Goals

* Dynamo will not define, create, or manage topology CRs (e.g., Grove ClusterTopology). Admins configure topology in the underlying framework per its documentation.

* Dynamic updates to topology constraints after PCS creation are out of scope (blocked by Grove's immutability).

* Backend- or framework-specific default strategies (e.g., different defaults for vLLM vs TRT-LLM) are out of scope for the initial implementation.

* Dynamo does not own the mapping from domain names to node labels. That translation is the responsibility of the framework (e.g., Grove maps domains to labels via the `ClusterTopology` CR). For future non-Grove backends, an operator-level translation config may be needed; this is out of scope.

## Requirements

### REQ 1 Opt-In by Providing Constraints

TAS **MUST** be opt-in: it is in effect when any topology constraint is set at any level (deployment-level, service-level, or both). If no topology constraints are set anywhere, no TAS is applied. Existing DGDs without topology constraints **MUST** continue to work unchanged after an operator upgrade.

### REQ 2 API Abstraction

The DGD API **MUST NOT** expose Grove-specific terms (PCS, PCSG, PodClique). Users **SHOULD** specify constraints in terms of deployment-level and per-service only.

### REQ 3 Independent Optional Constraints

Topology constraints are independently optional at the deployment level and at each service level. `spec.topologyConstraint` with `topologyProfile` is always required when any TAS constraint is set (it names the `ClusterTopology` CR). Users **MAY** set:
- Only `spec.topologyConstraint` with `topologyProfile` and `packDomain` (broad deployment-level constraint; services inherit).
- `spec.topologyConstraint` with `topologyProfile` only (no `packDomain`), plus per-service `topologyConstraint` with `packDomain` (targeted constraints for specific services).
- Both: `spec.topologyConstraint` with `topologyProfile` and `packDomain`, plus per-service overrides with narrower `packDomain`.

**Note on constraint type:** Today, all topology constraints in the underlying framework (Grove) are hard/required constraints — there is no "preferred" or soft mode. If the scheduler cannot satisfy the constraint, the workload will not be scheduled. This is acceptable because TAS is explicitly opt-in and users specify constraints knowingly. When the underlying framework adds support for preferred/soft constraints, the API could be extended accordingly.

### REQ 5 Hierarchy Validation

When both spec-level and service-level constraints are set, the service-level `packDomain` **MUST** be equal to or narrower than the deployment-level `packDomain`. "Narrower" is determined by the ordering of levels in the referenced `ClusterTopology` CR (`spec.levels` is ordered broadest-first; a higher index means a narrower domain). The operator **MUST** reject a DGD where a service constraint is broader than the deployment constraint.

### REQ 6 Framework Agnosticism

The high-level DGD API (`topologyConstraint`, `topologyProfile`, `packDomain`) **MUST** be the same for all supported frameworks. Domain names are **free-form strings** whose meaning is defined by the cluster admin in a `ClusterTopology` CR (or equivalent). Users express topology in whatever domain vocabulary the admin has configured (e.g., `rack`, `block`, `zone`); Dynamo does not enforce a fixed set. The DGD API does not change across frameworks — only the internal mapping from domain names to framework-specific constructs differs.

**Why domain names instead of node labels:** Node labels are cluster-specific and cloud-provider-specific (e.g., `cloud.provider.com/topology-rack` vs `topology.kubernetes.io/rack`), so DGDs written with labels would not be portable across clusters. Domain names let users express intent in natural terms ("pack in the same rack") without knowing infrastructure-level label strings. The framework's `ClusterTopology` CR maps these domain names to the actual node labels.

### REQ 7 Backward Compatibility

No automatic TAS application on existing DGDs. Users opt in by setting topology constraints at the level(s) they need; to add TAS to an existing DGD they must recreate it with constraints set.

# Proposal

## API Design

Add an optional `topologyConstraint` field to `DynamoGraphDeploymentSpec` (deployment-level) and to each `ServiceSpec` (service-level). Both use the same `TopologyConstraint` struct with contextual validation:

```go
type DynamoGraphDeploymentSpec struct {
    // ... existing fields ...

    // TopologyConstraint is the deployment-level topology constraint.
    // When set, topologyProfile is required and names the ClusterTopology CR to use.
    // packDomain is optional here — it can be omitted when only services carry constraints.
    // Services without their own topologyConstraint inherit this value.
    // +optional
    TopologyConstraint *TopologyConstraint `json:"topologyConstraint,omitempty"`
}

type ServiceSpec struct {
    // ... existing fields (name, replicas, multinode, etc.) ...

    // TopologyConstraint for this service. topologyProfile is inherited from
    // spec.topologyConstraint and must not be set here. packDomain is required.
    // When both this and spec.topologyConstraint are set, packDomain must be
    // narrower than or equal to the spec-level packDomain.
    // +optional
    TopologyConstraint *TopologyConstraint `json:"topologyConstraint,omitempty"`
}

type TopologyConstraint struct {
    // TopologyProfile is the name of the ClusterTopology CR that defines the
    // topology hierarchy for this deployment. Required at spec level when any
    // topology constraint is set. Must not be set at service level (inherited).
    // +optional
    TopologyProfile string `json:"topologyProfile,omitempty"`

    // PackDomain is the topology domain to pack pods within. Must match a
    // domain defined in the referenced ClusterTopology CR.
    // Optional at spec level (when only providing the profile for service-level
    // constraints); required at service level.
    // +optional
    PackDomain TopologyDomain `json:"packDomain,omitempty"`
}

// TopologyDomain is a free-form topology level identifier.
// Must match the regex ^[a-z0-9]+[a-z0-9-]*$ (lowercase alphanumeric, may contain hyphens).
// Domain names are defined by the cluster admin in the ClusterTopology CR.
// Common examples: "region", "zone", "datacenter", "block", "rack", "host", "numa".
type TopologyDomain string
```

**Contextual validation (same struct, different rules):**

| Field | At `spec.topologyConstraint` | At `service.topologyConstraint` |
|-------|------------------------------|----------------------------------|
| `topologyProfile` | **Required** when any TAS constraint is set | **Forbidden** (inherited from spec) |
| `packDomain` | Optional | **Required** |

**User examples:**

Broad deployment-level constraint only (services inherit):

```yaml
spec:
  topologyConstraint:
    topologyProfile: gpu-cluster-topology
    packDomain: zone
  services:
    - name: VllmWorker
      replicas: 2
    - name: Frontend
      replicas: 1
```

Mixed: deployment-level default with a per-service override:

```yaml
spec:
  topologyConstraint:
    topologyProfile: gpu-cluster-topology
    packDomain: zone
  services:
    - name: VllmWorker
      multinode: { nodeCount: 4 }
      replicas: 2
      topologyConstraint:
        packDomain: block   # narrower than zone — valid
    - name: Frontend
      replicas: 1
      # inherits zone from spec.topologyConstraint
```

Service-level only (no deployment-level `packDomain`; profile still at spec level):

```yaml
spec:
  topologyConstraint:
    topologyProfile: gpu-cluster-topology
  services:
    - name: VllmWorker
      multinode: { nodeCount: 4 }
      replicas: 2
      topologyConstraint:
        packDomain: rack    # only this service gets topology packing
    - name: Frontend
      replicas: 1
      # no topology constraint
```

**Mapping from DGD to generated spec (Grove PCS):**

| DGD | Grove | When | Value |
|-----|-------|------|--------|
| spec.topologyConstraint.topologyProfile | ClusterTopology CR name (passed through to PCS/PodGang) | Always when TAS active | User-specified profile name |
| spec.topologyConstraint.packDomain | PCS.Spec.Template.TopologyConstraint.PackDomain | when set | User-specified |
| service.topologyConstraint.packDomain | PCSG.TopologyConstraint.PackDomain | Multinode svc, when set | User-specified |
| service.topologyConstraint.packDomain | PodClique.TopologyConstraint.PackDomain | Single-node svc, when set | User-specified |
| — | PodClique (multinode) | Always | No explicit constraint (inherits from PCSG or PCS) |
| — | PodClique/PCSG | Service has no constraint | No explicit constraint (inherits from PCS template if set) |

## Constraint Resolution

Topology constraints are independently optional at every level. The controller injects whichever constraints are specified; it does not read the topology CRD at reconcile time. The webhook (see Validation) reads the topology CRD for admission-time validation.

**`topologyProfile` resolution:** The `topologyProfile` field is set once at `spec.topologyConstraint` and inherited by all services. Services **MUST NOT** set `topologyProfile`; they inherit the profile from the spec level. The profile name is passed through to the framework so it can resolve domains against the correct `ClusterTopology` CR.

**`packDomain` resolution rules:**

- **spec.topologyConstraint has `packDomain`, service has no constraint:** Service inherits from the PCS template (no explicit constraint injected on the child resource; the framework handles inheritance).
- **spec.topologyConstraint has `packDomain`, service also has `packDomain`:** Both are injected; hierarchy is validated (service must be ≤ spec-level per the CRD ordering).
- **spec.topologyConstraint has no `packDomain`, service has `packDomain`:** Only the service's constraint is injected on the child resource (PCSG or PodClique). No PCS-level `packDomain`.
- **Neither set (no `topologyConstraint` anywhere):** No topology constraints (backward compatible).

## PCS Generation

In `GenerateGrovePodCliqueSet` (or equivalent), inject the topology profile and whichever `packDomain` constraints are specified:

- **Topology profile:** The `topologyProfile` from `spec.topologyConstraint` is passed through to the generated PCS as the `ClusterTopology` name (e.g., via annotation or field, depending on the framework's API for selecting a named topology).
- **PCS template:** `packDomain` set from `spec.topologyConstraint.packDomain` (nil if not set or no `packDomain`).
- **For each service:**
  - If service has `topologyConstraint.packDomain`:
    - Multinode: set PCSG topology constraint; no explicit constraint on child cliques (they inherit from PCSG).
    - Single-node: set PodClique topology constraint.
  - If service has no `topologyConstraint`: no explicit constraint on the child resource (it inherits from PCS template if set, or has no constraint).

If the underlying framework rejects the generated resources (e.g., topology definition missing or invalid domain), the error surfaces at the framework level.

## Validation (Webhook)

### On CREATE

- **`topologyProfile` (spec level):** When any topology constraint is set (at any level), `spec.topologyConstraint` **MUST** be present and its `topologyProfile` **MUST** be a non-empty string. The webhook **MUST** validate that a `ClusterTopology` CR with this name exists and is readable. If the CR cannot be read (e.g., RBAC missing, CRD not installed, CR not found), the webhook **MUST** reject the DGD with an error.
- **`topologyProfile` (service level):** Service-level `topologyConstraint` **MUST NOT** set `topologyProfile`. If set, the webhook **MUST** reject with an error ("topologyProfile is inherited from spec.topologyConstraint and must not be set at the service level").
- **`packDomain` format:** Every `packDomain` that is set **MUST** match the regex `^[a-z0-9]+[a-z0-9-]*$` (lowercase alphanumeric, may contain hyphens after the first character).
- **`packDomain` existence:** Each `packDomain` used (at spec or service level) **MUST** exist as a `domain` in the referenced `ClusterTopology` CR's `spec.levels[]`. If a domain is not found, the webhook **MUST** reject with an error listing the invalid domain and the available domains.
- **Hierarchy:** When both `spec.topologyConstraint.packDomain` and a service's `topologyConstraint.packDomain` are set, the service's domain **MUST** be equal to or narrower than the spec-level domain. "Narrower" means the service's domain appears at an equal or higher index in the `ClusterTopology` CR's `spec.levels` array (ordered broadest-first).
- **RBAC:** The Dynamo operator needs read-only access to the framework's topology CRD (e.g., `clustertopologies.grove.io`) for webhook validation when that backend is in use.

### On UPDATE

All topology-related fields are **immutable** once the DGD is created, mirroring the framework's immutability of topology constraints on generated resources. The following changes **MUST** be rejected:

- **`spec.topologyConstraint`:** Cannot be added, removed, or have `topologyProfile` or `packDomain` changed.
- **Service `topologyConstraint`:** Cannot be added, removed, or have `packDomain` changed on any existing service.

The `ClusterTopology` CRD check is **not** re-run on UPDATE because TAS fields are immutable — domains were already validated at creation time and the topology may have changed since.

To change topology constraints, users must delete and recreate the DGD. This aligns with the framework's model and avoids update failures on generated resources.

## Framework Agnosticism and Topology Ownership

**Free-form domain vocabulary:** `TopologyDomain` is a free-form string — Dynamo does not define or restrict a fixed set of domain names. Domain names and their meaning are defined by the cluster admin when they create `ClusterTopology` CRs. Common names like `region`, `zone`, `rack`, `host` are conventional but not enforced by the type system. Dynamo's webhook validates that each `packDomain` exists as a level in the referenced `ClusterTopology` CR, catching typos and misconfiguration at admission time.

**Topology profiles and multi-topology:** The `topologyProfile` field lets users select which `ClusterTopology` CR to use. This enables clusters with multiple topology definitions (e.g., one for H100 node pools, another for GB200) — each DGD references the profile that matches its hardware target. Dynamo passes the profile name through to the framework; the framework resolves domain names to node labels using the selected topology.

**Webhook vs controller:** The webhook reads the framework's topology CRD (identified by `topologyProfile`) when any topology constraint is set and rejects the DGD with an error if the CR cannot be read or validation fails. The controller does not read the topology CRD at reconcile time; it injects whichever constraints are specified and passes the profile name through.

Dynamo does not define or create topology CRs. When Grove is used, the cluster's topology (ClusterTopology) is configured by the admin per Grove's documentation. Dynamo requires read-only RBAC for the framework's topology CRD when that backend is in use (for webhook validation).

## Status and Observability

**Topology condition on DGD status:**

When any topology constraint is set (spec-level, service-level, or both), the controller surfaces a `TopologyLevelsAvailable` condition on the DGD status. The controller reads topology-related conditions from the underlying workload resource (framework-specific) and translates them into this Dynamo-level condition:

| Condition Type | Status | Reason | When |
|----------------|--------|--------|------|
| `TopologyLevelsAvailable` | `True` | `AllTopologyLevelsAvailable` | TAS active and all required topology levels are available in the cluster topology |
| `TopologyLevelsAvailable` | `False` | `TopologyLevelsUnavailable` | The framework reports that one or more required topology levels are no longer available |
| `TopologyLevelsAvailable` | `False` | `TopologyDefinitionNotFound` | The framework reports that the topology definition resource (identified by `topologyProfile`) was not found |
| *(absent)* | — | — | No topology constraints set at any level |

This condition is necessary because the underlying framework may not fail the deployment when a topology level becomes unavailable — it may keep pods running but silently stop enforcing the constraint. Without propagating this, the DGD would appear `Ready` with no indication that topology enforcement stopped.

**Framework translation (Grove):** The controller reads the PCS `Conditions` and maps Grove's `TopologyLevelsUnavailable` condition (with reasons `ClusterTopologyLevelsUnavailable` and `ClusterTopologyNotFound`) to the `TopologyLevelsAvailable` condition above. Other frameworks would implement equivalent mappings from their topology health signals.

Example DGD status when a topology level becomes unavailable:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
    - type: TopologyLevelsAvailable
      status: "False"
      reason: TopologyLevelsUnavailable
      message: "Topology level 'rack' is no longer available in the cluster topology"
```

- **Events:** Normal when constraints are applied; Warning when `TopologyLevelsAvailable` transitions to `False`.
- **Logging:** Info when constraints are applied; Warning when topology levels become unavailable.

## Migration and Backward Compatibility

- No topology constraints at any level → no TAS (backward compatible).
- TAS is applied when any topology constraint is set. The webhook validates the `topologyProfile` references an existing `ClusterTopology` CR and each `packDomain` exists in it; rejection on failure.
- Existing DGDs are unchanged; no automatic topology constraints on existing resources.
- Rollout: deploy updated operator; admins create one or more `ClusterTopology` CRs per the framework's docs; users set `topologyProfile` and topology constraints on new or recreated DGDs to use TAS.

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| TAS in effect when any topology constraint is set | Providing constraints at any level opts in; backward compatible (no constraints = no TAS). No wrapper struct needed. |
| `topologyProfile` required at spec level, inherited by services | Single point of configuration for which ClusterTopology CR to use. Services should not diverge on the topology definition. |
| `TopologyDomain` is a free-form string, not a fixed enum | Domain names are defined by cluster admins in ClusterTopology CRs and vary across clusters. A fixed enum would not accommodate custom or provider-specific topology levels. Regex validation (`^[a-z0-9]+[a-z0-9-]*$`) catches format errors; CRD validation catches non-existent domains. |
| Independent optional `topologyConstraint` at spec and service levels | Mirrors the framework's flexibility; users can set broad deployment-level constraint, targeted per-service constraints, or both. Services without a constraint inherit from spec-level. |
| Same struct at spec and service levels with contextual validation | Avoids two separate types; webhook enforces `topologyProfile` required at spec / forbidden at service, `packDomain` optional at spec / required at service. |
| CRD-based hierarchy validation instead of built-in ordering | Since domains are free-form, the ordering of levels comes from the ClusterTopology CR's `spec.levels` array. This is the single source of truth for "which domain is narrower." |
| Multinode cliques inherit from PCSG | Constraint set on PCSG; child cliques have no explicit constraint, avoiding over-constraining many workers to one host. |
| Struct-based `TopologyConstraint` | Aligns with Grove; room for `FallbackDomain` later. |
| Webhook reads topology CRD for validation; rejects on failure | Gives users immediate feedback at DGD creation; controller does not read CRD. Rejects DGD with error if topology CRD unreadable or validation fails. |
| Dynamo does not define or create topology CRs | Admins set up topology per framework docs; Dynamo only documents the requirement. |
| No smart defaults | Users explicitly set constraints at the level(s) they need; unset levels are left unconstrained (or inherit from parent). |
| Hard constraints (for now) | Only mode available in Grove today; preferred/soft mode can be adopted when the framework supports it. |
| Topology fields immutable on UPDATE | Mirrors Grove's PCS immutability; avoids update failures. Delete and recreate to change. |

## Future Work

* **Preferred / soft constraints:** Today all constraints are hard — the workload fails to schedule if the constraint cannot be satisfied. When the underlying framework (e.g., Grove/KAI) adds support for preferred or soft constraints, Dynamo could extend `TopologyConstraint` with a `mode` field (`required` | `preferred`).

* **Smart defaults enablement:** Smart defaults (e.g., auto-inferring a service constraint when only spec-level is set) were considered but deferred. The current design relies on framework-level inheritance for unset levels. Once frameworks support auto-packing or equivalent, Dynamo could layer on additional defaults.

# Alternate Solutions

## Alt 1 Top-Level serviceOverrides Map Instead of Inline

**Pros:**

* All topology configuration in one place (`topologyConstraints` block).
* Easy to see at a glance all topology settings for the whole deployment.

**Cons:**

* Requires service name as map key — prone to typos, needs name-key validation.
* Disconnects constraint from the service it applies to.
* Less consistent with Grove's pattern (each level carries its own `topologyConstraint`).

**Reason Rejected:**

* Inline on services eliminates service-name validation, mirrors Grove's pattern, and is more consistent with Kubernetes API conventions (fields on the resource they configure).

## Alt 2 Dynamo Reads ClusterTopology and Skips TAS When Missing

**Pros:**

* Graceful degradation: DGD always deploys; TAS is best-effort.
* Dynamo-level status explains why TAS was skipped (e.g., "ClusterTopology not found").

**Cons:**

* Embeds Grove-specific logic (ClusterTopology read) into Dynamo.
* Requires Dynamo RBAC for ClusterTopology.
* Two components (Dynamo + Grove) both need a notion of "TAS available."

**Reason Rejected:**

* Violates framework agnosticism: Dynamo should not adapt behavior based on Grove-specific resources. The framework that owns the PCS is the right place for topology validation.

## Alt 2 Dynamo Defines Its Own Topology CRs and Translates to Grove

**Pros:**

* Dynamo has full control over topology definition.
* Could support multiple backends with a single topology model.

**Cons:**

* Duplicates topology management already handled by Grove.
* Two sources of truth for topology configuration.
* Significantly more complexity and maintenance burden.

**Reason Rejected:**

* Unnecessary duplication; Grove already owns topology lifecycle. Admins should configure topology once per the framework's documentation.

## Alt 3 Automatic TAS on All DGDs (No Opt-In)

**Pros:**

* Simpler UX: users get TAS without any configuration.

**Cons:**

* Breaks existing DGDs on operator upgrade (Grove immutability: cannot add constraints to existing PCS).
* No way for users to opt out if TAS causes scheduling issues.

**Reason Rejected:**

* Backward compatibility is a hard requirement. Grove's immutability rules make automatic TAS application unsafe for existing deployments.
