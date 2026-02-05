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

AI inference workloads such as disaggregated vLLM benefit significantly from locality-aware placement. Packing related pods within the same network block or rack reduces inter-node latency and improves throughput. Today, the Dynamo operator generates Grove PodCliqueSet resources from DGDs with no topology constraints, leaving users unable to express placement preferences through the DGD API.

Grove already provides Topology Aware Scheduling via its ClusterTopology CRD and TopologyConstraint fields on PCS, PCSG, and PodClique resources (GREP-244). The missing piece is a user-facing API on DGD that abstracts away Grove internals and translates deployment/service-level topology preferences into the correct PCS fields.

## Goals

* Allow DGD users to opt in to topology-aware scheduling via a simple, framework-agnostic API.

* Provide smart defaults (PCS=block, PCSG=rack) so users do not have to configure every level.

* Support per-service overrides for fine-grained control.

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

### REQ 4 User Overrides

Users **MUST** be able to override the deployment-level constraint and per-service constraints. When an override is provided, it **MUST** be used instead of the smart default.

### REQ 5 Hierarchy Validation

Service-level constraints **MUST** be equal to or narrower than the deployment-level constraint. The operator **MUST** reject a DGD where a service constraint is broader than the deployment constraint.

### REQ 6 Framework Agnosticism

Dynamo **MUST NOT** read or depend on framework-specific topology resources (e.g., Grove ClusterTopology). It **MUST** only inject resolved constraints into the generated spec. The underlying framework is responsible for validating domain existence and topology availability.

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

    // Per-service overrides. If nil for a service when enabled=true: smart default (rack).
    ServiceOverrides map[string]*TopologyConstraint `json:"serviceOverrides,omitempty"`
}

type TopologyConstraint struct {
    PackDomain TopologyDomain `json:"packDomain"`
}

type TopologyDomain string  // region|zone|datacenter|block|rack|host|numa
```

**User example:**

```yaml
spec:
  topologyConstraints:
    enabled: true
    deployment:
      packDomain: zone
    serviceOverrides:
      VllmWorker:
        packDomain: block
      Frontend:
        packDomain: zone
  services:
    - name: VllmWorker
      multinode: { nodeCount: 4 }
      replicas: 2
    - name: Frontend
      replicas: 1
```

**Mapping from DGD to generated spec (Grove PCS):**

| DGD | Grove | When | Value |
|-----|-------|------|--------|
| deployment | PCS.Spec.Template.TopologyConstraint | enabled=true | Override or default (block) |
| serviceOverrides[svc] | PCSG.TopologyConstraint | Multinode svc | Override or default (rack) |
| serviceOverrides[svc] | PodClique.TopologyConstraint | Single-node svc | Override or default (rack) |
| — | PodClique (multinode) | Multinode svc | No explicit constraint (inherit PCSG) |

## Smart Defaults and Override Resolution

Dynamo does not read ClusterTopology or any framework-specific topology resource. It computes resolved constraints from user overrides and smart defaults only.

| Level | Default | Rationale |
|-------|--------|-----------|
| PCS (deployment) | block | Good capacity/locality balance for the whole deployment |
| PCSG (multinode service) | rack | Packs service replicas; cliques inherit so workers can span hosts within a rack |
| Single-node clique | rack | Locality without over-constraining |

For each level: if user set a value, use it; otherwise use the smart default. Validate hierarchy (service ≤ deployment) and service-name validity. Dynamo does not validate domain existence; the framework does.

## PCS Generation

In `GenerateGrovePodCliqueSet` (or equivalent):

1. If `topologyConstraints` is nil or `enabled=false` → generate PCS without topology constraints.
2. If `enabled=true` → compute resolved constraints, validate hierarchy and service names, inject:
   - PCS template: deployment constraint (or default block).
   - For each multinode service: set PCSG constraint (or default rack); no explicit constraint on child cliques.
   - For each single-node service: set PodClique constraint (or default rack).

If the underlying framework rejects the PCS (e.g., ClusterTopology missing or invalid domain), the error surfaces at the framework level.

## Validation (Webhook)

- **Enabled flag:** If `enabled=false` and constraints are specified → reject.
- **Struct:** `PackDomain` **MUST** be set and be one of: region, zone, datacenter, block, rack, host, numa.
- **Hierarchy:** Each service constraint **MUST** be ≤ deployment (narrower or equal).
- **Service names:** Every key in `serviceOverrides` **MUST** exist in `spec.services`.

Dynamo does not validate domain existence or ClusterTopology presence.

## Framework Agnosticism and Topology Ownership

Dynamo does not read or check ClusterTopology (or any framework-specific topology resource). It injects resolved constraints into the generated PCS when `enabled=true`. The framework that owns the generated resource (e.g., Grove) is responsible for validation and error reporting if topology is missing or invalid.

Dynamo does not define or create topology CRs. When Grove is used, ClusterTopology and TAS are configured by the admin per Grove's documentation. Dynamo documents this prerequisite. No Dynamo RBAC for ClusterTopology is required.

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
| `deployment` + `serviceOverrides`, nil = default | Clear override semantics; smart defaults reduce configuration. |
| Multinode cliques inherit from PCSG | Avoids forcing many workers onto one host; respects capacity. |
| Single-node clique default: rack | Good locality without over-constraining. |
| Struct-based `TopologyConstraint` | Aligns with Grove; room for `FallbackDomain` later. |
| Dynamo does not read ClusterTopology | Keeps Dynamo framework-agnostic; framework validates and surfaces errors. |
| Dynamo does not define or create topology CRs | Admins set up topology per framework docs; Dynamo only documents the requirement. |

# Alternate Solutions

## Alt 1 Dynamo Reads ClusterTopology and Skips TAS When Missing

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
