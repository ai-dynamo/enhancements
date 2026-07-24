# Dynamo Protocol Compatibility and Schema Evolution

**Status**: Draft

**Authors**: @nnshah1

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

---

## Summary

This DEP defines the schema compatibility requirements and evolution rules for Dynamo's internal protocols. It establishes guidelines for maintaining backwards and forwards compatibility between frontend, router, and backend components as schemas evolve.

---

## Motivation

Dynamo's distributed architecture involves multiple components (frontend, router, backends) that communicate via serialized data structures. As the platform evolves, these schemas must change to support new features. Without clear compatibility rules:

1. **Silent failures**: An old frontend may fail to deserialize a new backend's response
2. **Upgrade complexity**: All components must be upgraded simultaneously
3. **Testing gaps**: No clear contract for what combinations should work
4. **Inconsistent patterns**: Some fields use `#[serde(default)]`, others don't

### Current State

Analysis of `PreprocessedRequest` revealed inconsistent serde configuration:
- Some fields have `#[builder(default)]` but not `#[serde(default)]`
- This means the builder API accepts optional fields, but deserialization requires them
- Example: `eos_token_ids`, `mdc_sum`, `annotations`, `router_config_override`

---

## Goals

1. Document all protocol schemas that cross component boundaries
2. Establish clear rules for schema evolution (adding/removing/renaming fields)
3. Define serde configuration requirements for compatibility
4. Specify versioning strategy where needed
5. Create testing guidelines for compatibility verification

### Non-Goals

1. Changing the underlying transport mechanisms (NATS, gRPC, etc.)
2. Breaking changes to existing deployed versions
3. Defining application-level semantics (covered by Backend-Frontend Contract DEP)

---

## Schema Inventory

### Component Boundary Schemas

| Schema | Location | Direction | Transport |
|--------|----------|-----------|-----------|
| ModelDeploymentCard | `lib/llm/src/model_card.rs` | Backend → Frontend (via etcd) | etcd |
| PreprocessedRequest | `lib/llm/src/protocols/common/preprocessor.rs` | Frontend → Backend | NATS |
| PreprocessedEmbeddingRequest | `lib/llm/src/protocols/common/preprocessor.rs` | Frontend → Backend | NATS |
| LLMEngineOutput | `lib/llm/src/protocols/common/llm_backend.rs` | Backend → Frontend | NATS |
| BackendOutput | `lib/llm/src/protocols/common/llm_backend.rs` | Backend → Frontend | NATS |
| KV Cache Events | `components/src/dynamo/*/publisher.py` | Backend → Router | ZMQ/NATS |
| Health Check Payloads | Various | Frontend → Backend | NATS |
| WorkerStats/KvStats | `lib/llm/src/model_card.rs` | Backend → Router | etcd |

### etcd Key Structure

```
/{namespace}/models/{model_name}/
├── mdc                     # ModelDeploymentCard JSON
├── checksum                # MDC checksum for validation
└── endpoints/
    ├── {worker_id_1}       # Worker endpoint info
    └── {worker_id_2}
```

### NATS Subject Conventions

```
dynamo.{namespace}.{component}.{model_name}.generate
dynamo.{namespace}.{component}.{model_name}.prefill
dynamo.{namespace}.{component}.{model_name}.decode
```

---

## Compatibility Rules

### Rule 1: All Optional Fields MUST Have `#[serde(default)]`

**Requirement**: Every field with `#[builder(default)]` MUST also have `#[serde(default)]`.

```rust
// CORRECT
#[builder(default)]
#[serde(default, skip_serializing_if = "Option::is_none")]
pub mdc_sum: Option<String>,

// INCORRECT - will break deserialization if field is missing
#[builder(default)]
pub mdc_sum: Option<String>,
```

**Rationale**: The builder API and serde deserialization should have consistent optionality.

### Rule 2: No `deny_unknown_fields` on Protocol Types

**Requirement**: Protocol types that cross component boundaries MUST NOT use `#[serde(deny_unknown_fields)]`.

```rust
// CORRECT - unknown fields are ignored
#[derive(Serialize, Deserialize)]
pub struct PreprocessedRequest { ... }

// INCORRECT - will fail if sender has new fields
#[derive(Serialize, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct PreprocessedRequest { ... }
```

**Rationale**: Allows newer senders to work with older receivers (forward compatibility).

### Rule 3: Adding Fields

**Safe if**:
- New field has `#[serde(default)]` attribute
- Default value is semantically correct for old senders

**Compatibility Matrix**:
| Scenario | Result |
|----------|--------|
| New sender → Old receiver | ✅ Unknown field ignored |
| Old sender → New receiver | ✅ Gets default value |

### Rule 4: Removing Fields

**Safe if**:
- Field had `#[serde(default)]` on all deployed versions
- No receiver logic depends on the field being present

**Compatibility Matrix**:
| Scenario | Result |
|----------|--------|
| New sender → Old receiver (field had default) | ✅ Gets default |
| New sender → Old receiver (field was required) | ❌ Deserialization fails |
| Old sender → New receiver | ✅ Unknown field ignored |

### Rule 5: Renaming Fields

**Approach**: Use `#[serde(alias = "old_name")]` for transition period.

```rust
#[serde(alias = "old_field_name")]
pub new_field_name: Option<String>,
```

**Deprecation Timeline**:
1. Add alias, support both names
2. Update all senders to use new name
3. Remove alias after all receivers upgraded

### Rule 6: Changing Field Types

**Generally unsafe**. Requires:
- Coordinated upgrade of all components
- Or versioned schema with migration logic

---

## Versioning Strategy

### ModelDeploymentCard

MDC already has a `version` field for schema versioning:

```rust
pub struct ModelDeploymentCard {
    pub version: Option<String>,  // e.g., "1.0"
    // ...
}
```

### PreprocessedRequest

Currently no version field. Options:

**Option A**: Add version field (recommended)
```rust
#[serde(default = "default_protocol_version")]
pub protocol_version: String,
```

**Option B**: Rely on serde defaults (current approach)
- Works for additive changes only
- No explicit negotiation

### Version Negotiation

For breaking changes, implement version negotiation:
```
X-Dynamo-Protocol-Version: 1.0
```

Frontend checks backend's supported versions via MDC and routes accordingly.

---

## Serde Configuration Checklist

For any new protocol struct:

- [ ] No `#[serde(deny_unknown_fields)]`
- [ ] All `Option<T>` fields have `#[serde(default, skip_serializing_if = "Option::is_none")]`
- [ ] All `Vec<T>` fields that can be empty have `#[serde(default)]`
- [ ] All `HashMap<K,V>` fields that can be empty have `#[serde(default)]`
- [ ] Consider `#[serde(default)]` for structs with sensible defaults

---

## Testing Requirements

### Schema Compatibility Tests

```rust
#[test]
fn test_preprocessed_request_backwards_compat() {
    // Old format (missing new fields)
    let old_json = r#"{"model": "test", "token_ids": [1,2,3], ...}"#;

    // Should deserialize successfully with defaults
    let request: PreprocessedRequest = serde_json::from_str(old_json).unwrap();
    assert!(request.mdc_sum.is_none());  // Default applied
}

#[test]
fn test_preprocessed_request_forwards_compat() {
    // New format with unknown field
    let new_json = r#"{"model": "test", "token_ids": [1,2,3], "future_field": "ignored", ...}"#;

    // Should deserialize successfully, ignoring unknown
    let request: PreprocessedRequest = serde_json::from_str(new_json).unwrap();
}
```

### CI Integration

- Add schema compatibility tests to CI
- Test cross-version compatibility in integration tests
- Validate serde configuration in lint checks

---

## Implementation Checklist

### Phase 1: Fix Existing Inconsistencies

- [x] Add `#[serde(default)]` to `PreprocessedRequest` fields with `#[builder(default)]`
- [x] Add `#[serde(default)]` to `PreprocessedEmbeddingRequest` fields with `#[builder(default)]`
- [ ] Audit other protocol structs for same issue

### Phase 2: Add Compatibility Tests

- [ ] Create `tests/protocol_compatibility/` test module
- [ ] Add backwards compatibility tests for all protocol types
- [ ] Add forwards compatibility tests for all protocol types

### Phase 3: Documentation

- [ ] Document schema in JSON Schema format
- [ ] Add compatibility notes to struct documentation
- [ ] Create developer guide for protocol evolution

---

## Related Proposals

- [DEP-0000: Backend-Frontend Contract Formalization](0000-backend-frontend-contract.md) - Focuses on enforcement and validation
- [DEP-0000: Feature Stability Policy](0000-feature-stability-policy.md) - Defines stability tiers

---

## Alternate Solutions

### Alt 1: Protobuf for All Internal Communication

**Pros**:
- Built-in schema evolution rules
- Explicit field numbering
- Backwards compatible by design

**Cons**:
- Requires significant migration effort
- Less readable for debugging
- Additional tooling dependency

**Reason Rejected**: Current serde-based approach is working; adding proper defaults is sufficient.

### Alt 2: Runtime Schema Validation

**Pros**:
- Catch incompatibilities at runtime
- Detailed error messages

**Cons**:
- Performance overhead
- Complexity

**Reason Rejected**: Compile-time serde configuration is simpler and sufficient.

---

## References

- [Serde Attributes](https://serde.rs/attributes.html)
- [Protocol Buffers - Language Guide (proto3)](https://developers.google.com/protocol-buffers/docs/proto3)
- [API Evolution - Google Cloud](https://cloud.google.com/apis/design/compatibility)
