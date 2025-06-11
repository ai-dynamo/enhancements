# Instance Model for Structured Etcd Paths

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

Introduce `Instance` as the foundational identifier system for distributed components, providing THE canonical string representation for namespaces, components, and endpoints through the `Identity` trait. This establishes a robust foundation with strong validation rules and type safety, while separating identifier concerns from etcd path generation and key-value operations.

# Motivation

Dynamo requires a structured approach to component identification that enforces hierarchical naming conventions and provides canonical string identifiers for distributed system operations.

## Requirements for Instance

1. **Canonical Identifiers**: Provide THE authoritative string representation for components via `to_string()`
2. **Typed Entity Creation**: Create `Namespace`, `Component`, and `Endpoint` entities from descriptors
3. **Hierarchical Navigation**: Support parent/child relationships in namespace hierarchies
4. **Validation Enforcement**: Implement robust validation rules for all identifier components
5. **Separation of Concerns**: Focus purely on identification, not etcd path generation or key-value operations

## Goals

- **Canonical Identification**: `Instance` provides THE standard string identifier for all components
- **Entity Type System**: Create typed `Namespace`, `Component`, and `Endpoint` entities from descriptors
- **Hierarchical Navigation**: Support parent/child traversal in namespace hierarchies
- **Strong Validation**: Implement comprehensive validation for component identifiers
- **Clean Separation**: Identifier logic separate from etcd operations and key-value storage
- **Uniform Interface**: All types implement `Identity` trait for consistent access to canonical strings

# Current Implementation

The current implementation includes @Instance, @Component, @Endpoint, and @Namespace entities defined in `dynamo/lib/runtime/src/component.rs`.

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Instance {
    pub component: String,
    pub endpoint: String,
    pub namespace: String,
    pub instance_id: i64,
    pub transport: TransportType,
}
```

There exists an implementation for Instance but it is not widely used through the codebase and was created out of necessity. This proposal makes `Instance` a core piece that manages the validation and identification of components for ETCD.

We also have various ad hoc methods by which to create paths or slugify. This was fine when we had a fixed set of namespace, component, endpoint. However, now that we will allow multiple namespaces and will start to have components communicate by reading and writing to etcd (leader/worker barriers) we want to nail down the etcd contract.

We have various functions like these scattered through the codebase.
```rust
    pub fn etcd_root(&self) -> String {
        let ns = self.namespace.name();
        let cp = &self.name;
        format!("{INSTANCE_ROOT_PATH}/{ns}/{cp}")
    }

    pub fn path(&self) -> String {
        format!("{}/{}", self.namespace.name(), self.name)
    }

    pub fn service_name(&self) -> String {
        let service_name = format!("{}_{}", self.namespace.name(), self.name);
        Slug::slugify(&service_name).to_string()
    }

    pub fn etcd_path(&self, lease_id: i64) -> String {
        let endpoint_root = self.etcd_root();
        if self.is_static {
            endpoint_root
        } else {
            format!("{endpoint_root}:{lease_id:x}")
        }
    }

    pub fn name_with_id(&self, lease_id: i64) -> String {
        if self.is_static {
            self.name.clone()
        } else {
            format!("{}-{:x}", self.name, lease_id)
        }
    }

    pub fn subject_to(&self, lease_id: i64) -> String {
        format!(
            "{}.{}",
            self.component.service_name(),
            self.name_with_id(lease_id)
        )
    }
```

We plan to split identification and typed entity creation.

# Proposal

## Core Type Hierarchy

### 1. Instance Foundation

```rust
/// Provides THE canonical identifier for distributed components
/// Focuses purely on identification and typed entity creation
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Instance {
    pub namespace: String,
    pub component: Option<String>,
    pub endpoint: Option<String>,
    pub lease_id: Option<i64>,
}
```

### 2. Identity Trait

```rust
/// Trait for types that can produce a canonical string identifier
pub trait Identity {
    /// Get the canonical string representation
    fn to_string(&self) -> String;
}
```

### 3. Typed Entity Structs

```rust
/// Represents a validated namespace hierarchy with runtime context
#[derive(Debug, Clone)]
pub struct Namespace {
    hierarchy: String,
    runtime: DistributedRuntime,
}

/// Represents a component within a namespace
#[derive(Debug, Clone)]
pub struct Component {
    namespace: Namespace,
    name: String,
}

/// Represents an endpoint within a component
#[derive(Debug, Clone)]
pub struct Endpoint {
    component: Component,
    name: String,
    lease_id: Option<i64>,
}

impl Identity for Namespace {
    fn to_string(&self) -> String {
        // Create temporary Instance and use it as single source of truth
        let instance = Instance {
            namespace: self.hierarchy.clone(),
            component: None,
            endpoint: None,
            lease_id: None,
        };
        instance.to_string()
    }
}

impl Identity for Component {
    fn to_string(&self) -> String {
        // Create temporary Instance and use it as single source of truth
        let instance = Instance {
            namespace: self.namespace.hierarchy.clone(),
            component: Some(self.name.clone()),
            endpoint: None,
            lease_id: None,
        };
        instance.to_string()
    }
}

impl Identity for Endpoint {
    fn to_string(&self) -> String {
        // Create temporary Instance and use it as single source of truth
        let instance = Instance {
            namespace: self.component.namespace.hierarchy.clone(),
            component: Some(self.component.name.clone()),
            endpoint: Some(self.name.clone()),
            lease_id: self.lease_id,
        };
        instance.to_string()
    }
}
```

### 4. Entity Creation and Identity

```rust
impl Identity for Instance {
    /// Provides THE canonical string identifier for this component
    /// This is the authoritative representation used throughout the system
    ///
    /// IMPORTANT: This is the SINGLE SOURCE OF TRUTH for canonical string format.
    /// All other Identity implementations MUST delegate to this method.
    fn to_string(&self) -> String {
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
}

impl Instance {
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
    pub fn child(&self, name: &str) -> Result<Namespace, InstanceError> {
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

## 5. Constructor Methods

```rust
impl Instance {
    /// Create a new namespace descriptor
    pub fn new_namespace(namespace: &str) -> Result<Self, InstanceError> {
        validate_namespace(namespace)?;
        Ok(Instance {
            namespace: namespace.to_string(),
            component: None,
            endpoint: None,
            lease_id: None,
        })
    }

    /// Create a new component descriptor
    pub fn new_component(namespace: &str, component: &str) -> Result<Self, InstanceError> {
        validate_namespace(namespace)?;
        validate_component(component)?;
        Ok(Instance {
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
    ) -> Result<Self, InstanceError> {
        validate_namespace(namespace)?;
        validate_component(component)?;
        validate_endpoint(endpoint)?;
        Ok(Instance {
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

## 6. Validation System

```rust
impl Instance {
    /// Validate the entire descriptor
    pub fn validate(&self) -> Result<(), InstanceError> {
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
```

## 7. Usage Examples

```rust
use dynamo::discovery::{Instance, DistributedRuntime, Identity};

// Create descriptors at different levels
let namespace_desc = Instance::new_namespace("production.api.v1")?;
let component_desc = Instance::new_component("production.api.v1", "gateway")?;
let endpoint_desc = Instance::new_endpoint("production.api.v1", "gateway", "http")?;

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

// All types implement Identity trait - same interface everywhere
println!("From namespace: {}", namespace.to_string());  // {production.api.v1}
println!("From component: {}", component.to_string());  // {production.api.v1.gateway}
println!("From endpoint: {}", endpoint.to_string());    // {production.api.v1.gateway.http}
```

# 8. Benefits

1. **Canonical Identification**: `Instance.to_string()` provides THE authoritative identifier used throughout the system
2. **Clean Separation**: Pure identification logic separate from etcd operations and key-value storage
3. **Typed Entities**: Create `Namespace`, `Component`, and `Endpoint` entities with runtime context
4. **Hierarchical Navigation**: Support parent/child relationships with compound namespace handling
5. **Validation Enforcement**: Comprehensive validation for all identifier components
6. **Consistent Interface**: All types implement `Identity` trait, providing uniform access to canonical strings
7. **Single Source of Truth**: All canonical string generation delegates to `Instance.to_string()`, ensuring consistency

# 9. Alternate Solutions

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

Distributed systems require canonical identifiers for components that are consistent, validated, and provide clear hierarchical structure. The Instance establishes THE standard for component identification while keeping concerns clearly separated.

This approach follows established patterns from:
- Kubernetes resource naming (namespace/name structure)
- DNS hierarchical naming conventions
- Service mesh discovery patterns (Istio, Linkerd)
- Clean Architecture principles (separation of concerns)

The Instance provides the foundational identifier system with strong validation, while delegating etcd operations and key-value storage to specialized traits and objects. See [discovery-trait-system](enhancements/0000-discovery-trait-system.md)

## References

1. [Kubernetes Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)
2. [DNS Naming Conventions](https://tools.ietf.org/html/rfc1035)
3. [Service Discovery Patterns](https://microservices.io/patterns/service-registry.html)
