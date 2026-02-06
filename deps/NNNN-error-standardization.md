# Standardized Error System for Migration Detection and Network Propagation

**Status**:  Draft <!-- Draft | Under Review | Approved | Replaced | Deferred | Rejected -->

**Authors**: [@nnshah1](https://github.com/nnshah1) [@kthui](https://github.com/kthui) <!-- [Name/Team] -->

**Category**: Architecture <!-- Architecture | Process | Guidelines -->

**Replaces**: N/A <!-- [Link of previous proposal if applicable] -->

**Replaced By**: N/A <!-- [Link of previous proposal if applicable] -->

**Sponsor**: [@ryanolson](https://github.com/ryanolson) [@grahamking](https://github.com/grahamking) <!-- [Name of code owner or maintainer to shepard process] -->

**Required Reviewers**: [@grahamking](https://github.com/grahamking) [@PeaBrane](https://github.com/PeaBrane) <!-- [Names of technical leads that are required for acceptance] -->

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

Introduce a standardized error system for Dynamo that replaces ad-hoc string-based error detection with structured, chainable error types. The system provides a `DynamoError` trait with built-in support for error chaining, migration status propagation, and serialization across network boundaries. This enables reliable migration decisions, clear error messages in logs and user-facing responses, and consistent error handling across the entire Dynamo pipeline.

# Motivation

Dynamo's current error handling has several deficiencies that lead to poor migration detection and unclear error reporting:

**Unreliable migration detection.** The migration module (`migration.rs`) previously relied on matching a hardcoded string constant (`STREAM_ERR_MSG`) against error messages to decide whether a failed request should be retried on another worker. This approach is fragile: any upstream change to an error message, any wrapping of the error in a different format, or any new error condition that should also be migratable would silently break migration. In practice, migratable errors from new pipeline stages (e.g., disaggregated prefill) were not being detected, causing failures to be returned directly to the user instead of being retried.

**Loss of error context across the network.** Errors originating from remote engine workers are serialized through `Annotated` responses. Previously, errors were reduced to plain strings during this transmission, discarding the error type, cause chain, and any metadata. Once an error crossed a network boundary, the frontend could no longer determine what kind of error occurred or whether it was safe to retry.

**Unclear error messages.** Without structured error types, log messages and user-facing error responses lacked consistency. It was difficult to tell from a log line what error type occurred, where it originated (frontend vs. remote engine), or what chain of sub-errors led to the failure.

## Goals

* Enable reliable, flag-based migration decisions that do not depend on string matching.

* Preserve the full error chain, including error type names and migration flags, across network serialization boundaries.

* Provide clear, structured error messages for both log output and user-facing responses, with the format `ErrorTypeName: message; Caused by: InnerErrorTypeName: inner_message`.

* Make it easy for developers across different Dynamo modules to define custom error types with correct migration semantics, without needing to modify the migration module.

### Non Goals

* Make error passing more performant. The focus is on correctness and clarity, not on reducing serialization overhead.

* Make errors trivially easy to create for developers. The system standardizes error creation to produce clear, interoperable types that other modules can understand. While the `define_dynamo_error!` macro reduces boilerplate, this proposal does not aim to make the act of sending errors simpler, only to ensure that error information is not lost when errors are sent.

## Requirements

### REQ 1 Migration Flag Propagation

Every error type in the Dynamo pipeline **MUST** carry an intrinsic migration status (`migratable`, `not_migratable`, or `inherit`). The migration module **MUST** be able to resolve the effective migration status by walking the error chain without relying on error type names or message content.

### REQ 2 Network Serialization

Errors **MUST** be serializable and deserializable for transmission across network boundaries via `Annotated` responses. The serialized form **MUST** preserve the error name, message, migration status, and the full cause chain so that the receiving end can reconstruct the error with complete information.

### REQ 3 Error Chaining

Errors **MUST** support chaining, where an outer error wraps an inner error as its cause. The `Display` format **MUST** render the full chain: `OuterError: outer message; Caused by: InnerError: inner message`.

### REQ 4 Custom Error Types

Developers **MUST** be able to define custom error types in their own modules that participate in the Dynamo error system. Custom types **MUST** implement the `DynamoError` trait and **SHOULD** use the `define_dynamo_error!` macro for consistency.

### REQ 5 Result-Level and Stream-Level Error Detection

The migration module **MUST** detect migratable errors at both the stream level (errors inside `Annotated` responses) and the `Result` level (errors returned from pipeline `generate()` calls). Both paths **MUST** consult the `DynamoError` migration flag rather than relying on error type matching or string content.

# Proposal

## Overview

The proposal introduces three core components:

1. **`DynamoError` trait** (`lib/runtime/src/error.rs`) - A trait extending `std::error::Error + Send + Sync + 'static` that adds error naming, chaining, and migration status to all Dynamo errors.

2. **`AnyError` struct** (`lib/runtime/src/error.rs`) - A concrete, serializable implementation of `DynamoError` that serves as the universal wire format for transmitting errors across network boundaries and as a fallback error type for modules that have not yet defined custom errors.

3. **`define_dynamo_error!` macro** (`lib/runtime/src/error.rs`) - An exported macro that generates boilerplate for custom error types, ensuring they correctly implement `DynamoError` with a `NAME` constant derived from the type name and a descriptive migration keyword.

These components integrate with the existing `Annotated` and `MaybeError` protocols to enable end-to-end structured error handling.

## DynamoError Trait

The `DynamoError` trait is the foundation of the error system:

```rust
pub trait DynamoError: std::error::Error + Send + Sync + 'static {
    /// The error type name, automatically matching the struct name.
    fn error_name(&self) -> &str;

    /// The detailed error message.
    fn error_message(&self) -> &str;

    /// The inner cause of this error, if any.
    fn caused_by(&self) -> Option<&dyn DynamoError> { None }

    /// Intrinsic migration status of this error type alone.
    /// - Some(true): migratable
    /// - Some(false): not migratable
    /// - None: inherit from cause
    fn is_error_migratable(&self) -> Option<bool> { None }

    /// Resolved migration status for the entire error chain.
    /// Default implementation walks the chain - no need to override.
    fn is_chain_migratable(&self) -> bool { /* ... */ }
}
```

The `is_chain_migratable()` method provides a default implementation that resolves migration status by walking the error chain. Developers only need to set `is_error_migratable()` on their error type; the chain resolution is handled automatically.

## Migration Status Resolution

Each error type declares one of three migration statuses:

- **`migratable`** (`Some(true)`): Transient failures where retrying on another worker is expected to help (e.g., stream disconnected, engine dead).
- **`not_migratable`** (`Some(false)`): Permanent failures or intentional terminations where retrying would not help (e.g., cancelled by user, validation error).
- **`inherit`** (`None`): Wrapper errors that delegate the decision to their inner cause (e.g., a prefill execution error wrapping a stream disconnection).

The `is_chain_migratable()` default implementation resolves these through the chain:

| This Error | Cause Present? | Cause Migratable? | Result |
|:-----------|:---------------|:-------------------|:-------|
| `migratable` | No | N/A | `true` |
| `migratable` | Yes | `true` | `true` |
| `migratable` | Yes | `false` | `false` (inner overrides) |
| `not_migratable` | Any | Any | `false` (hard stop) |
| `inherit` | No | N/A | `false` (conservative default) |
| `inherit` | Yes | `true` | `true` (delegates to cause) |
| `inherit` | Yes | `false` | `false` (delegates to cause) |

This design means that a `not_migratable` error anywhere in the chain stops migration, a `migratable` error without a conflicting inner error allows migration, and an `inherit` error transparently passes the decision through to its cause.

## AnyError

`AnyError` is the serializable implementation of `DynamoError`:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct AnyError {
    pub name: String,
    pub message: String,
    pub migratable: Option<bool>,
    pub cause: Option<Box<AnyError>>,
}
```

It serves two purposes:

1. **Network wire format.** When an error needs to cross a network boundary (via `Annotated`), the sender converts any `DynamoError` into an `AnyError` using `From<&dyn DynamoError>`. The entire cause chain is recursively converted. The receiver deserializes it back into an `AnyError` and can inspect the error name, message, and migration status.

2. **Fallback error type.** Modules that have not yet defined custom error types can use `AnyError::msg("message")` to create a simple error with `inherit` migration status. This provides a migration path from unstructured `anyhow::Error` usage.

Conversion from `DynamoError` preserves all fields:
```rust
let stream_err = StreamDisconnectedError::new("connection lost");
let any_err = AnyError::from(&stream_err as &dyn DynamoError);
// any_err.name == "StreamDisconnectedError"
// any_err.migratable == Some(true)
```

Conversion from `std::error::Error` preserves the cause chain but uses "AnyError" as the name with `inherit` migration:
```rust
let io_err = std::io::Error::new(std::io::ErrorKind::Other, "disk full");
let any_err: AnyError = (&io_err as &(dyn std::error::Error + 'static)).into();
```

## Integration with Annotated and MaybeError

The `Annotated<R>` struct carries an optional `AnyError` field:

```rust
pub struct Annotated<R> {
    pub data: Option<R>,
    pub id: Option<String>,
    pub event: Option<String>,
    pub comment: Option<Vec<String>>,
    pub error: Option<AnyError>,
}
```

The `MaybeError` trait provides the interface for types that may contain errors:

```rust
pub trait MaybeError {
    fn from_err(err: &dyn DynamoError) -> Self;
    fn err(&self) -> Option<AnyError>;
    fn is_ok(&self) -> bool { !self.is_err() }
    fn is_err(&self) -> bool { self.err().is_some() }
}
```

When a pipeline stage produces an error, it calls `Annotated::from_err(&err)` which serializes the `DynamoError` into an `AnyError` and stores it in the `error` field. Downstream consumers call `.err()` to retrieve the `AnyError` and `.is_chain_migratable()` to check migration status.

## define_dynamo_error! Macro

The macro generates a complete error type with a single invocation:

```rust
define_dynamo_error!(
    /// Error indicating a stream was disconnected before completion.
    StreamDisconnectedError,
    migratable
);
```

This generates:
- A `pub struct StreamDisconnectedError` with `message` and `cause` fields
- A `NAME` constant automatically set to `"StreamDisconnectedError"` via `stringify!`
- `new(message)` and `with_cause(cause)` constructors
- `Display`, `Error`, and `DynamoError` trait implementations
- `source()` delegating to `caused_by()` for consistency

The macro accepts three migration keywords: `migratable`, `not_migratable`, and `inherit`. Custom error types are defined in their own modules close to where they are used, not centrally in `error.rs`.

## Migration Module Integration

The migration module (`migration.rs`) checks for migratable errors at two levels:

**Stream-level errors** (existing path): When processing responses from the stream, the `RetryManager` checks each `Annotated` response:
```rust
let is_migratable = response.err().map(|e| e.is_chain_migratable()).unwrap_or(false);
```

**Result-level errors** (new path): When `generate()` returns an `Err`, the `RetryManager` now also checks if the error wraps a `DynamoError` with a migratable chain. Operators like `PrefillRouter` wrap stream errors in typed `DynamoError` values (e.g., `PrefillExecutionError` with `inherit` status and a `StreamDisconnectedError` cause). These are stored in `anyhow::Error` via `Box<dyn DynamoError>`, and the migration module recovers them via `downcast_ref`:
```rust
if let Some(dynamo_err) = err.downcast_ref::<Box<dyn DynamoError>>()
    && dynamo_err.is_chain_migratable()
{
    // retry
}
```

# Implementation Details

## Custom Error Type Example

A module that introduces new error conditions defines its types locally:

```rust
use dynamo_runtime::{define_dynamo_error, error::DynamoError};

define_dynamo_error!(
    /// Prefill router has not been activated yet.
    PrefillNotActivatedError,
    not_migratable
);

define_dynamo_error!(
    /// Error during prefill execution. Inherits migration from cause.
    PrefillExecutionError,
    inherit
);
```

When a stream error occurs during prefill, the cause chain preserves migration semantics:
```rust
if let Some(err) = first_output.err() {
    return Err(Box::new(
        PrefillExecutionError::new("Prefill router returned error in output")
            .with_cause(err),  // err is AnyError with StreamDisconnectedError cause
    ) as Box<dyn DynamoError>);
}
```

The `PrefillExecutionError` has `inherit` status. Its cause is a `StreamDisconnectedError` with `migratable` status. Walking the chain, `is_chain_migratable()` resolves to `true`, and the migration module retries the request.

## Box<dyn DynamoError> as Internal Return Type

Internal functions that produce errors from multiple error types return `Box<dyn DynamoError>` to preserve the trait interface:

```rust
async fn execute_prefill(...) -> std::result::Result<(...), Box<dyn DynamoError>> {
    let router = router.ok_or_else(|| -> Box<dyn DynamoError> {
        Box::new(PrefillNotActivatedError::new("Prefill router not yet activated"))
    })?;
    // ...
}
```

At the `Operator::generate()` boundary, where the trait requires `anyhow::Result`, the error is wrapped:
```rust
return Err(anyhow::anyhow!(e));  // e: Box<dyn DynamoError>, stored as-is in anyhow
```

The migration module recovers it via `err.downcast_ref::<Box<dyn DynamoError>>()`.

## Deferred to Implementation

* Extending the pattern to Python bindings and other language interfaces.
* Adding structured error reporting to HTTP/gRPC response bodies (currently errors are returned as plain text messages derived from `Display`).

# Implementation Phases

## Phase 0: Core Error System

**Release Target**: Initial

**Effort Estimate**: 1 engineer, 1 week

**Supported API / Behavior:**

* `DynamoError` trait with `error_name()`, `error_message()`, `caused_by()`, `is_error_migratable()`, `is_chain_migratable()`
* `AnyError` struct with serialization, `From<&dyn DynamoError>`, `From<&dyn std::error::Error>`
* `define_dynamo_error!` macro with `migratable`, `not_migratable`, `inherit` keywords
* `Annotated` and `MaybeError` updated to use `AnyError`
* `StreamDisconnectedError` defined in `pipeline::network`
* Migration module updated to use `is_chain_migratable()` at both stream and Result levels

**Not Supported:**

* Custom error types in modules beyond `pipeline::network` (deferred to Phase 1)

## Phase 1: Module-Specific Error Types

**Release Target**: Subsequent

**Effort Estimate**: 1-2 engineers, 1-2 weeks

**Supported API / Behavior:**

* Prefill router error types (`PrefillNotActivatedError`, `PrefillExecutionError`, `PrefillNoDisaggParamsError`)
* Additional error types across other pipeline stages as modules are updated
* Gradual replacement of `anyhow::Error` with typed errors in key pipeline paths

**Not Supported:**

* Full replacement of all `anyhow::Error` usage across the codebase (incremental)

# Related Proposals

* [DEP 0005 - LLM Request Migration on Engine Failure](/enhancements/deps/0005-llm-request-migration.md)

* [DEP 0001 - Streaming Responses and Error Handling](/enhancements/deps/0001-stream-res-err.md)

# Alternate Solutions

## Alt 1: Use std::error::Error Directly

**Pros:**

* No new trait to learn; uses Rust's standard error trait.
* Broad ecosystem compatibility.

**Cons:**

* `std::error::Error` has no mechanism for custom flags like migration status. There is no way to attach a `migratable` flag to a standard error and have it propagate through a chain.
* `std::error::Error` is not serializable. `source()` returns `&dyn Error` which cannot be serialized for network transmission. Similar bridging work to `AnyError` would be needed regardless.
* No standardized error naming. `Display` output varies by implementation, making it impossible to programmatically identify error types across network boundaries.

**Reason Rejected:**

* Cannot carry migration flags. The migration module would still need a separate mechanism to determine whether an error is migratable, which defeats the purpose of standardization.
* Cannot be serialized over the network without a custom wrapper similar to `AnyError`, so the implementation effort is comparable while providing fewer guarantees.

## Alt 2: Use anyhow::Error

**Pros:**

* Already widely used in the Dynamo codebase.
* Easy to create: `anyhow::anyhow!("message")`.
* Supports downcasting to concrete types.

**Cons:**

* `anyhow::Error` erases the trait surface. Once an error is wrapped in `anyhow`, you can no longer call trait methods like `is_chain_migratable()` without downcasting to a specific concrete type.
* Cannot carry custom flags like migration status in a standardized way.
* Not serializable for network transmission.
* Downcasting requires knowing the exact concrete type, which creates tight coupling between modules. The migration module would need to enumerate and check every possible error type rather than relying on a common trait method.

**Reason Rejected:**

* Erases the `DynamoError` trait surface, making it impossible to generically check migration status without enumerating concrete types. This was demonstrated in the PoC where `anyhow::anyhow!(e)` on a `Box<dyn DynamoError>` required `downcast_ref::<Box<dyn DynamoError>>()` to recover the trait, adding complexity.
* Not serializable, so the same bridging work is needed for network transmission.

## Alt 3: Use thiserror Enums

**Pros:**

* Generates `std::error::Error` implementations with less boilerplate.
* Supports `#[from]` for automatic wrapping.
* Widely used in the Rust ecosystem.

**Cons:**

* Migration status can be embedded via custom fields, but there is no standardized way to access it. Each module could define migration flags slightly differently (e.g., a `bool` field vs. a method vs. an enum variant), making it impossible for the migration module to inspect migration status through a common interface.
* Enum-based errors become a central dependency. Adding a new variant requires modifying the enum definition, which may live in a different module. This forces developers to constantly coordinate with the module that owns the enum.
* Not serializable out of the box for network transmission.

**Reason Rejected:**

* Lacks a standardized interface for migration flags. Without a common trait, the migration module would need to pattern-match on specific enum variants or error type names, reintroducing the fragile string/type matching that this proposal eliminates.
* The `DynamoError` trait provides the standardized interface that `thiserror` cannot, while the `define_dynamo_error!` macro provides comparable boilerplate reduction.

## Alt 4: Migration Flag via Error Type Registration

An alternative to embedding the migration flag in each error type would be to maintain a registry of error type names in the migration module. Each error type name would be mapped to its migration status. The `DynamoError` trait would still be needed for error naming and serialization, but the migration decision would be centralized.

**Pros:**

* Migration decisions are centralized in one place.
* Error types do not need to carry a migration flag, saving a small amount of memory per error.

**Cons:**

* Error type names must be globally unique and transmitted over the network for the registry to match them. The `DynamoError` trait (or equivalent) is still needed to provide the error name, so the implementation effort is comparable.
* Name collisions are possible when different modules independently define error types. Two modules could define a `TimeoutError` with different migration semantics, and the registry would have no way to distinguish them.
* Developers must register their error types with the migration module, creating cross-module coordination overhead. A developer working on the prefill module would need to go to the migration module to register `PrefillExecutionError`, breaking module isolation.
* Adding a new error type requires a change in two places (the module defining the error and the migration registry), increasing the risk of the registry becoming stale.

**Reason Rejected:**

* Does not reduce implementation complexity since error naming and serialization are still required.
* Introduces cross-module coupling and coordination overhead that the per-type flag avoids.
* The per-type flag approach keeps the migration semantics co-located with the error definition, which is where the developer has the most context about whether an error is transient or permanent. The trade-off of a small amount of memory per error is negligible compared to the developer experience benefit.

# Background

## References

* [Rust std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html)
* [anyhow crate](https://docs.rs/anyhow)
* [thiserror crate](https://docs.rs/thiserror)
* [RFC 2119 - Key words for use in RFCs](https://datatracker.ietf.org/doc/html/rfc2119)

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **DynamoError** | The core error trait for the Dynamo error system, extending `std::error::Error + Send + Sync + 'static` with error naming, chaining, and migration status. |
| **AnyError** | A concrete, serializable implementation of `DynamoError` used as the wire format for network transmission and as a fallback for untyped errors. |
| **Migration** | The process of retrying a failed LLM request on a different engine worker. |
| **Migration status** | A per-error-type flag indicating whether a failed request should be retried: `migratable`, `not_migratable`, or `inherit`. |
| **Error chain** | A linked sequence of errors where each error may have a `caused_by` inner error, forming a chain from the outermost wrapper to the root cause. |
| **Stream-level error** | An error carried inside an `Annotated` response as an `AnyError`, detected by the migration module via `MaybeError::err()`. |
| **Result-level error** | An error returned as `Err(anyhow::Error)` from a pipeline `generate()` call, detected by the migration module via `downcast_ref`. |
