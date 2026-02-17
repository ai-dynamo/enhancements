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

* Require explicit deployment-level and per-service topology constraints when TAS is used (no defaults). TAS is in effect when `spec.topologyConstraint` is set.

* Maintain full backward compatibility with existing DGDs (no TAS unless topology constraints are provided).

* Validate topology at DGD creation time (webhook) so users get immediate feedback; when `spec.topologyConstraint` is set, the webhook must validate against the framework's topology CRD and reject the DGD with an error if validation fails or the CRD cannot be read.

### Non Goals

* Dynamo will not define, create, or manage topology CRs (e.g., Grove ClusterTopology). Admins configure topology in the underlying framework per its documentation.

* Dynamic updates to topology constraints after PCS creation are out of scope (blocked by Grove's immutability).

* Backend- or framework-specific default strategies (e.g., different defaults for vLLM vs TRT-LLM) are out of scope for the initial implementation.

* Generalizing the domain vocabulary for non-Grove backends is out of scope. The abstract domain names (region, zone, datacenter, block, rack, host, numa) align with Grove today. For backends that use node labels (e.g., Kueue Topology CRD), a `domainToLabelMapping` in operator config (e.g., `rack` → `cloud.provider.com/topology-rack`) can be introduced so the webhook can validate and translate; only needed when that backend is supported.

* Multi-topology support (selecting a named topology definition per deployment) is out of scope for the initial implementation; it is still in design at the Grove/KAI scheduler level. See Future Work.

## Requirements

### REQ 1 Opt-In by Providing Constraints

TAS **MUST** be opt-in: it is in effect when the user sets `spec.topologyConstraint`. If `topologyConstraint` is nil, no TAS is applied. Existing DGDs without topology constraints **MUST** continue to work unchanged after an operator upgrade. This avoids violating Grove's immutability rules on existing PCS resources.

### REQ 2 API Abstraction

The DGD API **MUST NOT** expose Grove-specific terms (PCS, PCSG, PodClique). Users **SHOULD** specify constraints in terms of deployment-level and per-service only.

### REQ 3 Explicit Constraints When TAS Used

When `spec.topologyConstraint` is set (TAS in effect), each service's `topologyConstraint` **MUST** also be explicitly set by the user. There are no defaults; the operator uses only the user-specified values.

**Note on constraint type:** Today, all topology constraints in the underlying framework (Grove) are hard/required constraints — there is no "preferred" or soft mode. If the scheduler cannot satisfy the constraint, the workload will not be scheduled. This is acceptable because TAS is explicitly opt-in and users specify constraints knowingly. When the underlying framework adds support for preferred/soft constraints, the API could be extended accordingly.

### REQ 5 Hierarchy Validation

Service-level constraints **MUST** be equal to or narrower than the deployment-level constraint. The operator **MUST** reject a DGD where a service constraint is broader than the deployment constraint.

### REQ 6 Framework Agnosticism

The high-level DGD API (topology constraints, `packDomain`) **MUST** be the same for all supported frameworks. Users express topology in Dynamo's abstract vocabulary (e.g., `region`, `zone`, `block`, `rack`, `host`, `numa`), not in framework-specific terms. Future internal work may be required to translate this API to certain frameworks (e.g., mapping abstract domains to framework-specific node labels); the DGD API itself does not change.

**Why abstract domain names instead of node labels:** Node labels are cluster-specific and cloud-provider-specific (e.g., `cloud.provider.com/topology-rack` vs `topology.kubernetes.io/rack`), so DGDs written with labels would not be portable across clusters. Labels also do not work with Grove, whose `PackDomain` takes a domain name (e.g., `rack`), not a label; using labels would require a fragile reverse-lookup from label to domain name. Abstract domains let users express intent in natural terms ("pack in the same rack") without knowing infrastructure-level label strings. The operator handles framework-specific translation internally: for Grove the domains map 1:1; for future backends like Kueue a `domainToLabelMapping` config translates without changing the DGD API.

### REQ 7 Backward Compatibility

No automatic TAS application on existing DGDs. Users opt in by setting `spec.topologyConstraint` (and per-service `topologyConstraint`); to get TAS on an existing DGD they must recreate it with constraints set.

# Proposal

## API Design

Add an optional `topologyConstraint` field to `DynamoGraphDeploymentSpec` (deployment-level) and to each `ServiceSpec` (service-level). Both use the same `TopologyConstraint` struct:

```go
type DynamoGraphDeploymentSpec struct {
    // ... existing fields ...

    // TopologyConstraint is the deployment-level topology constraint.
    // When set, TAS is in effect and each service must also have topologyConstraint set.
    // When nil, no TAS is applied.
    // +optional
    TopologyConstraint *TopologyConstraint `json:"topologyConstraint,omitempty"`
}

type ServiceSpec struct {
    // ... existing fields (name, replicas, multinode, etc.) ...

    // TopologyConstraint for this service. Required when spec.topologyConstraint is set.
    // +optional
    TopologyConstraint *TopologyConstraint `json:"topologyConstraint,omitempty"`
}

type TopologyConstraint struct {
    PackDomain TopologyDomain `json:"packDomain"`
}

type TopologyDomain string  // region|zone|datacenter|block|rack|host|numa
```

**User example:**

```yaml
spec:
  topologyConstraint:
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
| spec.topologyConstraint | PCS.Spec.Template.TopologyConstraint | when set | User-specified |
| service.topologyConstraint | PCSG.TopologyConstraint | Multinode svc | User-specified |
| service.topologyConstraint | PodClique.TopologyConstraint | Single-node svc | User-specified |
| — | PodClique (multinode) | Multinode svc | No explicit constraint (inherit PCSG) |

## Constraint Resolution

The controller uses only user-specified constraints when `spec.topologyConstraint` is set; it does not read the topology CRD. The webhook (see Validation) reads the topology CRD for admission-time validation. When `spec.topologyConstraint` is set, each service's `topologyConstraint` is also required; the controller injects these values into the generated PCS. Validate `packDomain` against the known `TopologyDomain` values and validate hierarchy (service ≤ spec-level). The webhook validates that each `packDomain` is a valid level in the cluster's topology (see Validation).

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

In `GenerateGrovePodCliqueSet` (or equivalent):

1. If `spec.topologyConstraint` is nil → generate PCS without topology constraints. Any `topologyConstraint` on services is ignored.
2. If `spec.topologyConstraint` is set → for each service, use its `topologyConstraint` (required), validate hierarchy, then inject:
   - PCS template: spec-level constraint (user-specified).
   - For each multinode service: set PCSG constraint from service's `topologyConstraint`; no explicit constraint on child cliques (they inherit from PCSG).
   - For each single-node service: set PodClique constraint from service's `topologyConstraint`.

If the underlying framework rejects the PCS (e.g., ClusterTopology missing or invalid domain), the error surfaces at the framework level.

## Validation (Webhook)

### On CREATE

- **Inference:** TAS is in effect when `spec.topologyConstraint` is set. If `spec.topologyConstraint` is nil but any service has `topologyConstraint` → reject: "spec-level topologyConstraint is required when any service specifies topologyConstraint".
- **Required when TAS in effect:** When `spec.topologyConstraint` is set, each service **MUST** have `topologyConstraint` set.
- **Struct:** `PackDomain` **MUST** be set and be one of the known `TopologyDomain` values: `region`, `zone`, `datacenter`, `block`, `rack`, `host`, `numa`.
- **Hierarchy:** Each service's `topologyConstraint` **MUST** be ≤ `spec.topologyConstraint` (narrower or equal).
- **Topology CRD validation:** When `spec.topologyConstraint` is set, the webhook **MUST** read the framework's topology CRD (e.g., Grove ClusterTopology) to validate that each `packDomain` (spec-level and per-service) is a valid level in the cluster's topology (single/default topology). For Grove, domain names match 1:1 (rack = rack). If the CRD cannot be read (e.g., RBAC missing, CRD not installed) or validation fails, the webhook **MUST** reject the DGD with an error.
- **RBAC:** The Dynamo operator needs read-only access to the framework's topology CRD (e.g., `clustertopologies.grove.io`) for webhook validation when that backend is in use.

### On UPDATE

All topology-related fields are **immutable** once the DGD is created, mirroring Grove's immutability of topology constraints on PCS. The following changes **MUST** be rejected:

- **`spec.topologyConstraint`:** Cannot be added, removed, or have `packDomain` changed (adding/removing would require changing constraints on PCS → Grove rejects).
- **Service `topologyConstraint`:** Cannot be added, removed, or have `packDomain` changed on any existing service.

To change topology constraints, users must delete and recreate the DGD (which creates a new PCS). This aligns with Grove's model and avoids PCS update failures.

### General

When `spec.topologyConstraint` is set, the webhook must validate domain existence against the topology CRD; if the CRD cannot be read or validation fails, the webhook rejects the DGD with an error.

## Framework Agnosticism and Topology Ownership

**Abstract vocabulary:** `PackDomain` uses Dynamo's own abstract topology vocabulary (`region`, `zone`, `datacenter`, `block`, `rack`, `host`, `numa`) — defined as a `TopologyDomain` type in the DGD API. These are Dynamo's own domain names, not "Grove's domains leaked into Dynamo"; they align with Grove today because both use natural, user-friendly topology terms. The alternative — using node labels directly — would make DGDs non-portable across clusters (labels are cloud/cluster-specific, e.g., `cloud.provider.com/topology-rack` vs `topology.kubernetes.io/rack`) and would not work with Grove (whose `PackDomain` takes domain names, not labels). Dynamo's webhook validates that `packDomain` is one of these known names (catches typos at admission). The framework validates that the domain is a valid level in the cluster's topology. For future non-Grove backends (e.g., Kueue), a translation layer (e.g., operator config mapping abstract domains to framework-specific node labels) can translate without changing the DGD API.

**Webhook vs controller:** The webhook reads the framework's topology CRD when `spec.topologyConstraint` is set and rejects the DGD with an error if the CRD cannot be read or validation fails. The controller does not read the topology CRD; it injects constraints when `spec.topologyConstraint` is set.

Dynamo does not define or create topology CRs. When Grove is used, the cluster's topology (ClusterTopology) is configured by the admin per Grove's documentation. Dynamo requires read-only RBAC for the framework's topology CRD when that backend is in use (for webhook validation).

## Status and Observability

- **Status:** Add `AppliedTopologyConstraints` to DGD status reporting the applied constraints when `spec.topologyConstraint` is set.
- **Events:** Normal when constraints are applied; Warning/Error on validation failures.
- **Logging:** Info when constraints are applied; Error when validation fails.

## Migration and Backward Compatibility

- `spec.topologyConstraint` nil → no TAS (backward compatible).
- TAS is applied when `spec.topologyConstraint` is set. The webhook must validate topology in that case and rejects the DGD with an error if the topology CRD cannot be read or validation fails.
- Existing DGDs are unchanged; no automatic topology constraints on existing PCS.
- Rollout: deploy updated operator; when using Grove, admins configure ClusterTopology per Grove docs; users set `spec.topologyConstraint` (and per-service constraints) on new or recreated DGDs to use TAS.

## Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| TAS in effect when `spec.topologyConstraint` is set | Simpler API; providing constraints opts in; backward compatible (nil = no TAS). No wrapper struct needed. |
| Same `topologyConstraint` field on spec and services | Consistent pattern at both levels; no wrapper struct; mirrors Grove's inline approach. |
| Multinode cliques inherit from PCSG | Constraint set on PCSG; child cliques have no explicit constraint, avoiding over-constraining many workers to one host. |
| Struct-based `TopologyConstraint` | Aligns with Grove; room for `FallbackDomain` later. |
| Webhook reads topology CRD for validation; rejects on failure | Gives users immediate feedback at DGD creation; controller does not read CRD. Rejects DGD with error if topology CRD unreadable or validation fails. |
| Dynamo does not define or create topology CRs | Admins set up topology per framework docs; Dynamo only documents the requirement. |
| No smart defaults; user must set spec.topologyConstraint and all service constraints when using TAS | Explicit configuration; no implicit defaults. |
| `TopologyDomain` is a fixed enum, not configurable | Domain names (region, zone, etc.) are Dynamo's own abstract vocabulary; the framework validates that a domain exists in the referenced topology. For non-Grove backends, a translation layer can map abstract domains to framework-specific values. |
| Hard constraints (for now) | Only mode available in Grove today; preferred/soft mode can be adopted when the framework supports it. |
| Topology fields immutable on UPDATE | Mirrors Grove's PCS immutability; avoids update failures. Delete and recreate to change. |

## Future Work

* **Multi-topology support:** Multiple named topology definitions per cluster (e.g., one for H100, one for GB200 node pools) is still in design at the Grove/KAI scheduler level. Once that is available, Dynamo could add an optional field (e.g., `topologyName`) to the DGD API so users can select which topology applies to their deployment; the operator would pass it through to the generated PCS (e.g., `clusterTopologyName` in Grove).

* **Smart defaults enablement:** Smart defaults (e.g., deployment=block, service=rack when constraints are omitted) were considered but deferred. Block and rack domains vary from cluster to cluster, and optimal placement varies by workload, making a single good default difficult; for an opt-in feature, defaults are nice-to-have rather than a requirement, and the design errs on the side of not introducing defaults that could cause issues. Once Grove has support for auto-packing (or equivalent), Dynamo could introduce smart defaults—for example, making that framework behavior the default when the user sets `spec.topologyConstraint` but omits some service constraints.

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
