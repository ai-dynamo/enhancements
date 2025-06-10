# Component Descriptor Model for Structured Etcd Paths

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

Introduce `ComponentDescriptor` as the foundational identifier system for distributed components, providing THE canonical string representation for namespaces, components, and endpoints. This establishes a robust foundation with strong validation rules and type safety, while separating identifier concerns from etcd path generation and key-value operations.

# Motivation

Dynamo requires a structured approach to component identification that enforces hierarchical naming conventions and provides canonical string identifiers for distributed system operations.

## Requirements for ComponentDescriptor

1. **Canonical Identifiers**: Provide THE authoritative string representation for components via `to_string()`
2. **Typed Entity Creation**: Create `Namespace`, `Component`, and `Endpoint` entities from descriptors
3. **Hierarchical Navigation**: Support parent/child relationships in namespace hierarchies
4. **Validation Enforcement**: Implement robust validation rules for all identifier components
5. **Separation of Concerns**: Focus purely on identification, not etcd path generation or key-value operations

## Goals

- **Canonical Identification**: `ComponentDescriptor` provides THE standard string identifier for all components
- **Entity Type System**: Create typed `Namespace`, `Component`, and `Endpoint` entities from descriptors
- **Hierarchical Navigation**: Support parent/child traversal in namespace hierarchies
- **Strong Validation**: Implement comprehensive validation for component identifiers
- **Clean Separation**: Identifier logic separate from etcd operations and key-value storage

# Proposal

## Core Type Hierarchy

### 1. ComponentDescriptor Foundation

```rust
/// Provides THE canonical identifier for distributed components
/// Focuses purely on identification and typed entity creation
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct ComponentDescriptor {
    pub namespace: String,
    pub component: Option<String>,
    pub endpoint: Option<String>,
    pub lease_id: Option<i64>,
}
```

### 2. Typed Entity Structs

```rust
/// Represents a validated namespace hierarchy with runtime context
#[derive(Debug, Clone)]
pub struct Namespace {
    pub hierarchy: String,
    pub runtime: DistributedRuntime,
}

/// Represents a component within a namespace
#[derive(Debug, Clone)]
pub struct Component {
    pub namespace: Namespace,
    pub name: String,
}

/// Represents an endpoint within a component
#[derive(Debug, Clone)]
pub struct Endpoint {
    pub component: Component,
    pub name: String,
    pub lease_id: Option<i64>,
}
```

### 3. Entity Creation and Canonical Identification

```rust
impl ComponentDescriptor {
    /// Provides THE canonical string identifier for this component
    /// This is the authoritative representation used throughout the system
    pub fn to_string(&self) -> String {
        let mut identifier = format!("{{{}}}", self.namespace);

        if let Some(component) = &self.component {
            identifier.push_str(&format!(".{}", component));

            if let Some(endpoint) = &self.endpoint {
                identifier.push_str(&format!(".{}", endpoint));

                if let Some(lease_id) = self.lease_id {
                    identifier.push_str(&format!(":{:x}", lease_id));
                }
            }
        }

        identifier.push('}');
        identifier
    }

    /// Create a Namespace entity from this descriptor
    pub fn to_namespace(&self, drt: &DistributedRuntime) -> Namespace {
        Namespace {
            hierarchy: self.namespace.clone(),
            runtime: drt.clone(),
        }
    }

    /// Create a Component entity from this descriptor (if component is present)
    pub fn to_component(&self, drt: &DistributedRuntime) -> Option<Component> {
        self.component.as_ref().map(|name| Component {
            namespace: self.to_namespace(drt),
            name: name.clone(),
        })
    }

    /// Create an Endpoint entity from this descriptor (if endpoint is present)
    pub fn to_endpoint(&self, drt: &DistributedRuntime) -> Option<Endpoint> {
        if let Some(component) = self.to_component(drt) {
            self.endpoint.as_ref().map(|name| Endpoint {
                component,
                name: name.clone(),
                lease_id: self.lease_id,
            })
        } else {
            None
        }
    }
}
```

### 4. Hierarchical Navigation

```rust
impl Namespace {
    /// Get individual namespace segments
    pub fn segments(&self) -> Vec<&str> {
        self.hierarchy.split('.').collect()
    }

    /// Get parent namespace (returns None if this is root)
    pub fn parent(&self) -> Option<Namespace> {
        let segments = self.segments();
        if segments.len() <= 1 {
            return None;
        }

        let parent_hierarchy = segments[..segments.len()-1].join(".");
        Some(Namespace {
            hierarchy: parent_hierarchy,
            runtime: self.runtime.clone(),
        })
    }

    /// Create child namespace
    pub fn child(&self, name: &str) -> Result<Namespace, ComponentDescriptorError> {
        validate_namespace_segment(name)?;
        Ok(Namespace {
            hierarchy: format!("{}.{}", self.hierarchy, name),
            runtime: self.runtime.clone(),
        })
    }

    /// Get the root namespace segment
    pub fn root(&self) -> &str {
        self.hierarchy.split('.').next().unwrap_or(&self.hierarchy)
    }
}

impl Component {
    /// Get the parent namespace
    pub fn parent_namespace(&self) -> &Namespace {
        &self.namespace
    }
}

impl Endpoint {
    /// Get the parent component
    pub fn parent_component(&self) -> &Component {
        &self.component
    }
}
```

## Factory Methods

```rust
impl ComponentDescriptor {
    /// Create a new namespace descriptor
    pub fn new_namespace(namespace: &str) -> Result<Self, ComponentDescriptorError> {
        validate_namespace(namespace)?;
        Ok(ComponentDescriptor {
            namespace: namespace.to_string(),
            component: None,
            endpoint: None,
            lease_id: None,
        })
    }

    /// Create a new component descriptor
    pub fn new_component(namespace: &str, component: &str) -> Result<Self, ComponentDescriptorError> {
        validate_namespace(namespace)?;
        validate_component(component)?;
        Ok(ComponentDescriptor {
            namespace: namespace.to_string(),
            component: Some(component.to_string()),
            endpoint: None,
            lease_id: None,
        })
    }

    /// Create a new endpoint descriptor
    pub fn new_endpoint(
        namespace: &str,
        component: &str,
        endpoint: &str
    ) -> Result<Self, ComponentDescriptorError> {
        validate_namespace(namespace)?;
        validate_component(component)?;
        validate_endpoint(endpoint)?;
        Ok(ComponentDescriptor {
            namespace: namespace.to_string(),
            component: Some(component.to_string()),
            endpoint: Some(endpoint.to_string()),
            lease_id: None,
        })
    }

    /// Add lease ID to an existing descriptor
    pub fn with_lease_id(mut self, lease_id: i64) -> Self {
        self.lease_id = Some(lease_id);
        self
    }
}
```

## Validation System

```rust
impl ComponentDescriptor {
    /// Validate the entire descriptor
    pub fn validate(&self) -> Result<(), ComponentDescriptorError> {
        validate_namespace(&self.namespace)?;

        if let Some(component) = &self.component {
            validate_component(component)?;
        }

        if let Some(endpoint) = &self.endpoint {
            validate_endpoint(endpoint)?;
        }

        Ok(())
    }
}

impl Namespace {
    /// Validate namespace hierarchy format
    pub fn validate(&self) -> Result<(), ComponentDescriptorError> {
        validate_namespace(&self.hierarchy)
    }
}

impl Component {
    /// Validate component name
    pub fn validate(&self) -> Result<(), ComponentDescriptorError> {
        self.namespace.validate()?;
        validate_component(&self.name)
    }
}

impl Endpoint {
    /// Validate endpoint name and lease ID
    pub fn validate(&self) -> Result<(), ComponentDescriptorError> {
        self.component.validate()?;
        validate_endpoint(&self.name)
    }
}
```

## Keys Object for Key-Value Operations

The `ComponentDescriptor` focuses purely on identification. For key-value storage operations, a separate `Keys` object holds a descriptor along with key paths:

```rust
/// Holds a ComponentDescriptor with associated key paths for etcd operations
pub struct Keys {
    pub descriptor: ComponentDescriptor,
    pub key_paths: Vec<String>,
}

impl DiscoveryKey for Keys {
    /// Create/update a key-value pair
    fn create(&self, key: &str, value: Vec<u8>) -> Result<(), DiscoveryError> {
        // Implementation generates etcd paths from descriptor using URI format
        // e.g., {production.api.v1.gateway.http} -> dynamo://production.api.v1/_component_/gateway/_endpoint_/http/_keys_/key
    }

    fn put(&self, key: &str, value: Vec<u8>) -> Result<(), DiscoveryError> {
        // Similar etcd path generation for updates
    }

    fn get(&self, key: &str) -> Result<Option<Vec<u8>>, DiscoveryError> {
        // Generate etcd path and retrieve value
    }

    fn delete(&self, key: &str) -> Result<bool, DiscoveryError> {
        // Generate etcd path and delete key
    }
}
```

# Implementation Details

## Implementation Foundation

```rust
impl ComponentDescriptor {
    /// THE canonical identifier - used throughout the system
    /// Format: {namespace.component.endpoint:lease_id}
    /// Examples:
    /// - {production.api.v1}
    /// - {production.api.v1.gateway}
    /// - {production.api.v1.gateway.http}
    /// - {production.api.v1.gateway.http:1a2b3c4d}
    pub fn to_string(&self) -> String {
        // Provides the authoritative string representation
    }
}
```

## Usage Examples

```rust
use dynamo::discovery::{ComponentDescriptor, DistributedRuntime};

// Create descriptors at different levels
let namespace_desc = ComponentDescriptor::new_namespace("production.api.v1")?;
let component_desc = ComponentDescriptor::new_component("production.api.v1", "gateway")?;
let endpoint_desc = ComponentDescriptor::new_endpoint("production.api.v1", "gateway", "http")?;

// Get canonical identifiers - THE string representation used everywhere
println!("Namespace: {}", namespace_desc.to_string());  // {production.api.v1}
println!("Component: {}", component_desc.to_string());  // {production.api.v1.gateway}
println!("Endpoint: {}", endpoint_desc.to_string());    // {production.api.v1.gateway.http}

// Create typed entities with runtime context
let drt = DistributedRuntime::new();
let namespace = endpoint_desc.to_namespace(&drt);
let component = endpoint_desc.to_component(&drt).unwrap();
let endpoint = endpoint_desc.to_endpoint(&drt).unwrap();

// Navigate hierarchy - compound namespace produces child, use parent() to access parents
let parent_ns = namespace.parent();  // Returns Option<Namespace> for parent
let child_ns = namespace.child("v2")?;  // Creates production.api.v1.v2 namespace

// For key-value operations, use Keys object (separate from descriptor)
let keys = Keys {
    descriptor: endpoint_desc,
    key_paths: vec!["config".to_string(), "health".to_string()],
};

// DiscoveryKey operations generate etcd paths behind the scenes from the identifier
keys.put("config", b"server_config_data".to_vec())?;  // Uses dynamo://production.api.v1/_component_/gateway/_endpoint_/http/_keys_/config
keys.put("health", b"healthy".to_vec())?;             // Uses dynamo://production.api.v1/_component_/gateway/_endpoint_/http/_keys_/health
```

# Benefits

1. **Canonical Identification**: `ComponentDescriptor.to_string()` provides THE authoritative identifier used throughout the system
2. **Clean Separation**: Pure identification logic separate from etcd operations and key-value storage
3. **Typed Entities**: Create `Namespace`, `Component`, and `Endpoint` entities with runtime context
4. **Hierarchical Navigation**: Support parent/child relationships with compound namespace handling
5. **Validation Enforcement**: Comprehensive validation for all identifier components
6. **Behind-the-Scenes Operations**: etcd path generation hidden in DiscoveryKey/DiscoveryPlane implementations

# Alternate Solutions

## Alt 1: Generic Path-Based Naming

**Pros:**
- More generic, could apply to non-component use cases
- Familiar path-based naming convention

**Cons:**
- Name doesn't reflect component-centric purpose
- Generic naming doesn't convey discovery semantics

**Reason Rejected:**
- Component-focused naming better communicates intended usage

## Alt 2: Separate Types Without Relationship

**Pros:**
- Maximum type safety
- Clear separation of concerns

**Cons:**
- Difficult to convert between types
- Code duplication in validation

**Reason Rejected:**
- Loses hierarchical relationship benefits

# Background

Distributed systems require canonical identifiers for components that are consistent, validated, and provide clear hierarchical structure. The ComponentDescriptor establishes THE standard for component identification while keeping concerns clearly separated.

This approach follows established patterns from:
- Kubernetes resource naming (namespace/name structure)
- DNS hierarchical naming conventions
- Service mesh discovery patterns (Istio, Linkerd)
- Clean Architecture principles (separation of concerns)

The ComponentDescriptor provides the foundational identifier system with strong validation, while delegating etcd operations and key-value storage to specialized traits and objects.

## References

1. [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
2. [DNS Naming Conventions](https://tools.ietf.org/html/rfc1035)
3. [Service Discovery Patterns](https://microservices.io/patterns/service-registry.html)
