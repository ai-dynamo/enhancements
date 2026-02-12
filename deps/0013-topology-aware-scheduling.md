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

Add topology-aware scheduling (TAS) support to DynamoGraphDeployment (DGD) so that users can express topology placement preferences (e.g., pack pods within a block or rack) at the deployment and service level. The Dynamo operator resolves these preferences—using smart defaults and user overrides—and injects them into the generated PodCliqueSet (PCS) spec. The underlying framework (e.g., Grove) owns topology validation and enforcement; Dynamo remains framework-agnostic.

# Motivation

AI inference workloads such as disaggregated serving benefit significantly from locality-aware placement. Packing related pods within the same network block or rack reduces inter-node latency and improves throughput. Today, the Dynamo operator generates Grove PodCliqueSet resources from DGDs with no topology constraints, leaving users unable to express placement preferences through the DGD API.

Grove already provides Topology Aware Scheduling via its ClusterTopology CRD and TopologyConstraint fields on PCS, PCSG, and PodClique resources (GREP-244). The missing piece is a user-facing API on DGD that abstracts away Grove internals and translates deployment/service-level topology preferences into the correct PCS fields.

## Goals

* Allow DGD users to opt in to topology-aware scheduling via a simple, framework-agnostic API.

* Provide smart defaults (PCS=block, PCSG=rack) so users do not have to configure every level.

* Support per-service overrides for fine-grained control.

* Maintain full backward compatibility with existing DGDs (no TAS unless explicitly enabled).

* Validate topology at DGD creation time (webhook) so users get immediate feedback; when TAS is enabled, the webhook must validate against the framework's topology CRD and reject the DGD with an error if validation fails or the CRD cannot be read.

### Non Goals

* Dynamo will not define, create, or manage topology CRs (e.g., Grove ClusterTopology). Admins configure topology in the underlying framework per its documentation.

* Dynamic updates to topology constraints after PCS creation are out of scope (blocked by Grove's immutability).

* Backend- or framework-specific default strategies (e.g., different defaults for vLLM vs TRT-LLM) are out of scope for the initial implementation.

* Generalizing the domain vocabulary for non-Grove backends is out of scope. The abstract domain names (region, zone, datacenter, block, rack, host, numa) align with Grove today. For backends that use node labels (e.g., Kueue Topology CRD), a `domainToLabelMapping` in operator config (e.g., `rack` → `cloud.provider.com/topology-rack`) can be introduced so the webhook can validate and translate; only needed when that backend is supported.

* Multi-topology support (selecting a named topology definition per deployment) is out of scope for the initial implementation; it is still in design at the Grove/KAI scheduler level. See Future Work.

## Requirements

### REQ 1 Explicit Opt-In

TAS **MUST** be opt-in via a boolean `enabled` field (default `false`). Existing DGDs without topology constraints **MUST** continue to work unchanged after an operator upgrade. This avoids violating Grove's immutability rules on existing PCS resources.

### REQ 2 API Abstraction

The DGD API **MUST NOT** expose Grove-specific terms (PCS, PCSG, PodClique). Users **SHOULD** specify constraints in terms of "deployment" and "services" only.

### REQ 3 Smart Defaults

When `enabled: true` and no explicit constraints are provided, the operator **MUST** apply sensible defaults:
- Deployment level (PCS): `packDomain: block`
- Multinode services (PCSG): `packDomain: rack`; child cliques inherit from PCSG (no explicit constraint)
- Single-node services (PodClique): `packDomain: rack`

The smart defaults **MUST** be configurable at the Dynamo operator level (e.g., via Helm values or operator configuration), so that cluster administrators can tune them per cluster without requiring users to modify their DGDs.

**Note on constraint type:** Today, all topology constraints in the underlying framework (Grove) are hard/required constraints — there is no "preferred" or soft mode. This means that if the scheduler cannot satisfy the constraint, the workload will not be scheduled. This is acceptable because:
1. TAS is explicitly opt-in; users choose to enable it knowing constraints will be enforced.
2. Users can override defaults per service or disable the feature entirely if constraints cause scheduling issues.
3. When the underlying framework adds support for preferred/soft constraints, the smart defaults could be changed to preferred mode, reducing scheduling failure risk while still providing best-effort topology placement.

### REQ 4 User Overrides

Users **MUST** be able to override the deployment-level constraint and per-service constraints. When an override is provided, it **MUST** be used instead of the smart default.

### REQ 5 Hierarchy Validation

Service-level constraints **MUST** be equal to or narrower than the deployment-level constraint. The operator **MUST** reject a DGD where a service constraint is broader than the deployment constraint.

### REQ 6 Framework Agnosticism

The high-level DGD API (topology constraints, `packDomain`) **MUST** be the same for all supported frameworks. Users express topology in Dynamo's abstract vocabulary (e.g., `region`, `zone`, `block`, `rack`, `host`, `numa`), not in framework-specific terms. Future internal work may be required to translate this API to certain frameworks (e.g., mapping abstract domains to framework-specific node labels); the DGD API itself does not change.

### REQ 7 Backward Compatibility

No automatic TAS application on existing DGDs. Users **MUST** explicitly enable TAS before first deployment or recreate their DGD with `enabled: true`.

# Proposal

## API Design

Add an optional `TopologyConstraints` section to `DynamoGraphDeploymentSpec`:

```go
type TopologyConstraints struct {
    // Enabled controls whether TAS is applied. Default: false.
    Enabled bool `json:"enabled"`

    // Deployment-level constraint. If nil when enabled=true: smart default (block).
    Deployment *TopologyConstraint `json:"deployment,omitempty"`
}

type TopologyConstraint struct {
    PackDomain TopologyDomain `json:"packDomain"`
}

type TopologyDomain string  // region|zone|datacenter|block|rack|host|numa
```

Per-service topology constraints are specified inline on each service definition:

```go
type ServiceSpec struct {
    // ... existing fields (name, replicas, multinode, etc.) ...

    // TopologyConstraint overrides the smart default for this service.
    // If nil when TAS is enabled: smart default (rack).
    // +optional
    TopologyConstraint *TopologyConstraint `json:"topologyConstraint,omitempty"`
}
```

**User example (inline service constraints):**

```yaml
spec:
  topologyConstraints:
    enabled: true
    deployment:
      packDomain: zone
  services:
    - name: VllmWorker
      multinode: { nodeCount: 4 }
      replicas: 2
      topologyConstraint:
        packDomain: block
    - name: Frontend
      replicas: 1
      topologyConstraint:
        packDomain: zone
```

**Mapping from DGD to generated spec (Grove PCS):**

| DGD | Grove | When | Value |
|-----|-------|------|--------|
| topologyConstraints.deployment | PCS.Spec.Template.TopologyConstraint | enabled=true | Override or default (block) |
| service.topologyConstraint | PCSG.TopologyConstraint | Multinode svc | Override or default (rack) |
| service.topologyConstraint | PodClique.TopologyConstraint | Single-node svc | Override or default (rack) |
| — | PodClique (multinode) | Multinode svc | No explicit constraint (inherit PCSG) |

## Smart Defaults and Override Resolution

The controller computes resolved constraints from user overrides and smart defaults only; it does not read the topology CRD. The webhook (see Validation) may read the topology CRD for admission-time validation.

| Level | Default | Rationale |
|-------|--------|-----------|
| PCS (deployment) | block | Good capacity/locality balance for the whole deployment |
| PCSG (multinode service) | rack | Packs service replicas; cliques inherit so workers can span hosts within a rack |
| Single-node clique | rack | Locality without over-constraining |

The smart defaults are configurable at the Dynamo operator level (e.g., Helm values), allowing admins to adjust them per cluster.

**Operator configuration example (Helm values):**

```yaml
operator:
  topologyAwareScheduling:
    defaults:
      deployment: block
      service: rack
    # Future (non-Grove backends, e.g. Kueue): map abstract domains to node labels for validation.
    # domainToLabelMapping:
    #   block: "cloud.provider.com/topology-block"
    #   rack: "cloud.provider.com/topology-rack"
    #   host: "kubernetes.io/hostname"
```

For each level: if user set a value (inline `topologyConstraint` on the service), use it; otherwise use the smart default. Validate `packDomain` against the known `TopologyDomain` values and validate hierarchy (service ≤ deployment). When the webhook can read the framework's topology CRD, it also validates that each `packDomain` is a valid level in the referenced topology (see Validation); otherwise the framework surfaces errors at reconciliation.

## PCS Generation

In `GenerateGrovePodCliqueSet` (or equivalent):

1. If `topologyConstraints` is nil or `enabled=false` → generate PCS without topology constraints. Any `topologyConstraint` on services is ignored.
2. If `enabled=true` → for each service, resolve its topology constraint (inline override or smart default), validate hierarchy, then inject:
   - PCS template: deployment constraint (or default block).
   - For each multinode service: set PCSG constraint (service's `topologyConstraint` or default rack); no explicit constraint on child cliques.
   - For each single-node service: set PodClique constraint (service's `topologyConstraint` or default rack).

If the underlying framework rejects the PCS (e.g., ClusterTopology missing or invalid domain), the error surfaces at the framework level.

## Validation (Webhook)

### On CREATE

- **Enabled flag:** If `enabled=false` and (`deployment` set, or any service has `topologyConstraint`) → reject: "cannot specify topology constraints when TAS is disabled".
- **Struct:** `PackDomain` **MUST** be set and be one of the known `TopologyDomain` values: `region`, `zone`, `datacenter`, `block`, `rack`, `host`, `numa`.
- **Hierarchy:** Each service's `topologyConstraint` **MUST** be ≤ deployment (narrower or equal).
- **Topology CRD validation:** When TAS is enabled, the webhook **MUST** read the framework's topology CRD (e.g., Grove ClusterTopology) to validate that each `packDomain` (deployment and per-service) is a valid level in the cluster's topology (single/default topology). For Grove, domain names match 1:1 (rack = rack). If the CRD cannot be read (e.g., RBAC missing, CRD not installed) or validation fails, the webhook **MUST** reject the DGD with an error.
- **RBAC:** The Dynamo operator needs read-only access to the framework's topology CRD (e.g., `clustertopologies.grove.io`) for webhook validation when that backend is in use.

### On UPDATE

All topology-related fields are **immutable** once the DGD is created, mirroring Grove's immutability of topology constraints on PCS. The following changes **MUST** be rejected:

- **`enabled`:** Cannot change from `true` to `false` (would require removing constraints from PCS → Grove rejects) or from `false` to `true` (would require adding constraints to existing PCS → Grove rejects).
- **`deployment`:** Cannot be added, removed, or have `packDomain` changed.
- **Service `topologyConstraint`:** Cannot be added, removed, or have `packDomain` changed on any existing service.

To change topology constraints, users must delete and recreate the DGD (which creates a new PCS). This aligns with Grove's model and avoids PCS update failures.

### General

When TAS is enabled, the webhook must validate domain existence against the topology CRD; if the CRD cannot be read or validation fails, the webhook rejects the DGD with an error.

## Framework Agnosticism and Topology Ownership

**Abstract vocabulary:** `PackDomain` uses Dynamo's own abstract topology vocabulary (`region`, `zone`, `datacenter`, `block`, `rack`, `host`, `numa`) — defined as a `TopologyDomain` type in the DGD API. This is not "Grove's domains leaked into Dynamo"; it aligns with Grove today because these are natural, user-friendly terms. Dynamo's webhook validates that `packDomain` is one of these known names (catches typos at admission). The framework validates that the domain is a valid level in the cluster's topology. For future non-Grove backends (e.g., Kueue), a translation layer (e.g., operator config mapping abstract domains to framework-specific node labels) can translate without changing the DGD API.

**Webhook vs controller:** The webhook reads the framework's topology CRD when TAS is enabled and rejects the DGD with an error if the CRD cannot be read or validation fails. The controller does not read the topology CRD; it injects resolved constraints when `enabled=true`.

Dynamo does not define or create topology CRs. When Grove is used, the cluster's topology (ClusterTopology) is configured by the admin per Grove's documentation. Dynamo requires read-only RBAC for the framework's topology CRD when that backend is in use (for webhook validation).

## Status and Observability

- **Status:** Add `AppliedTopologyConstraints` to DGD status reporting the applied constraints when `enabled=true`.
- **Events:** Normal when constraints are applied; Warning/Error on validation failures.
- **Logging:** Info when constraints are applied; Error when validation fails.

## Migration and Backward Compatibility

- No `topologyConstraints` block or `enabled=false` → no TAS (backward compatible).
- TAS is applied only when `enabled=true`. The webhook must validate topology when TAS is enabled and rejects the DGD with an error if the topology CRD cannot be read or validation fails.
- Existing DGDs are unchanged; no automatic topology constraints on existing PCS.
- Rollout: deploy updated operator; when using Grove, admins configure ClusterTopology per Grove docs; users set `enabled=true` on new or recreated DGDs.

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| `enabled: bool`, default false | Explicit opt-in; backward compatible; avoids immutability issues. |
| Inline `topologyConstraint` on services (not `serviceOverrides` map) | Co-locates constraint with service; eliminates name-key typos and service-name validation; mirrors Grove pattern. |
| Multinode cliques inherit from PCSG | Avoids forcing many workers onto one host; respects capacity. |
| Single-node clique default: rack | Good locality without over-constraining. |
| Struct-based `TopologyConstraint` | Aligns with Grove; room for `FallbackDomain` later. |
| Webhook reads topology CRD for validation; rejects on failure | Gives users immediate feedback at DGD creation; controller does not read CRD. Rejects DGD with error if topology CRD unreadable or validation fails. |
| Dynamo does not define or create topology CRs | Admins set up topology per framework docs; Dynamo only documents the requirement. |
| Smart defaults configurable at operator level | Admins can tune default `packDomain` values per cluster without user DGD changes. |
| `TopologyDomain` is a fixed enum, not configurable | Domain names (region, zone, etc.) are Dynamo's own abstract vocabulary; the framework validates that a domain exists in the referenced topology. For non-Grove backends, a translation layer can map abstract domains to framework-specific values. |
| Hard constraints (for now) | Only mode available in Grove today; preferred/soft mode can be adopted when the framework supports it. |
| Topology fields immutable on UPDATE | Mirrors Grove's PCS immutability; avoids update failures. Delete and recreate to change. |

## Future Work

* **Multi-topology support:** Multiple named topology definitions per cluster (e.g., one for H100, one for GB200 node pools) is still in design at the Grove/KAI scheduler level. Once that is available, Dynamo could add an optional field (e.g., `topologyName`) to the DGD API so users can select which topology applies to their deployment; the operator would pass it through to the generated PCS (e.g., `clusterTopologyName` in Grove).

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
