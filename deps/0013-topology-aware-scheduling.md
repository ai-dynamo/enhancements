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

Add topology-aware scheduling (TAS) support to DynamoGraphDeployment (DGD) so that users can express topology placement preferences (e.g., pack pods within a block or rack) at the deployment and service level, and optionally select which named topology definition to use for heterogeneous clusters. The Dynamo operator resolves these preferences—using smart defaults and user overrides—and injects them into the generated PodCliqueSet (PCS) spec. The underlying framework (e.g., Grove) owns topology validation and enforcement; Dynamo remains framework-agnostic.

# Motivation

AI inference workloads such as disaggregated serving benefit significantly from locality-aware placement. Packing related pods within the same network block or rack reduces inter-node latency and improves throughput. Today, the Dynamo operator generates Grove PodCliqueSet resources from DGDs with no topology constraints, leaving users unable to express placement preferences through the DGD API.

Grove already provides Topology Aware Scheduling via its ClusterTopology CRD and TopologyConstraint fields on PCS, PCSG, and PodClique resources (GREP-244). Grove is also adding multi-topology support (GREP-369), allowing multiple named ClusterTopology resources per cluster for heterogeneous hardware (e.g., H100 vs GB200 node pools with different network hierarchies). The missing piece is a user-facing API on DGD that abstracts away Grove internals and translates deployment/service-level topology preferences—including topology selection—into the correct PCS fields.

## Goals

* Allow DGD users to opt in to topology-aware scheduling via a simple, framework-agnostic API.

* Provide smart defaults (PCS=block, PCSG=rack) so users do not have to configure every level.

* Support per-service overrides for fine-grained control.

* Support multi-topology clusters by allowing users to specify which named topology definition applies to their deployment (e.g., for heterogeneous hardware or multi-cloud environments).

* Maintain full backward compatibility with existing DGDs (no TAS unless explicitly enabled).

* Keep Dynamo free of framework-specific logic (no ClusterTopology reads, no Grove-specific validation).

### Non Goals

* Dynamo will not define, create, or manage topology CRs (e.g., Grove ClusterTopology). Admins configure topology in the underlying framework per its documentation.

* Dynamo will not validate that topology domains exist in the cluster; the framework handles that at admission or reconciliation.

* Dynamic updates to topology constraints after PCS creation are out of scope (blocked by Grove's immutability).

* Backend- or framework-specific default strategies (e.g., different defaults for vLLM vs TRT-LLM) are out of scope for the initial implementation.

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

Both the smart defaults and the list of allowed topology domain names **MUST** be configurable at the Dynamo operator level (e.g., via Helm values or operator configuration), so that cluster administrators can tune them per cluster without requiring users to modify their DGDs. For Grove, the Helm defaults ship with `[region, zone, datacenter, block, rack, host, numa]`; for other frameworks, the admin overrides the list to match the supported domains.

**Note on constraint type:** Today, all topology constraints in the underlying framework (Grove) are hard/required constraints — there is no "preferred" or soft mode. This means that if the scheduler cannot satisfy the constraint, the workload will not be scheduled. This is acceptable because:
1. TAS is explicitly opt-in; users choose to enable it knowing constraints will be enforced.
2. Users can override defaults per service or disable the feature entirely if constraints cause scheduling issues.
3. When the underlying framework adds support for preferred/soft constraints, the smart defaults could be changed to preferred mode, reducing scheduling failure risk while still providing best-effort topology placement.

### REQ 4 User Overrides

Users **MUST** be able to override the deployment-level constraint and per-service constraints. When an override is provided, it **MUST** be used instead of the smart default.

### REQ 5 Hierarchy Validation

Service-level constraints **MUST** be equal to or narrower than the deployment-level constraint. The operator **MUST** reject a DGD where a service constraint is broader than the deployment constraint.

### REQ 6 Framework Agnosticism

Dynamo **MUST NOT** read or depend on framework-specific topology resources (e.g., Grove ClusterTopology). It **MUST** only inject resolved constraints into the generated spec. The underlying framework is responsible for validating domain existence and topology availability.

### REQ 7 Backward Compatibility

No automatic TAS application on existing DGDs. Users **MUST** explicitly enable TAS before first deployment or recreate their DGD with `enabled: true`.

### REQ 8 Multi-Topology Support

Users **SHOULD** be able to specify an optional topology name to select which topology definition applies to their deployment (e.g., for clusters with heterogeneous hardware). The topology name **MUST** be at the deployment level (not per-service), since topology selection applies to the entire generated PCS. If omitted, the framework's default topology **MUST** be used. Dynamo **MUST NOT** validate that the topology name references an existing resource; the framework does that.

# Proposal

## API Design

Add an optional `TopologyConstraints` section to `DynamoGraphDeploymentSpec`:

```go
type TopologyConstraints struct {
    // Enabled controls whether TAS is applied. Default: false.
    Enabled bool `json:"enabled"`

    // TopologyName references a specific topology definition (e.g., a named
    // ClusterTopology in Grove). If omitted, the framework's default topology is used.
    // Useful for heterogeneous clusters with multiple topology hierarchies.
    // +optional
    TopologyName string `json:"topologyName,omitempty"`

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

**User example (with named topology and inline service constraints):**

```yaml
spec:
  topologyConstraints:
    enabled: true
    topologyName: aws-p5   # optional; omit for framework default
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
| topologyName | PCS.Spec.Template.ClusterTopologyName | enabled=true and topologyName set | User-specified name |
| topologyConstraints.deployment | PCS.Spec.Template.TopologyConstraint | enabled=true | Override or default (block) |
| service.topologyConstraint | PCSG.TopologyConstraint | Multinode svc | Override or default (rack) |
| service.topologyConstraint | PodClique.TopologyConstraint | Single-node svc | Override or default (rack) |
| — | PodClique (multinode) | Multinode svc | No explicit constraint (inherit PCSG) |

## Smart Defaults and Override Resolution

Dynamo does not read ClusterTopology or any framework-specific topology resource. It computes resolved constraints from user overrides and smart defaults only.

| Level | Default | Rationale |
|-------|--------|-----------|
| PCS (deployment) | block | Good capacity/locality balance for the whole deployment |
| PCSG (multinode service) | rack | Packs service replicas; cliques inherit so workers can span hosts within a rack |
| Single-node clique | rack | Locality without over-constraining |

Both the smart defaults and the allowed domain list are configurable at the Dynamo operator level (e.g., Helm values), allowing admins to adjust them per cluster.

**Operator configuration example (Helm values):**

```yaml
operator:
  topologyAwareScheduling:
    defaults:
      deployment: block
      service: rack
    allowedDomains:
      - region
      - zone
      - datacenter
      - block
      - rack
      - host
      - numa
```

For each level: if user set a value (inline `topologyConstraint` on the service), use it; otherwise use the smart default. Validate `packDomain` against the configured `allowedDomains` list and validate hierarchy (service ≤ deployment). Dynamo does not validate domain existence in the cluster; the framework does.

## PCS Generation

In `GenerateGrovePodCliqueSet` (or equivalent):

1. If `topologyConstraints` is nil or `enabled=false` → generate PCS without topology constraints. Any `topologyConstraint` on services is ignored.
2. If `enabled=true` → for each service, resolve its topology constraint (inline override or smart default), validate hierarchy, then inject:
   - If `topologyName` is set: set `PCS.Spec.Template.ClusterTopologyName`.
   - PCS template: deployment constraint (or default block).
   - For each multinode service: set PCSG constraint (service's `topologyConstraint` or default rack); no explicit constraint on child cliques.
   - For each single-node service: set PodClique constraint (service's `topologyConstraint` or default rack).

If the underlying framework rejects the PCS (e.g., ClusterTopology missing or invalid domain), the error surfaces at the framework level.

## Validation (Webhook)

### On CREATE

- **Enabled flag:** If `enabled=false` and (`topologyName` set, or `deployment` set, or any service has `topologyConstraint`) → reject: "cannot specify topology constraints when TAS is disabled".
- **Struct:** `PackDomain` **MUST** be set and be one of the allowed domain names configured at the operator level (defaults to Grove's domains: region, zone, datacenter, block, rack, host, numa).
- **Hierarchy:** Each service's `topologyConstraint` **MUST** be ≤ deployment (narrower or equal).
- **Topology name:** Dynamo does not validate that `topologyName` references an existing topology; the framework does that at PCS admission.

### On UPDATE

All topology-related fields are **immutable** once the DGD is created, mirroring Grove's immutability of topology constraints on PCS. The following changes **MUST** be rejected:

- **`enabled`:** Cannot change from `true` to `false` (would require removing constraints from PCS → Grove rejects) or from `false` to `true` (would require adding constraints to existing PCS → Grove rejects).
- **`topologyName`:** Cannot be added, removed, or changed.
- **`deployment`:** Cannot be added, removed, or have `packDomain` changed.
- **Service `topologyConstraint`:** Cannot be added, removed, or have `packDomain` changed on any existing service.

To change topology constraints, users must delete and recreate the DGD (which creates a new PCS). This aligns with Grove's model and avoids PCS update failures.

### General

Dynamo does not validate domain existence, topology name validity, or ClusterTopology presence. The underlying framework does that when the PCS is admitted or reconciled.

## Framework Agnosticism and Topology Ownership

Dynamo does not read or check ClusterTopology (or any framework-specific topology resource). It injects resolved constraints and the optional topology name into the generated PCS when `enabled=true`. The framework that owns the generated resource (e.g., Grove) is responsible for validation and error reporting if topology is missing or invalid.

**Multi-topology support:** Clusters may have multiple topology definitions for heterogeneous hardware (e.g., H100 vs GB200 node pools with different network hierarchies; see GREP-369). The optional `topologyName` field allows users (or external systems like Run:ai) to specify which topology applies to a DGD. Dynamo passes this through to the generated PCS (`clusterTopologyName`); the framework resolves it. If omitted, the framework uses its default topology.

Dynamo does not define or create topology CRs. When Grove is used, ClusterTopology resources (default and additional named topologies) are configured by the admin per Grove's documentation. Dynamo documents this prerequisite. No Dynamo RBAC for ClusterTopology is required.

## Status and Observability

- **Status:** Add `AppliedTopologyConstraints` to DGD status reporting the applied constraints when `enabled=true`.
- **Events:** Normal when constraints are applied; Warning/Error on validation failures.
- **Logging:** Info when constraints are applied; Error when validation fails.

## Migration and Backward Compatibility

- No `topologyConstraints` block or `enabled=false` → no TAS (backward compatible).
- TAS is applied only when `enabled=true`. The framework may fail if topology is not configured; Dynamo does not check.
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
| Dynamo does not read ClusterTopology | Keeps Dynamo framework-agnostic; framework validates and surfaces errors. |
| Dynamo does not define or create topology CRs | Admins set up topology per framework docs; Dynamo only documents the requirement. |
| `topologyName` at top level, not inside `TopologyConstraint` | Topology selection is per-deployment (maps to PCS-level `clusterTopologyName`), not per-service. |
| Generic `topologyName` naming (not `clusterTopologyName`) | Framework-agnostic; avoids exposing Grove-specific resource names in DGD API. |
| Smart defaults and allowed domains configurable at operator level | Admins can tune defaults and domain vocabulary per cluster/framework without user DGD changes or code changes. |
| Hard constraints (for now) | Only mode available in Grove today; preferred/soft mode can be adopted when the framework supports it. |
| Topology fields immutable on UPDATE | Mirrors Grove's PCS immutability; avoids update failures. Delete and recreate to change. |

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
