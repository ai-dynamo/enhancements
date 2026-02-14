# Descriptor and Entity Model for Distributed Component Management

**Status**: Draft

**Authors**: [ryanolson](https://github.com/ryanolson), [aflowers](https://github.com/alec-flowers)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [nnshah1](https://github.com/nnshah1)

**Required Reviewers**: [dzier](https://github.com/dzier), [statiraju](https://github.com/statiraju), [kkranen](https://github.com/kkranen)

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Introduce a two-tier architecture separating **descriptors** (`Identifier`, `Instance`, `Keys`) for pure data representation from **entities** (`Namespace`, `Component`, `Endpoint`, `Path`) that provide distributed runtime operations. Descriptors handle validation, string parsing, and canonical path representation, while entities enable actual etcd operations through embedded DistributedRuntime handles.

# Motivation

Dynamo requires a structured approach to component identification that separates data representation from runtime operations, enforcing hierarchical naming conventions while providing clean architectural boundaries.

## Current Problems

1. **Mixed Concerns**: Data structures contain both representation and runtime dependencies
2. **Ad Hoc Path Generation**: Various path creation methods scattered throughout codebase
3. **No Clear Separation**: Descriptors and entities are conflated into single types
4. **Inconsistent Validation**: Different validation logic in different places
5. **Limited Extensibility**: No systematic way to handle reserved keywords or path extensions

## Goals

- **Clean Architecture**: Separate pure data representation from operational concerns
- **Type Safety**: Distinct types for different abstraction levels
- **Canonical Format**: Standardize on `dynamo://` path representation
- **Validation Consistency**: All paths through same validation logic
- **Extension System**: Support reserved keyword extensions for special paths

# Proposal

## Architecture Overview

1. **Descriptors** (`dynamo::runtime::descriptor`): Pure data types with no runtime dependencies
   - Own canonical string format (`dynamo://` paths)
   - Self-contained parsing and validation
   - Immutable data representation

2. **Entities** (`dynamo::runtime::entity`): Operational types with DistributedRuntime
   - Created from descriptors via factory pattern
   - Enable etcd operations
   - Business logic implementation

## Core Types

### Constants

```rust
pub const ETCD_ROOT_PATH: &str = "dynamo://";
pub const COMPONENT_KEYWORD: &str = "_component_";
pub const ENDPOINT_KEYWORD: &str = "_endpoint_";
pub const PATH_KEYWORD: &str = "_path_";
pub const BARRIER_KEYWORD: &str = "_barrier_";
```

### Unified Parser

```rust
/// Intermediate representation of a parsed dynamo path
struct DynamoPath {
    namespace: String,
    component: Option<String>,
    endpoint: Option<String>,
    instance_id: Option<i64>,
    extra_segments: Vec<String>,
}

impl DynamoPath {
    /// Parse any dynamo:// path into intermediate representation
    fn parse(input: &str) -> Result<Self, DescriptorError> {
        // Single parsing implementation used by all descriptors
        // Handles all path formats and extracts components
    }

    /// Convert to specific descriptor types with lenient parsing
    fn try_into_identifier(self) -> Result<Identifier, DescriptorError> {
        // Drops instance_id and extra_segments if present
    }

    fn try_into_instance(self) -> Result<Instance, DescriptorError> {
        // Requires instance_id, drops extra_segments
    }

    fn try_into_keys(self) -> Result<Keys, DescriptorError> {
        // Preserves all information
    }
}
```

### Descriptor Types

```rust
/// Base descriptor for component identification
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Identifier {
    namespace: String,
    component: Option<String>,
    endpoint: Option<String>,
}

impl Identifier {
    pub fn new_namespace(namespace: &str) -> Result<Self, DescriptorError>;
    pub fn new_component(namespace: &str, component: &str) -> Result<Self, DescriptorError>;
    pub fn new_endpoint(namespace: &str, component: &str, endpoint: &str) -> Result<Self, DescriptorError>;

    pub fn namespace(&self) -> &str;
    pub fn component(&self) -> Option<&str>;
    pub fn endpoint(&self) -> Option<&str>;
}

impl TryFrom<&str> for Identifier {
    fn try_from(input: &str) -> Result<Self, Self::Error> {
        DynamoPath::parse(input)?.try_into_identifier()
    }
}

/// Identifier with instance ID (immutable once created)
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Instance {
    identifier: Identifier,  // Private to enforce immutability
    instance_id: i64,
}

impl Instance {
    pub fn new(identifier: Identifier, instance_id: i64) -> Result<Self, DescriptorError>;
    pub fn identifier(&self) -> &Identifier;
    pub fn instance_id(&self) -> i64;
}

/// Extended path descriptor supporting reserved keywords
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Keys {
    base: KeysBase,
    keys: Vec<String>,
}

pub enum KeysBase {
    Identifier(Identifier),
    Instance(Instance),
}

impl Keys {
    pub fn from_identifier(identifier: Identifier, keys: Vec<String>) -> Result<Self, DescriptorError>;
    pub fn from_instance(instance: Instance, keys: Vec<String>) -> Result<Self, DescriptorError>;
    pub fn base(&self) -> &KeysBase;
    pub fn keys(&self) -> &[String];
}
```

### Entity Types (Operational)

```rust
/// All entities embed DistributedRuntime for operations
pub struct Namespace {
    descriptor: Identifier,
    runtime: DistributedRuntime,
}

pub struct Component {
    descriptor: Identifier,
    runtime: DistributedRuntime,
}

pub struct Endpoint {
    descriptor: Identifier,
    instance_id: Option<i64>,
    runtime: DistributedRuntime,
}

pub struct Path {
    keys: Keys,
    runtime: DistributedRuntime,
}

/// Factory trait for descriptor to entity conversion
pub trait ToEntity {
    type Entity;
    fn to_entity(self, runtime: DistributedRuntime) -> Result<Self::Entity, Error>;
}
```

## Reserved Keywords System

The system supports reserved keywords for extended functionality:

- **`_component_`**, **`_endpoint_`**: Internal path markers (parser use only)
- **`_barrier_`**: Distributed synchronization points
- **`_path_`**: Arbitrary path extensions for component data

### Usage Examples

```rust
// Standard paths
let comp = Identifier::new_component("prod", "gateway")?;
// dynamo://prod/_component_/gateway

// Instance with ID
let inst = Instance::new(endpoint_id, 0x1234)?;
// dynamo://prod/_component_/gateway/_endpoint_/http:1234

// Extended paths with reserved keywords
let config = Keys::from_identifier(
    comp,
    vec![PATH_KEYWORD.to_string(), "config".to_string()]
)?;
// dynamo://prod/_component_/gateway/_path_/config

// Lenient parsing - parse complex path as simpler type
let path = "dynamo://ns/_component_/comp/_endpoint_/ep:1234/extra/data";
let id: Identifier = path.try_into()?;    // Drops :1234 and /extra/data
let inst: Instance = path.try_into()?;    // Drops /extra/data
let keys: Keys = path.try_into()?;        // Keeps everything
```

## Validation

```rust
static ALLOWED_CHARS_REGEX: Lazy<Regex> =
    Lazy::new(|| Regex::new(r"^[a-z0-9-_]+$").unwrap());

fn validate_namespace(namespace: &str) -> Result<(), DescriptorError> {
    // Validates dot-separated segments
}

fn validate_component(component: &str) -> Result<(), DescriptorError> {
    // Validates single segment
}

fn validate_endpoint(endpoint: &str) -> Result<(), DescriptorError> {
    // Validates single segment
}

fn validate_path_segment(segment: &str) -> Result<(), DescriptorError> {
    // Allows reserved keywords and valid segments
}
```

## Benefits

1. **Single Source of Truth**: Descriptors own the canonical `dynamo://` format
2. **Clean Separation**: Pure data descriptors vs operational entities
3. **Unified Parser**: Single parsing implementation eliminates duplication
4. **Lenient Parsing**: Flexible conversion between types
5. **Type Safety**: Compile-time guarantees for valid paths
6. **Extensibility**: Reserved keyword system for future features

## Migration Path

The existing `EtcdPath` implementation will be replaced by descriptors:
- `Identifier` replaces `EtcdPath` for basic paths
- `Instance` replaces `EtcdPath` with `lease_id`
- `Keys` replaces `EtcdPath` with `extra_path`

This ensures backward compatibility while providing a cleaner architecture.

