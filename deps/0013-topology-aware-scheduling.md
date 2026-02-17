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

Add topology-aware scheduling (TAS) support to DynamoGraphDeployment (DGD) so that users can express topology placement preferences (e.g., pack pods within a block or rack) at the deployment and service level. The Dynamo operator injects these user-specified constraints into the generated PodCliqueSet (PCS) spec. The underlying framework (e.g., Grove) owns topology validation and enforcement; Dynamo remains framework-agnostic.

# Motivation

AI inference workloads such as disaggregated serving benefit significantly from locality-aware placement. Packing related pods within the same network block or rack reduces inter-node latency and improves throughput. Today, the Dynamo operator generates Grove PodCliqueSet resources from DGDs with no topology constraints, leaving users unable to express placement preferences through the DGD API.

Grove already provides Topology Aware Scheduling via its ClusterTopology CRD and TopologyConstraint fields on PCS, PCSG, and PodClique resources (GREP-244). The missing piece is a user-facing API on DGD that abstracts away Grove internals and translates deployment/service-level topology preferences into the correct PCS fields.

## Goals

* Allow DGD users to opt in to topology-aware scheduling via a simple, framework-agnostic API.

* Topology constraints are independently optional at the deployment level and at each service level. TAS is in effect when any topology constraint is set at any level. Services without an explicit constraint inherit from the deployment-level constraint (if set).

* Maintain full backward compatibility with existing DGDs (no TAS unless topology constraints are provided).

* Validate topology at DGD creation time (webhook) so users get immediate feedback; when any topology constraint is set, the webhook must validate against the framework's topology CRD and reject the DGD with an error if validation fails or the CRD cannot be read.

### Non Goals

* Dynamo will not define, create, or manage topology CRs (e.g., Grove ClusterTopology). Admins configure topology in the underlying framework per its documentation.

* Dynamic updates to topology constraints after PCS creation are out of scope (blocked by Grove's immutability).

* Backend- or framework-specific default strategies (e.g., different defaults for vLLM vs TRT-LLM) are out of scope for the initial implementation.

* Generalizing the domain vocabulary for non-Grove backends is out of scope. The abstract domain names (region, zone, datacenter, block, rack, host, numa) align with Grove today. For backends that use node labels (e.g., Kueue Topology CRD), a `domainToLabelMapping` in operator config (e.g., `rack` → `cloud.provider.com/topology-rack`) can be introduced so the webhook can validate and translate; only needed when that backend is supported.

* Multi-topology support (selecting a named topology definition per deployment) is out of scope for the initial implementation; it is still in design at the Grove/KAI scheduler level. See Future Work.

## Requirements

### REQ 1 Opt-In by Providing Constraints

TAS **MUST** be opt-in: it is in effect when any topology constraint is set at any level (deployment-level, service-level, or both). If no topology constraints are set anywhere, no TAS is applied. Existing DGDs without topology constraints **MUST** continue to work unchanged after an operator upgrade.

### REQ 2 API Abstraction

The DGD API **MUST NOT** expose Grove-specific terms (PCS, PCSG, PodClique). Users **SHOULD** specify constraints in terms of deployment-level and per-service only.

### REQ 3 Independent Optional Constraints

Topology constraints are independently optional at the deployment level and at each service level. Users **MAY** set:
- Only `spec.topologyConstraint` (broad deployment-level constraint; services inherit).
- Only per-service `topologyConstraint` (targeted constraint for specific services).
- Both (deployment-level default with per-service overrides).

**Note on constraint type:** Today, all topology constraints in the underlying framework (Grove) are hard/required constraints — there is no "preferred" or soft mode. If the scheduler cannot satisfy the constraint, the workload will not be scheduled. This is acceptable because TAS is explicitly opt-in and users specify constraints knowingly. When the underlying framework adds support for preferred/soft constraints, the API could be extended accordingly.

### REQ 5 Hierarchy Validation

When both spec-level and service-level constraints are set, the service-level constraint **MUST** be equal to or narrower than the deployment-level constraint. The operator **MUST** reject a DGD where a service constraint is broader than the deployment constraint.

### REQ 6 Framework Agnosticism

The high-level DGD API (topology constraints, `packDomain`) **MUST** be the same for all supported frameworks. Users express topology in Dynamo's abstract vocabulary (e.g., `region`, `zone`, `block`, `rack`, `host`, `numa`), not in framework-specific terms. Future internal work may be required to translate this API to certain frameworks (e.g., mapping abstract domains to framework-specific node labels); the DGD API itself does not change.

**Why abstract domain names instead of node labels:** Node labels are cluster-specific and cloud-provider-specific (e.g., `cloud.provider.com/topology-rack` vs `topology.kubernetes.io/rack`), so DGDs written with labels would not be portable across clusters. Labels also do not work with Grove, whose `PackDomain` takes a domain name (e.g., `rack`), not a label; using labels would require a fragile reverse-lookup from label to domain name. Abstract domains let users express intent in natural terms ("pack in the same rack") without knowing infrastructure-level label strings. The operator handles framework-specific translation internally: for Grove the domains map 1:1; for future backends like Kueue a `domainToLabelMapping` config translates without changing the DGD API.

### REQ 7 Backward Compatibility

No automatic TAS application on existing DGDs. Users opt in by setting topology constraints at the level(s) they need; to add TAS to an existing DGD they must recreate it with constraints set.

# Proposal

## API Design

Add an optional `topologyConstraint` field to `DynamoGraphDeploymentSpec` (deployment-level) and to each `ServiceSpec` (service-level). Both use the same `TopologyConstraint` struct and are independently optional:

```go
type DynamoGraphDeploymentSpec struct {
    // ... existing fields ...

    // TopologyConstraint is the deployment-level topology constraint.
    // When set, applies a broad topology constraint across the whole deployment.
    // Services without their own topologyConstraint inherit this value.
    // +optional
    TopologyConstraint *TopologyConstraint `json:"topologyConstraint,omitempty"`
}

type ServiceSpec struct {
    // ... existing fields (name, replicas, multinode, etc.) ...

    // TopologyConstraint for this service. When both this and spec.topologyConstraint
    // are set, this must be narrower than or equal to the spec-level constraint.
    // +optional
    TopologyConstraint *TopologyConstraint `json:"topologyConstraint,omitempty"`
}

type TopologyConstraint struct {
    PackDomain TopologyDomain `json:"packDomain"`
}

type TopologyDomain string  // region|zone|datacenter|block|rack|host|numa
```

**User examples:**

Broad deployment-level constraint only (services inherit):

```yaml
spec:
  topologyConstraint:
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

Service-level only (no deployment-level constraint):

```yaml
spec:
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
| spec.topologyConstraint | PCS.Spec.Template.TopologyConstraint | when set | User-specified |
| service.topologyConstraint | PCSG.TopologyConstraint | Multinode svc, when set | User-specified |
| service.topologyConstraint | PodClique.TopologyConstraint | Single-node svc, when set | User-specified |
| — | PodClique (multinode) | Always | No explicit constraint (inherits from PCSG or PCS) |
| — | PodClique/PCSG | Service has no constraint | No explicit constraint (inherits from PCS template if set) |

## Constraint Resolution

Topology constraints are independently optional at every level. The controller injects whichever constraints are specified; it does not read the topology CRD. The webhook (see Validation) reads the topology CRD for admission-time validation.

**Resolution rules:**

- **spec.topologyConstraint set, service has no constraint:** Service inherits from the PCS template (no explicit constraint injected on the child resource; the framework handles inheritance).
- **spec.topologyConstraint set, service also has constraint:** Both are injected; hierarchy is validated (service must be ≤ spec-level).
- **spec.topologyConstraint nil, service has constraint:** Only the service's constraint is injected on the child resource (PCSG or PodClique). No PCS-level constraint.
- **Neither set:** No topology constraints anywhere (backward compatible).

**Operator configuration (optional, for future use):**

```yaml
operator:
  topologyAwareScheduling:
    # Future (non-Grove backends, e.g. Kueue): map abstract domains to node labels for validation.
    # domainToLabelMapping:
    #   block: "cloud.provider.com/topology-block"
    #   rack: "cloud.provider.com/topology-rack"
    #   host: "kubernetes.io/hostname"
```

## PCS Generation

In `GenerateGrovePodCliqueSet` (or equivalent), inject whichever constraints are specified:

- **PCS template:** Set from `spec.topologyConstraint` (nil if not set).
- **For each service:**
  - If service has `topologyConstraint`:
    - Multinode: set PCSG topology constraint; no explicit constraint on child cliques (they inherit from PCSG).
    - Single-node: set PodClique topology constraint.
  - If service has no `topologyConstraint`: no explicit constraint on the child resource (it inherits from PCS template if set, or has no constraint).

If the underlying framework rejects the generated resources (e.g., topology definition missing or invalid domain), the error surfaces at the framework level.

## Validation (Webhook)

### On CREATE

- **Struct:** Every `packDomain` that is set **MUST** be one of the known `TopologyDomain` values: `region`, `zone`, `datacenter`, `block`, `rack`, `host`, `numa`.
- **Hierarchy:** When both `spec.topologyConstraint` and a service's `topologyConstraint` are set, the service's `packDomain` **MUST** be narrower than or equal to the spec-level `packDomain`.
- **Topology CRD validation:** When any topology constraint is set (at any level), the webhook **MUST** read the framework's topology CRD (e.g., Grove ClusterTopology) to validate that each `packDomain` used is a valid level in the cluster's topology. If the CRD cannot be read (e.g., RBAC missing, CRD not installed) or validation fails, the webhook **MUST** reject the DGD with an error.
- **RBAC:** The Dynamo operator needs read-only access to the framework's topology CRD (e.g., `clustertopologies.grove.io`) for webhook validation when that backend is in use.

### On UPDATE

All topology-related fields are **immutable** once the DGD is created, mirroring the framework's immutability of topology constraints on generated resources. The following changes **MUST** be rejected:

- **`spec.topologyConstraint`:** Cannot be added, removed, or have `packDomain` changed.
- **Service `topologyConstraint`:** Cannot be added, removed, or have `packDomain` changed on any existing service.

To change topology constraints, users must delete and recreate the DGD. This aligns with the framework's model and avoids update failures on generated resources.

## Framework Agnosticism and Topology Ownership

**Abstract vocabulary:** `PackDomain` uses Dynamo's own abstract topology vocabulary (`region`, `zone`, `datacenter`, `block`, `rack`, `host`, `numa`) — defined as a `TopologyDomain` type in the DGD API. These are Dynamo's own domain names, not "Grove's domains leaked into Dynamo"; they align with Grove today because both use natural, user-friendly topology terms. The alternative — using node labels directly — would make DGDs non-portable across clusters (labels are cloud/cluster-specific, e.g., `cloud.provider.com/topology-rack` vs `topology.kubernetes.io/rack`) and would not work with Grove (whose `PackDomain` takes domain names, not labels). Dynamo's webhook validates that `packDomain` is one of these known names (catches typos at admission). The framework validates that the domain is a valid level in the cluster's topology. For future non-Grove backends (e.g., Kueue), a translation layer (e.g., operator config mapping abstract domains to framework-specific node labels) can translate without changing the DGD API.

**Webhook vs controller:** The webhook reads the framework's topology CRD when any topology constraint is set (at any level) and rejects the DGD with an error if the CRD cannot be read or validation fails. The controller does not read the topology CRD; it injects whichever constraints are specified.

Dynamo does not define or create topology CRs. When Grove is used, the cluster's topology (ClusterTopology) is configured by the admin per Grove's documentation. Dynamo requires read-only RBAC for the framework's topology CRD when that backend is in use (for webhook validation).

## Status and Observability

**Topology condition on DGD status:**

When any topology constraint is set (spec-level, service-level, or both), the controller surfaces a `TopologyConstraintsEnforced` condition on the DGD status. The controller reads topology-related conditions from the underlying workload resource (framework-specific) and translates them into this Dynamo-level condition:

| Condition Type | Status | Reason | When |
|----------------|--------|--------|------|
| `TopologyConstraintsEnforced` | `True` | `AllTopologyLevelsAvailable` | TAS active and all required topology levels are available in the cluster topology |
| `TopologyConstraintsEnforced` | `False` | `TopologyLevelsUnavailable` | The framework reports that one or more required topology levels are no longer available |
| `TopologyConstraintsEnforced` | `False` | `TopologyDefinitionNotFound` | The framework reports that the topology definition resource was not found |
| *(absent)* | — | — | No topology constraints set at any level |

This condition is necessary because the underlying framework may not fail the deployment when a topology level becomes unavailable — it may keep pods running but silently stop enforcing the constraint. Without propagating this, the DGD would appear `Ready` with no indication that topology enforcement stopped.

**Framework translation (Grove):** The controller reads the PCS `Conditions` and maps Grove's `TopologyLevelsUnavailable` condition (with reasons `ClusterTopologyLevelsUnavailable` and `ClusterTopologyNotFound`) to the `TopologyConstraintsEnforced` condition above. Other frameworks would implement equivalent mappings from their topology health signals.

Example DGD status when a topology level becomes unavailable:

```yaml
status:
  conditions:
    - type: Ready
      status: "True"
    - type: TopologyConstraintsEnforced
      status: "False"
      reason: TopologyLevelsUnavailable
      message: "Topology level 'rack' is no longer available in the cluster topology"
```

- **Events:** Normal when constraints are applied; Warning when `TopologyConstraintsEnforced` transitions to `False`.
- **Logging:** Info when constraints are applied; Warning when topology levels become unavailable.

## Migration and Backward Compatibility

- No topology constraints at any level → no TAS (backward compatible).
- TAS is applied when any topology constraint is set. The webhook validates topology in that case and rejects the DGD with an error if the topology CRD cannot be read or validation fails.
- Existing DGDs are unchanged; no automatic topology constraints on existing resources.
- Rollout: deploy updated operator; when using Grove, admins configure ClusterTopology per Grove docs; users set topology constraints at the level(s) they need on new or recreated DGDs to use TAS.

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| TAS in effect when any topology constraint is set | Providing constraints at any level opts in; backward compatible (no constraints = no TAS). No wrapper struct needed. |
| Independent optional `topologyConstraint` at spec and service levels | Mirrors the framework's flexibility; users can set broad deployment-level constraint, targeted per-service constraints, or both. Services without a constraint inherit from spec-level. |
| Multinode cliques inherit from PCSG | Constraint set on PCSG; child cliques have no explicit constraint, avoiding over-constraining many workers to one host. |
| Struct-based `TopologyConstraint` | Aligns with Grove; room for `FallbackDomain` later. |
| Webhook reads topology CRD for validation; rejects on failure | Gives users immediate feedback at DGD creation; controller does not read CRD. Rejects DGD with error if topology CRD unreadable or validation fails. |
| Dynamo does not define or create topology CRs | Admins set up topology per framework docs; Dynamo only documents the requirement. |
| No smart defaults | Users explicitly set constraints at the level(s) they need; unset levels are left unconstrained (or inherit from parent). |
| `TopologyDomain` is a fixed enum, not configurable | Domain names (region, zone, etc.) are Dynamo's own abstract vocabulary; the framework validates that a domain exists in the referenced topology. For non-Grove backends, a translation layer can map abstract domains to framework-specific values. |
| Hard constraints (for now) | Only mode available in Grove today; preferred/soft mode can be adopted when the framework supports it. |
| Topology fields immutable on UPDATE | Mirrors Grove's PCS immutability; avoids update failures. Delete and recreate to change. |

## Future Work

* **Multi-topology support:** Multiple named topology definitions per cluster (e.g., one for H100, one for GB200 node pools) is still in design at the Grove/KAI scheduler level. Once that is available, Dynamo could add an optional field (e.g., `topologyName`) to the DGD API so users can select which topology applies to their deployment; the operator would pass it through to the generated PCS (e.g., `clusterTopologyName` in Grove).

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
