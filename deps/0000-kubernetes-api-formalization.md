# DEP: Kubernetes API Formalization

| Status | Draft |
|--------|-------|
| **Authors** | @nnshah1 |
| **Category** | Process |
| **Sponsor** | TBD |
| **Required Reviewers** | TBD |
| **Review Date** | TBD |
| **Related PRs** | #54 |

## Summary

This DEP defines the API formalization and versioning strategy for Dynamo's Kubernetes deployment components. It establishes which APIs require stabilization before 1.0 GA, defines graduation criteria from v1alpha1 to v1, and specifies deprecation policies.

## Motivation

All Dynamo Kubernetes CRDs are currently at `v1alpha1`, indicating unstable APIs that may change without notice. For enterprise adoption and 1.0 GA release, users need:

1. **API Stability Guarantees** - Clear understanding of which fields are stable
2. **Versioning Policy** - Predictable version graduation path
3. **Deprecation Policy** - Advance notice before breaking changes
4. **Schema Documentation** - Formal specifications for configuration contracts

### Current State

| API Surface | Count | State |
|-------------|-------|-------|
| CRDs | 5 | v1alpha1 |
| Labels/Annotations | 12+ | Ad-hoc, undocumented |
| Service Ports | 4 | Implicit |
| Environment Variables | 6+ | Implicit contracts |
| Helm Values | Multiple | No schema validation |
| Inter-component Contracts | 5 | Undocumented |

## Goals

1. Define version graduation criteria (v1alpha1 → v1beta1 → v1)
2. Document all public API surfaces requiring stabilization
3. Establish deprecation policy with minimum notice periods
4. Create formal schemas for configuration contracts
5. Define multi-version support windows

## Non-Goals

1. Implementation of conversion webhooks (separate DEP)
2. Multi-cluster federation APIs
3. Cloud-specific deployment APIs (AWS, GCP, Azure)

---

## API Inventory

### 1. Custom Resource Definitions (CRDs)

All CRDs are in API group `nvidia.com`, version `v1alpha1`.

#### DynamoGraphDeployment (DGD)

**Purpose**: Primary deployment interface for inference graphs.

| Field | Type | Stability | Notes |
|-------|------|-----------|-------|
| `spec.backendFramework` | enum | Must stabilize | `vllm`, `sglang`, `trtllm` |
| `spec.services` | map | Must stabilize | Max 25 items |
| `spec.pvcs` | array | Must stabilize | Max 100 items |
| `spec.envs` | array | Stable | Standard K8s EnvVar |
| `status.state` | string | Must stabilize | Lifecycle states |
| `status.conditions` | array | Stable | K8s Conditions |
| `status.services` | map | Must stabilize | ServiceReplicaStatus |

#### DynamoComponentDeployment (DCD)

**Purpose**: Per-component configuration (created by DGD controller).

| Field | Type | Stability | Notes |
|-------|------|-----------|-------|
| `spec.componentType` | string | Must stabilize | Taxonomy: planner, frontend, worker |
| `spec.subComponentType` | string | Must stabilize | prefill, decode |
| `spec.replicas` | int32 | Stable | Min 0 |
| `spec.resources` | object | Must stabilize | GPU, CPU, memory, custom |
| `spec.multinode` | object | Must stabilize | NodeCount (min 2) |
| `spec.scalingAdapter` | object | Must stabilize | DGDSA ownership |
| `spec.ingress` | object | Must stabilize | TLS, VirtualService |
| `spec.modelRef` | object | Must stabilize | Model discovery binding |
| `spec.autoscaling` | object | **Deprecated** | Use DGDSA |
| `spec.dynamoNamespace` | string | **Deprecated** | Implicit from DGD |

#### DynamoGraphDeploymentRequest (DGDR)

**Purpose**: SLA-based profiling and auto-configuration.

| Field | Type | Stability | Notes |
|-------|------|-----------|-------|
| `spec.model` | string | Stable | Required, model identifier |
| `spec.backend` | enum | Must stabilize | vllm, sglang, trtllm |
| `spec.useMocker` | bool | Stable | Default: false |
| `spec.enableGpuDiscovery` | bool | Stable | Default: false |
| `spec.profilingConfig` | object | Must stabilize | Needs JSON Schema |
| `spec.autoApply` | bool | Stable | Default: false |
| `spec.deploymentOverrides` | object | Must stabilize | DGD customization |
| `status.state` | enum | Must stabilize | Lifecycle states |
| `status.generatedDeployment` | RawExtension | Must stabilize | Full DGD |

#### DynamoGraphDeploymentScalingAdapter (DGDSA)

**Purpose**: Bridge between external autoscalers (HPA/KEDA) and DGD.

| Field | Type | Stability | Notes |
|-------|------|-----------|-------|
| `spec.replicas` | int32 | Stable | Required, min 0 |
| `spec.dgdRef.name` | string | Stable | Required |
| `spec.dgdRef.serviceName` | string | Stable | Required |
| `status.replicas` | int32 | Stable | For scale subresource |
| `status.selector` | string | Stable | HPA compatibility |
| `/scale` subresource | - | Must stabilize | HPA/KEDA contract |

#### DynamoModel (DM)

**Purpose**: Model service discovery and endpoint tracking.

| Field | Type | Stability | Notes |
|-------|------|-----------|-------|
| `spec.modelName` | string | Stable | Required |
| `spec.baseModelName` | string | Stable | Required, for discovery |
| `spec.modelType` | enum | Must stabilize | base, lora, adapter |
| `spec.source.uri` | string | Must stabilize | s3://, hf:// format |
| `status.endpoints` | array | Must stabilize | Pod discovery |
| `status.readyEndpoints` | int | Stable | Count |

---

### 2. Kubernetes Labels & Annotations

**Namespace**: `nvidia.com/`

| Label | Purpose | Set By | Read By |
|-------|---------|--------|---------|
| `nvidia.com/selector` | Component selection | Operator | K8s |
| `nvidia.com/dynamo-graph-deployment-name` | DGD identity | Operator | All |
| `nvidia.com/dynamo-component` | Component name | Operator | Discovery |
| `nvidia.com/dynamo-namespace` | Dynamo namespace | Operator | Runtime |
| `nvidia.com/dynamo-component-type` | planner/frontend/worker | Operator | Discovery |
| `nvidia.com/dynamo-sub-component-type` | prefill/decode | Operator | Router |
| `nvidia.com/dynamo-base-model` | Model identifier | Operator | DynamoModel |
| `nvidia.com/dynamo-base-model-hash` | Model hash | Operator | Discovery |
| `nvidia.com/dynamo-discovery-backend` | kubernetes/etcd | Operator | Runtime |
| `nvidia.com/dynamo-discovery-enabled` | Enable discovery | User | Operator |
| `nvidia.com/metrics-enabled` | Enable metrics | User | Operator |
| `nvidia.com/kai-scheduler-queue` | Kai scheduler queue | User | Operator |

---

### 3. Service Ports

| Port | Name | Protocol | Service | Owner |
|------|------|----------|---------|-------|
| 8000 | http | TCP | DYNAMO_PORT | All components |
| 9085 | metrics | TCP | Planner metrics | Planner |
| 9090 | system | TCP | Health/admin | All components |
| 2222 | ssh | TCP | MPI SSH | Multi-node workers |

---

### 4. Environment Variables

| Variable | Scope | Format | Purpose |
|----------|-------|--------|---------|
| `DYN_DEPLOYMENT_CONFIG` | Component | YAML | Deployment configuration |
| `DYN_NAMESPACE` | Component | String | Dynamo namespace identifier |
| `DYN_COMPONENT` | Component | String | Component name |
| `DYN_DISCOVERY_BACKEND` | Component | Enum | kubernetes/etcd |
| `DYNAMO_PORT` | Component | Integer | Service port (default: 8000) |
| `PROMETHEUS_ENDPOINT` | Component | URL | Metrics endpoint |

---

### 5. Inter-Component Contracts

#### DYN_DEPLOYMENT_CONFIG Schema

```yaml
# Needs formal JSON Schema
namespace: string      # Dynamo namespace
component: string      # Component name
endpoints:             # Service discovery
  - name: string
    address: string
routing:               # Router configuration
  mode: string         # kv, round-robin, random
```

#### Model Discovery Contract

1. DynamoModel creates headless Service
2. Pods labeled with `nvidia.com/dynamo-base-model`
3. Endpoints discovered via K8s Endpoints API
4. LoRA readiness probed via POST `/loras`

#### DGDSA Scaling Contract

1. DGDSA implements `/scale` subresource
2. HPA/KEDA reads `status.replicas` and `status.selector`
3. Scaling writes to `spec.replicas`
4. Operator syncs to DGD `services[name].replicas`

---

## Versioning Strategy

### Graduation Criteria

#### v1alpha1 → v1beta1

- [ ] All `Must stabilize` fields locked
- [ ] JSON Schemas for complex types (profilingConfig, deploymentOverrides)
- [ ] Label registry documented
- [ ] Port registry documented
- [ ] Environment variable contracts documented
- [ ] Webhook validation for all required fields
- [ ] At least 2 releases in production use

#### v1beta1 → v1

- [ ] Deprecation policy enforced for 2+ releases
- [ ] Conversion webhooks for v1alpha1 → v1
- [ ] API reference documentation complete
- [ ] Upgrade testing with real workloads
- [ ] No breaking changes for 3+ releases

### Multi-Version Support

| Version | Status | Support Window |
|---------|--------|----------------|
| v1alpha1 | Current | Until v1beta1 + 2 releases |
| v1beta1 | Planned | Until v1 + 2 releases |
| v1 | GA Target | Long-term support |

---

## Deprecation Policy

### Field Deprecation

1. **Notice Period**: Minimum 2 releases before removal
2. **Documentation**: Deprecation noted in:
   - CRD field description
   - Release notes
   - Migration guide
3. **Warnings**: Webhook emits warning for deprecated field usage
4. **Removal**: Field removed in subsequent major version

### Currently Deprecated Fields

| CRD | Field | Deprecated In | Replacement | Remove In |
|-----|-------|---------------|-------------|-----------|
| DCD | `spec.autoscaling` | v0.7.0 | DGDSA | v1.0 |
| DCD | `spec.dynamoNamespace` | v0.8.0 | Implicit from DGD | v1.0 |

---

## Helm Chart Versioning

### Chart Version Strategy

| Chart | Current | 1.0 Target |
|-------|---------|------------|
| dynamo-platform | 0.8.0 | 1.0.0 |
| dynamo-crds | 0.8.0 | 1.0.0 |
| dynamo-operator | 0.8.0 | 1.0.0 |

### Values Schema

Create `values.schema.json` for:
- Required fields validation
- Type checking
- Default value documentation
- Deprecation warnings

---

## Implementation Checklist

### Phase 1: Documentation (v0.9)

- [ ] Create label registry document
- [ ] Create port registry document
- [ ] Create environment variable contract document
- [ ] Document DYN_DEPLOYMENT_CONFIG schema
- [ ] Document model discovery contract

### Phase 2: Schema Validation (v0.10)

- [ ] JSON Schema for profilingConfig
- [ ] JSON Schema for deploymentOverrides
- [ ] Helm values.schema.json
- [ ] Webhook validation enhancement

### Phase 3: v1beta1 Graduation (v0.11)

- [ ] Lock all `Must stabilize` fields
- [ ] Create v1beta1 API types
- [ ] Implement conversion webhooks
- [ ] Update documentation

### Phase 4: v1 GA (v1.0)

- [ ] Complete deprecation of v1alpha1 fields
- [ ] Full API reference documentation
- [ ] Upgrade testing
- [ ] Release v1 CRDs

---

## Alternate Solutions

### Option A: Skip v1beta1

Graduate directly from v1alpha1 to v1. **Rejected** because:
- Insufficient testing period for API stability
- No opportunity for feedback on stabilized fields
- Risk of breaking changes in GA

### Option B: Per-CRD Versioning

Allow different CRDs to have different versions. **Rejected** because:
- Increases complexity for users
- Makes compatibility matrix confusing
- Harder to document and test

### Option C: External Schema Registry

Use external schema registry (e.g., Apicurio). **Deferred** because:
- Adds infrastructure dependency
- Overkill for current scale
- Can revisit post-1.0

---

## Related Proposals

- DEP-0000: Dynamo API and Non-Feature Requirements (parent)
- DEP-XXXX: CRD Conversion Webhooks (future)
- DEP-XXXX: Multi-Cluster Federation (future)

---

## References

- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [Kubernetes Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)
- [CRD Versioning](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definition-versioning/)
