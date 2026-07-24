# Discovery Trait System for Structured Etcd Operations

**Status**: Draft

**Authors**: [ryanolson](https://github.com/ryanolson)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [nnshah1](https://github.com/nnshah1)

**Required Reviewers**: [dzier](https://github.com/dzier), [statiraju](https://github.com/statiraju), [kkranen](https://github.com/kkranen)

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Introduce the **Discovery trait system** that provides structured etcd operations for entities created from `Instance` descriptors. The system includes four traits: `DiscoveryKV` (key-value operations), `DiscoveryPlane` (discovery operations), `EtcdIdentity` (etcd path mapping), and `PathExtension` (path creation). These traits replace unstructured etcd access patterns with type-safe operations.

# Motivation

Prior to this enhancement, Namespace, Component, and Endpoint entities had limited validation rules and no standardized operations interface. Developers could perform arbitrary etcd operations without structure, leading to inconsistent discovery patterns and operational risks.

## Current Problems

1. **Unstructured Operations**: No standardized interface for discovery operations
2. **Inconsistent Patterns**: Each component implements its own discovery logic
3. **No Type Safety**: Operations not validated at compile time
4. **Poor Discoverability**: No clear way to enumerate entities

## Dependencies

This proposal builds upon:
- **DEP: Instance Model** - Provides `Instance` descriptors for creating typed entities and the `Identity` trait

This proposal adds discovery operations through new traits (`EtcdIdentity`, `PathExtension`, `DiscoveryKV`, `DiscoveryPlane`) and the `Path` type for extended paths. A unified `canonical_to_etcd_path()` function provides deterministic transformation from canonical strings to etcd paths.

## Goals

- **Standardized Operations**: Consistent interface across all entity types
- **Type Safety**: Compile-time validation of discovery operations
- **Discovery Capabilities**: List and enumerate paths, namespaces, components, and endpoints
- **Developer Experience**: Clear, discoverable API with deterministic path transformation

# Proposal

### 1. `DiscoveryKV` Trait - Key-Value Operations

```rust
pub trait DiscoveryKV {
    async fn create(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<()>;
    async fn create_or_validate(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<()>;
    async fn put(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<PutResponse>;
    async fn get(&self) -> Result<Vec<KeyValue>>;
    async fn delete(&self) -> Result<i64>;
    async fn watch(&self) -> Result<PrefixWatcher>;
}
```

### 2. `DiscoveryPlane` Trait - Discovery Operations

```rust
pub trait DiscoveryPlane {
    async fn list(&self) -> Result<DiscoveryList>;
    async fn exists(&self) -> Result<bool>;
}

#[derive(Debug, Clone)]
pub struct DiscoveryList {
    pub paths: Vec<String>,
    pub namespaces: Vec<Namespace>,
    pub components: Vec<Component>,
    pub endpoints: Vec<Endpoint>,
}
```

### 3. `EtcdIdentity` Trait - Etcd Path Mapping

```rust
#[derive(Debug, Clone, Copy)]
pub enum EntityType {
    Namespace,
    Component,
    Endpoint,
    Path,
}

/// Trait for types that have a representation in etcd
pub trait EtcdIdentity: Identity {
    fn entity_type(&self) -> EntityType;

    /// Transforms canonical string to etcd path
    fn etcd_path(&self) -> String {
        canonical_to_etcd_path(&self.to_string(), self.entity_type())
    }
}

/// Transform canonical identifier to etcd path: {a.b.c} → dynamo://a/b/c
fn canonical_to_etcd_path(canonical: &str, entity_type: EntityType) -> String {
    let trimmed = canonical.trim_start_matches('{').trim_end_matches('}');
    let (path_part, lease_suffix) = trimmed.rsplit_once(':').map_or((trimmed, ""), |(p, l)| (p, &trimmed[p.len()..]));
    let parts: Vec<&str> = path_part.split('.').collect();

    match entity_type {
        EntityType::Namespace => format!("dynamo://{}", parts.join("/")),
        EntityType::Component => {
            let (ns_parts, comp) = parts.split_at(parts.len() - 1);
            format!("dynamo://{}/_component_/{}", ns_parts.join("/"), comp[0])
        },
        EntityType::Endpoint => {
            let (ns_and_comp, ep) = parts.split_at(parts.len() - 1);
            let (ns_parts, comp) = ns_and_comp.split_at(ns_and_comp.len() - 1);
            format!("dynamo://{}/_component_/{}/_endpoint_/{}{}",
                ns_parts.join("/"), comp[0], ep[0], lease_suffix)
        },
        EntityType::Path => panic!("Path entities must override etcd_path()"),
    }
}
```

### 4. `PathExtension` Trait - Path Creation

```rust
/// Trait for types that can be extended with path segments
pub trait PathExtension: EtcdIdentity {
    fn to_path(&self, segments: &[&str]) -> Result<Path, InstanceError> {
        for segment in segments {
            validate_path_segment(segment)?;
        }
        Ok(Path {
            base: Box::new(self.clone()),
            segments: segments.iter().map(|s| s.to_string()).collect(),
        })
    }
}
```

### 5. Implementation Details

```rust
/// Extended path created via to_path()
#[derive(Clone)]
pub struct Path {
    pub base: Box<dyn EtcdIdentity>,
    pub segments: Vec<String>,
}

// Display implementations
impl std::fmt::Display for Path {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} @ {}", self.etcd_path(), self.to_string())
    }
}

impl std::fmt::Debug for Path {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Path({} @ {})", self.etcd_path(), self.to_string())
    }
}

// Trait implementations for entity types
impl EtcdIdentity for Namespace {
    fn entity_type(&self) -> EntityType { EntityType::Namespace }
}

impl EtcdIdentity for Component {
    fn entity_type(&self) -> EntityType { EntityType::Component }
}

impl EtcdIdentity for Endpoint {
    fn entity_type(&self) -> EntityType { EntityType::Endpoint }
}

impl EtcdIdentity for Path {
    fn entity_type(&self) -> EntityType { EntityType::Path }

    fn etcd_path(&self) -> String {
        if self.segments.is_empty() {
            self.base.etcd_path()
        } else {
            format!("{}/_keys_/{}", self.base.etcd_path(), self.segments.join("/"))
        }
    }
}

impl Identity for Path {
    fn to_string(&self) -> String {
        if self.segments.is_empty() {
            self.base.to_string()
        } else {
            format!("{}/{}", self.base.to_string(), self.segments.join("/"))
        }
    }
}

// All etcd identities can create paths
impl PathExtension for Namespace {}
impl PathExtension for Component {}
impl PathExtension for Endpoint {}
impl PathExtension for Path {}

// Path implements discovery operations
impl DiscoveryKV for Path {
    async fn create(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<()> {
        get_etcd_client().create(self.etcd_path(), value, lease_id).await
    }
    // ... other methods
}

impl DiscoveryPlane for Path {
    async fn list(&self) -> Result<DiscoveryList> {
        get_etcd_client().list_prefix(&self.etcd_path()).await
    }

    async fn exists(&self) -> Result<bool> {
        Ok(!self.get().await?.is_empty())
    }
}
```

## Design

### Trait Hierarchy

```
Identity (from Instance model)
    ↑
EtcdIdentity - Maps to etcd paths
    ↑
PathExtension - Creates Path objects
```

### Implementation Matrix

| Type | Identity | EtcdIdentity | PathExtension | DiscoveryKV | DiscoveryPlane |
|------|----------|--------------|---------------|-------------|----------------|
| Instance | ✓ | ✗ | ✗ | ✗ | ✗ |
| Namespace, Component, Endpoint | ✓ | ✓ | ✓ | ✗ | ✗ |
| Path | ✓ | ✓ | ✓ | ✓ | ✓ |

### Canonical to Etcd Path Transformation

The system transforms canonical identifiers to etcd paths deterministically:

| Canonical | Etcd Path |
|-----------|-----------|
| `{production.api.v1}` | `dynamo://production/api/v1` |
| `{production.api.v1.gateway}` | `dynamo://production/api/v1/_component_/gateway` |
| `{production.api.v1.gateway.http}` | `dynamo://production/api/v1/_component_/gateway/_endpoint_/http` |
| `{production.api.v1.gateway}/config` | `dynamo://production/api/v1/_component_/gateway/_keys_/config` |

Reserved keywords follow strict ordering: `_component_` → `_endpoint_` → `_keys_`

## Usage Examples

```rust
// Create entities from Instance descriptors
let namespace_desc = Instance::new_namespace("production.api")?;
let component_desc = Instance::new_component("production.api", "gateway")?;
let endpoint_desc = Instance::new_endpoint("production.api", "gateway", "grpc")?;

let drt = DistributedRuntime::new();
let namespace = namespace_desc.to_namespace(&drt);
let component = component_desc.to_component(&drt).unwrap();
let endpoint = endpoint_desc.to_endpoint(&drt).unwrap();

// Path creation and KV operations
let config_path = component.to_path(&["config", "server"])?;
config_path.put(b"server_config".to_vec(), None).await?;

let health_path = endpoint.to_path(&["health", "status"])?;
health_path.put(b"healthy".to_vec(), None).await?;

// Discovery operations
let discovery_list = namespace.list().await?;
println!("Found {} paths, {} components, {} endpoints",
    discovery_list.paths.len(),
    discovery_list.components.len(),
    discovery_list.endpoints.len()
);

// Debug output shows etcd path @ canonical
println!("{}", endpoint);
// dynamo://production/api/_component_/gateway/_endpoint_/grpc @ {production.api.gateway.grpc}

// Instance is just a descriptor - must create entities first
let instance = Instance::new_endpoint("ns", "comp", "ep")?;
// instance.etcd_path()     // Compile error!
// instance.to_path(&["x"]) // Compile error!
```
