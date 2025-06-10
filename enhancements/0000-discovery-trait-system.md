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

Introduce the **Discovery trait system** (`DiscoveryKV` and `DiscoveryPlane`) that provides structured etcd operations for `ComponentDescriptor`, `Namespace`, `Component`, and `Endpoint` entities. The `DiscoveryPlane` trait handles discovery operations (list, exists, join), while `DiscoveryKV` handles key-value operations. The `join()` method returns `Path` objects that implement `DiscoveryKV` for extended key operations, replacing unstructured etcd access patterns.

# Motivation

Prior to this enhancement, Namespace, Component, and Endpoint entities had limited validation rules and no standardized operations interface. Developers could perform arbitrary etcd operations without structure, leading to inconsistent discovery patterns and operational risks.

## Current Problems

1. **Unstructured Operations**: No standardized interface for discovery operations across entity types
2. **Limited Validation**: Minimal validation rules for namespace, component, and endpoint naming
3. **Inconsistent Patterns**: Each component implements its own discovery and listing logic
4. **No Type Safety**: Operations not validated against entity structure at compile time
5. **Poor Discoverability**: No clear way to enumerate available namespaces, components, or endpoints

## Dependencies

This proposal builds upon:
- **DEP: Component Descriptor Model** - Provides the foundational typed entities with rigorous naming conventions and validation rules

The ComponentDescriptor model establishes the strict validation rules that make these traits safe and reliable. See [Component Descriptor Model](deps/0000-component-descriptor-model.md) for detailed naming convention rationale.

## Goals

- **Standardized Operations**: Consistent interface across all discovery entity types
- **Type Safety**: Compile-time validation of discovery operations using ComponentDescriptor foundation
- **Discovery Capabilities**: List and enumerate Keys, Namespaces, Components, and Endpoints
- **Structured Access**: All operations use validated ComponentDescriptor paths with reserved keywords
- **Developer Experience**: Clear, discoverable API with rich listing capabilities

# Proposal

## Discovery Trait System

### 1. `DiscoveryKV` Trait - Key-Value Operations

```rust
/// Trait for key-value operations on etcd paths
pub trait DiscoveryKV {
    /// Atomically create a key if it does not exist
    async fn create(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<()>;

    /// Create key or validate existing value matches
    async fn create_or_validate(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<()>;

    /// Put/update a key-value pair
    async fn put(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<PutResponse>;

    /// Get value(s) for this key
    async fn get(&self) -> Result<Vec<KeyValue>>;

    /// Delete this key
    async fn delete(&self) -> Result<i64>;

    /// Watch for changes to this key/prefix
    async fn watch(&self) -> Result<PrefixWatcher>;

    /// Get the etcd path this entity represents
    fn path(&self) -> String;

    /// Get the display name for this entity (namespace.component.endpoint format)
    fn name(&self) -> String;
}
```

### 2. `DiscoveryPlane` Trait - Discovery Operations

```rust
/// Discovery operations for listing, existence checks, and path joining
pub trait DiscoveryPlane {
    /// List all entities under this path, returning structured collections
    async fn list(&self) -> Result<DiscoveryList>;

    /// Check if this entity exists
    async fn exists(&self) -> Result<bool>;

    /// Join additional path segments, returning a Path for KV operations
    fn join(&self, segments: &[&str]) -> Result<Path, DiscoveryError>;

    /// Get the ComponentDescriptor canonical representation with additional segments
    fn to_component_descriptor_with_segments(&self, segments: &[String]) -> String;

    /// Get the etcd path this entity represents
    fn path(&self) -> String;

    /// Get the display name for this entity
    fn name(&self) -> String;
}

/// Structured result from list operations
#[derive(Debug, Clone)]
pub struct DiscoveryList {
    /// Path entries found under _keys_ reserved word
    pub paths: Vec<String>,
    /// Nested namespaces
    pub namespaces: Vec<Namespace>,
    /// Components in this namespace
    pub components: Vec<Component>,
    /// Endpoints in this component
    pub endpoints: Vec<Endpoint>,
}
```

### 3. `Path` Object - Result of Join Operations

```rust
/// Represents an extended path created via join() operations
/// Implements DiscoveryKV for key-value operations
#[derive(Clone)]
pub struct Path {
    /// The base entity this path extends from
    pub base: Box<dyn DiscoveryPlane>,
    /// Additional path segments
    pub segments: Vec<String>,
}

impl std::fmt::Display for Path {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "{} @ {}", self.path(), self.component_descriptor())
    }
}

impl std::fmt::Debug for Path {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        write!(f, "Path({} @ {})", self.path(), self.component_descriptor())
    }
}

impl DiscoveryKV for Path {
    async fn create(&self, value: Vec<u8>, lease_id: Option<i64>) -> Result<()> {
        let etcd = get_etcd_client();
        etcd.create(self.path(), value, lease_id).await
    }

    fn path(&self) -> String {
        format!("{}/_keys_/{}",
            self.base.path(),
            self.segments.join("/")
        )
    }

            fn name(&self) -> String {
        format!("{}/{}",
            self.base.name(),
            self.segments.join("/")
        )
    }

    // ... other DiscoveryKV methods
}

impl Path {
    /// Get the ComponentDescriptor representation of this path
    fn component_descriptor(&self) -> String {
        // Convert base entity + segments back to ComponentDescriptor canonical format
        // This recreates the {namespace.component.endpoint:lease_id} format
        self.base.to_component_descriptor_with_segments(&self.segments)
    }
}
```

## Design Requirements

### 1. Naming and Path Conventions

**Display Names** use different delimiters to avoid ambiguity:
- **Namespace hierarchy**: `.` (DNS-like) → `production.api.v1`
- **Component/Endpoint separation**: `/` (filesystem-like) → `production.api.v1/gateway/grpc`

**Etcd Paths** use reserved keywords:
- Format: `dynamo://namespace/_component_/name/_endpoint_/name/_keys_/path`
- Reserved keyword ordering: `_component_` before `_endpoint_`, `_keys_` always last

### 2. Trait Responsibilities

**DiscoveryKV**: Pure key-value operations (create, put, get, delete, watch)
**DiscoveryPlane**: Discovery operations (list, exists, join)
**Path**: Result of `join()` operations, implements DiscoveryKV

### 3. Rich Debugging Format

All entities provide `path @ descriptor` format:
```
dynamo://production.api/_component_/gateway @ {production.api.gateway}
```

## Usage Examples

### 1. Basic Entity Operations

```rust
// Create entities - these implement both DiscoveryKV and DiscoveryPlane
let namespace = Namespace { hierarchy: "production.api".to_string(), runtime: drt };
let component = Component { namespace, name: "gateway".to_string() };
let endpoint = Endpoint { component, name: "grpc".to_string(), lease_id: None };

// Direct key-value operations via DiscoveryKV
component.put(serde_json::to_vec(&metadata)?, Some(lease_id)).await?;
let data = endpoint.get().await?;
let exists = namespace.exists().await?;
```

### 2. Path Extension with join()

```rust
// Use join() to create extended paths for additional data storage
let health_path = endpoint.join(&["health", "status"])?;  // Returns Path object
health_path.put(b"healthy".to_vec(), None).await?;
// Creates: dynamo://production.api/_component_/gateway/_endpoint_/grpc/_keys_/health/status

let config_path = component.join(&["config", "server"])?;
config_path.put(b"server_config".to_vec(), None).await?;
// Creates: dynamo://production.api/_component_/gateway/_keys_/config/server
```

### 3. Discovery Operations

```rust
// List entities and paths under a namespace
let discovery_list = namespace.list().await?;
println!("Found {} paths, {} components, {} endpoints",
    discovery_list.paths.len(),
    discovery_list.components.len(),
    discovery_list.endpoints.len()
);

// Rich debugging output shows both etcd path and canonical descriptor
println!("{}", endpoint);
// Output: dynamo://production.api/_component_/gateway/_endpoint_/grpc @ {production.api.gateway.grpc}
```

### 4. Reserved Keyword Patterns

The trait system enforces proper reserved keyword usage:
- `_component_` always comes before `_endpoint_` (enforced by entity structure)
- `_keys_` always comes last (automatically added by `join()`)
- Path structure validation prevents invalid combinations

```rust
// Valid patterns automatically generated:
namespace.join(&["config"])        // dynamo://ns/_keys_/config
component.join(&["health"])        // dynamo://ns/_component_/comp/_keys_/health
endpoint.join(&["metrics", "cpu"]) // dynamo://ns/_component_/comp/_endpoint_/ep/_keys_/metrics/cpu
```

## Implementation Strategy

### Phase 1: Foundation
- Implement Discovery traits for ComponentDescriptor entities
- Add `join()` method with Path object creation
- Establish naming convention patterns

### Phase 2: Integration
- Generic etcd client interface (removes `kv_*` method dependencies)
- Structured list parsing that returns DiscoveryList collections
- Rich Display/Debug implementations with `path @ descriptor` format

### Phase 3: Adoption
- Migration from unstructured etcd access patterns
- Developer tooling and validation
- Documentation and examples

## Generic Etcd Client Requirements

The trait implementations must use a generic etcd client interface:

```rust
trait EtcdClient {
    async fn create(&self, key: &str, value: Vec<u8>, lease_id: Option<i64>) -> Result<()>;
    async fn put_with_options(&self, key: &str, value: Vec<u8>, options: Option<PutOptions>) -> Result<PutResponse>;
    async fn get(&self, key: &str, options: Option<GetOptions>) -> Result<Vec<KeyValue>>;
    async fn delete(&self, key: &str, options: Option<DeleteOptions>) -> Result<i64>;
    async fn get_prefix(&self, prefix: &str) -> Result<Vec<KeyValue>>;
    async fn watch_prefix(&self, prefix: &str) -> Result<PrefixWatcher>;
}
```

**Benefits:**
- Removes dependencies on specific `kv_*` method names
- Enables testing with mock etcd clients
- Clean separation between discovery logic and etcd operations
- Flexible client implementations

# Benefits

1. **Clean Trait Separation**: `
