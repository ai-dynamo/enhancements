# Standardized Error System for Error Clarity, Migration Detection, and Network Propagation

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

Introduce a standardized error struct for Dynamo with a fixed set of error categories defined via an `ErrorType` enum. The `DynamoError` struct is serializable, chainable via `std::error::Error::source()`, and transmittable across network boundaries through `Annotated` responses. Error categorization enables clear error messages for end users and logs, and allows consumers like the migration module to make reliable decisions based on error type.

# Motivation

Dynamo's current error handling has three deficiencies: poor migration detection, loss of error context across the network, and unclear error messages.

**Unreliable migration detection.** The migration module previously relied on matching a hardcoded string constant (`STREAM_ERR_MSG`) against error messages to decide whether a failed request should be retried on another worker. This approach is fragile: any change to an error message, any wrapping of the error in a different format, or any new error condition that should also be migratable would silently break migration. In practice, migratable errors from new pipeline stages (e.g., disaggregated prefill) were not being detected, causing failures to be returned directly to the user instead of being retried.

**Loss of error context across the network.** Errors originating from remote engine workers are serialized through `Annotated` responses. Previously, errors were reduced to plain strings during this transmission, discarding the error type, cause chain, and any metadata. Once an error crossed a network boundary, the frontend could no longer determine what kind of error occurred or whether it was safe to retry.

**Unclear error messages.** Without structured error types, log messages and user-facing error responses lacked consistency. A log line might show an opaque message like `"Stream ended before generation completed"` without indicating whether the error came from a remote engine or the local frontend, whether it was a connection issue or a logic error, or what chain of sub-errors led to the failure. End users received similarly unhelpful error messages that offered no indication of the error category.

## Goals

* Provide clear, categorized error types that are immediately understandable in logs and user-facing responses. The outermost error type (e.g., `Disconnected`, `CannotConnect`) **SHOULD** tell the reader what happened at a glance, while the full cause chain is available for debugging.

* Enable reliable migration decisions based on error type categories, not string matching.

* Preserve the full error chain — including error types and messages — across network serialization boundaries, so that errors from remote workers arrive at the frontend with complete context.

* Keep the error type set centralized and fixed, so that consumers (e.g., migration, logging) can reason about the full set of possible error categories.

### Non Goals

* Make error passing more performant. The focus is on correctness and clarity, not on reducing serialization overhead.

* Make errors trivially easy to create for developers. The system enforces a standardized set of error types so that all modules produce errors that are clear and interoperable. While `DynamoError::new()` and `DynamoError::msg()` are straightforward constructors, this proposal does not aim to minimize the effort of sending errors, only to ensure that error information is not lost when they are sent.

* Allow individual modules to define their own error types. Error categories are a fixed, centralized enum so that consumers can reason about the full set.

## Requirements

### REQ 1 Fixed Error Categories

All errors in the Dynamo pipeline **MUST** be categorized using a fixed `ErrorType` enum. New error categories **MUST** be added to the central enum definition, not defined ad-hoc by individual modules.

### REQ 2 Network Serialization

Errors **MUST** be serializable and deserializable for transmission across network boundaries via `Annotated` responses. The serialized form **MUST** preserve the error type, message, and the full cause chain so that the receiving end can reconstruct the error with complete information.

### REQ 3 Error Chaining

Errors **MUST** support chaining via the standard `std::error::Error::source()` method. Each error in the chain is a `DynamoError` with its own type and message. `Display` **MUST** render only the current error (standard Rust convention); callers walk `source()` for the full chain.

### REQ 4 Consumer-Driven Action

The `DynamoError` struct **MUST NOT** carry migration flags. Instead, consumers (e.g., the migration module) **MUST** define which error types trigger which actions. This separation allows consumer policies to evolve independently of the error definitions.

### REQ 5 Clear Error Reporting

The error type name displayed to end users and in logs **MUST** be one of the fixed `ErrorType` variants (e.g., `Disconnected`, `CannotConnect`, `Unknown`). The `Display` output **MUST** follow the format `ErrorType: message`, providing immediate clarity about what category of error occurred without requiring the reader to parse the message text.

# Proposal

## Overview

The proposal introduces two core components:

1. **`ErrorType` enum** (`lib/runtime/src/error.rs`) — A fixed, centralized set of error categories. All Dynamo errors are classified into one of these categories. New categories are added to the enum, not defined by individual modules.

2. **`DynamoError` struct** (`lib/runtime/src/error.rs`) — A serializable error struct that carries an `ErrorType`, a human-readable message, and an optional cause chain. It implements `std::error::Error + Send + Sync + 'static` and `Serialize + Deserialize`.

These integrate with the existing `Annotated` and `MaybeError` protocols for end-to-end structured error handling.

## ErrorType Enum

The `ErrorType` enum defines the fixed set of error categories. It supports subcategories via nested enums for grouping related error types under a common prefix:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum BackendError {
    /// The engine process has shut down.
    EngineShutdown,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum ErrorType {
    /// Uncategorized or unknown error.
    Unknown,
    /// Failed to establish a connection to a remote worker.
    CannotConnect,
    /// An established connection was lost unexpectedly.
    Disconnected,
    /// A connection or request timed out.
    ConnectionTimeout,
    /// Error originating from a backend engine.
    Backend(BackendError),
    // Future: Router(PrefillError), ...
}
```

Each variant represents an error category that is meaningful to both end users and consumers. The enum is centralized so that all consumers can reason about the full set of possible error types.

### Subcategories

Top-level variants like `Unknown`, `CannotConnect`, `Disconnected`, and `ConnectionTimeout` represent common error categories that stand on their own. For domains with multiple related error types, a nested enum groups them under a shared prefix. For example, `Backend(BackendError)` groups all backend engine errors, displayed as `Backend.EngineShutdown`.

Nested enums preserve `Copy`, `Eq`, and `Serialize`/`Deserialize` since all inner variants are unit types. New subcategories are added by defining a new inner enum and adding a variant to `ErrorType`.

Consumers match on subcategories naturally:
```rust
const MIGRATABLE_ERRORS: &[ErrorType] = &[
    ErrorType::Disconnected,
    ErrorType::Backend(BackendError::EngineShutdown),
];
```

## DynamoError Struct

`DynamoError` is the standardized error type:

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DynamoError {
    error_type: ErrorType,
    message: String,
    caused_by: Option<Box<DynamoError>>,
}
```

Fields are private. Public access is via:
- `error_type()` — returns the `ErrorType`
- `message()` — returns the error message
- `source()` — returns the cause via `std::error::Error`

### Display

`Display` renders only the current error, following the standard Rust convention:

```
Disconnected: Stream ended before generation completed
Backend.EngineShutdown: engine shutting down
```

This means the outermost error type is immediately visible in logs and user-facing responses.
- A `Disconnected` error tells the reader "a connection was lost" at a glance
- A `Backend.EngineShutdown` error tells the reader "a backend engine process exited"

All without needing to parse the message text. The full cause chain is available for debugging by walking `source()`.

### Constructors

```rust
// Simple unknown error
let err = DynamoError::msg("something failed");

// Typed error with optional cause
let err = DynamoError::new(ErrorType::Disconnected, "worker lost", None::<DynamoError>);

// Typed error wrapping a cause
let cause = std::io::Error::new(std::io::ErrorKind::ConnectionReset, "reset by peer");
let err = DynamoError::new(ErrorType::Disconnected, "worker lost", Some(cause));
```

The `new()` constructor accepts any `impl std::error::Error + Send + Sync + 'static` as the optional cause. If the cause is already a `DynamoError`, it is preserved as-is. Otherwise, it is recursively converted to a `DynamoError` with `ErrorType::Unknown`.

### Conversions

`DynamoError` can be created from any `std::error::Error`:

- **`From<Box<dyn Error + Send + Sync + 'static>>`** — If the boxed error is already a `DynamoError`, ownership is taken without cloning. Otherwise, it is wrapped as `Unknown` with the display string as the message, and the `source()` chain is recursively converted.

- **`From<&(dyn Error + 'static)>`** — Reference-based conversion. If the error is a `DynamoError`, it is cloned. Otherwise, same wrapping as above.

These conversions ensure that any `std::error::Error` in the pipeline — including `thiserror` enums, `anyhow::Error` contents, and I/O errors — can be converted to `DynamoError` for serialization, with any existing `DynamoError` instances in the `source()` chain preserved with their original `ErrorType`.

## Error Clarity

The standardized error types provide immediate clarity at multiple levels:

**End-user responses.** When an error is returned from an API request, the outermost `DynamoError` is displayed. For example, `Disconnected: Stream ended before generation completed` tells the user that a connection was lost, without exposing internal details.

**Log output.** Structured logging shows the error type first, making it easy to grep and filter. `Disconnected` errors are visually distinct from `CannotConnect` or `Unknown` errors.

**Debugging.** The full cause chain is preserved via `source()`. A developer can walk the chain to see, for example, that a `Disconnected` error was caused by an `Unknown` error from a remote worker, which was in turn caused by an I/O error. Each level in the chain has its own `ErrorType` and message.

**Cross-network context.** When an error is serialized through `Annotated` and deserialized on the other side, the `ErrorType` and full cause chain survive. The frontend can see that a `Disconnected` error originated from a remote engine worker, not from the local network layer.

## Integration with Annotated and MaybeError

The `Annotated<R>` struct carries an optional `DynamoError` field:

```rust
pub struct Annotated<R> {
    pub data: Option<R>,
    pub id: Option<String>,
    pub event: Option<String>,
    pub comment: Option<Vec<String>>,
    pub error: Option<DynamoError>,
}
```

The `MaybeError` trait provides the interface for types that may contain errors:

```rust
pub trait MaybeError {
    fn from_err(err: Box<dyn std::error::Error + Send + Sync>) -> Self;
    fn err(&self) -> Option<DynamoError>;
    fn is_ok(&self) -> bool { !self.is_err() }
    fn is_err(&self) -> bool { self.err().is_some() }
}
```

The `from_err` signature accepts any boxed `std::error::Error`, matching the pre-existing interface. Internally, the error is converted to `DynamoError` for serialization. The `err()` method returns the deserialized `DynamoError` with its full cause chain intact.

## Migration Module Integration

The migration module (`migration.rs`) defines its own policy for which error types are migratable:

```rust
const MIGRATABLE_ERRORS: &[ErrorType] = &[
    ErrorType::CannotConnect,
    ErrorType::Disconnected,
    ErrorType::ConnectionTimeout,
];

const NON_MIGRATABLE_ERRORS: &[ErrorType] = &[
    // Future: ErrorType::Cancelled, etc.
];
```

When the migration module encounters an error — whether from a stream response or a `Result` — it walks the error chain via `source()`, downcasting each error to check if it is a `DynamoError`. If any `DynamoError` in the chain has an `ErrorType` in the migratable set, and no `DynamoError` in the chain has an `ErrorType` in the non-migratable set, the request is retried on another worker. Errors that are not `DynamoError` instances (e.g., `thiserror` enums, I/O errors) are skipped during the walk but do not block migration — only an explicit non-migratable `DynamoError` prevents retry.

## Error Chain Preservation Across Existing Error Types

Existing modules that define their own error types via `thiserror` (e.g., `PrefillError` in `prefill_router.rs`) continue to work. The key requirement is that when a module catches a `DynamoError` from a stream or sub-call, it **MUST** preserve it in the error chain via `#[source]` rather than stringifying it:

```rust
#[derive(Debug, thiserror::Error)]
pub enum PrefillError {
    #[error("Prefill execution failed")]
    PrefillError(#[source] DynamoError),  // preserves the DynamoError in source()
    // ...
}
```

This allows the migration module to walk `source()` and find the `DynamoError` with its original `ErrorType`, even when it is wrapped in a module-specific error type.

# Implementation Details

## Deferred to Implementation

* Extending the pattern to Python bindings and other language interfaces.
* Adding structured error reporting to HTTP/gRPC response bodies (currently errors are returned as plain text messages derived from `Display`).
* Replacing legacy NATS `NoResponders` detection with `ErrorType::CannotConnect`.
* Utility functions for walking error chains and matching against error type sets.

# Implementation Phases

## Phase 0: Core Error System

**Release Target**: Initial

**Effort Estimate**: 1 engineer, 1 week

**Supported API / Behavior:**

* `ErrorType` enum with `Unknown`, `CannotConnect`, `Disconnected`, `ConnectionTimeout`, and `Backend(BackendError)` subcategory with `EngineShutdown`
* `DynamoError` struct with serialization, `From<Box<dyn Error>>`, `From<&dyn Error>`
* `Annotated` and `MaybeError` updated to use `DynamoError`
* Network layer emits `ErrorType::Disconnected` for stream disconnections
* Python bindings emit `ErrorType::Backend(BackendError::EngineShutdown)` on engine process exit
* Migration module with `MIGRATABLE_ERRORS` / `NON_MIGRATABLE_ERRORS` sets, checking both stream-level and Result-level errors

**Not Supported:**

* Full replacement of all `anyhow::Error` usage across the codebase (incremental)
* Additional `ErrorType` variants beyond the initial set

## Phase 1: Extended Error Types and Adoption

**Release Target**: Subsequent

**Effort Estimate**: 1-2 engineers, 1-2 weeks

**Supported API / Behavior:**

* Additional `ErrorType` variants (e.g., `Cancelled`, `ValidationError`, `ResourceExhausted`)
* Updated migration error sets as new types are added
* Gradual adoption of `DynamoError::new(ErrorType::..., ...)` in place of `DynamoError::msg(...)` across pipeline stages
* Replacement of NATS `NoResponders` detection with `ErrorType::CannotConnect`

**Not Supported:**

* Full replacement of all `anyhow::Error` usage across the codebase (incremental)

# Related Proposals

* [DEP 0005 - LLM Request Migration on Engine Failure](/enhancements/deps/0005-llm-request-migration.md)

* [DEP 0001 - Streaming Responses and Error Handling](/enhancements/deps/0001-stream-res-err.md)

# Alternate Solutions

## Alt 1: Use std::error::Error Directly

**Pros:**

* No new struct to learn; uses Rust's standard error trait.
* Broad ecosystem compatibility.

**Cons:**

* `std::error::Error` is not serializable. `source()` returns `&dyn Error` which cannot be serialized for network transmission. A custom serializable wrapper would still be needed.
* No standardized error categorization. `Display` output varies by implementation, making it impossible to programmatically identify error types across network boundaries. End users and log readers would see inconsistent, opaque error messages.

**Reason Rejected:**

* Cannot be serialized over the network without a custom wrapper similar to `DynamoError`, so the implementation effort is comparable while providing neither clear error categories nor consistent user-facing messages.

## Alt 2: Use anyhow::Error

**Pros:**

* Already widely used in the Dynamo codebase.
* Easy to create: `anyhow::anyhow!("message")`.
* Supports downcasting to concrete types.

**Cons:**

* `anyhow::Error` erases type information. Once an error is wrapped in `anyhow`, you can no longer identify its category without downcasting to a specific concrete type, which requires knowing that type at the consumer site.
* Not serializable for network transmission.
* No standardized error categorization. Error messages are free-form strings with no guaranteed structure for log filtering or user-facing display.

**Reason Rejected:**

* Not serializable, so a custom wrapper is still needed for network transmission.
* No error categorization means end users and log readers see opaque messages like `"Stream ended before generation completed"` instead of clear categories like `Disconnected`.

## Alt 3: Use thiserror Enums

**Pros:**

* Generates `std::error::Error` implementations with less boilerplate via derive macros.
* Supports `#[from]` for automatic wrapping and `#[source]` for error chaining.
* Widely used in the Rust ecosystem.

**Cons:**

* Each module would define its own enum variants with potentially different error categories. There is no standardized set of error types, so consumers cannot reason about the full space of possible errors without inspecting every module's enum.
* Not serializable out of the box for network transmission. Custom `Serialize`/`Deserialize` implementations or a serializable wrapper would still be required to transmit errors across network boundaries.
* Error categories are scattered across the codebase rather than centralized, making it difficult to maintain consistent error reporting.

**Reason Rejected:**

* The `DynamoError` struct is intentionally simple — a single struct with an `ErrorType` enum, a message, and a cause chain. Using `thiserror` would add a library dependency for functionality that is straightforward to implement directly. The `DynamoError` struct needs custom serialization behavior (recursive cause chain serialization, `From` conversions that preserve `DynamoError` instances via downcast) that would not benefit from `thiserror`'s derive macros.
* The centralized `ErrorType` enum provides a single, consistent set of error categories that all modules share. `thiserror` enums are still usable within modules (e.g., `PrefillError`), but they wrap `DynamoError` via `#[source]` rather than replacing it, so the standardized categories are preserved through the error chain.

## Alt 4: Per-Error Migration Flags

An alternative to having the migration module decide which error types are migratable is to embed migration flags directly in each error. Each error would carry a flag (e.g., `migratable`, `not_migratable`, or `inherit`) and the migration module would read the flag rather than maintaining its own error type sets.

**Pros:**

* Migration semantics are co-located with the error definition, where the developer has the most context about whether an error is transient or permanent.
* No coordination needed with the migration module when adding new error types.

**Cons:**

* The module defining the error decides migration behavior, not the module handling the error. This inverts the typical pattern where the handler decides what action to take (analogous to exception handling). The migration module should own migration policy, since it has the most context about retry budgets, request state, and system health.
* Developers working on unrelated modules would need to understand migration semantics to set the correct flag on their errors, even though migration is not their concern. This distributes migration knowledge across the codebase rather than centralizing it.
* Each error instance carries an extra field for the migration flag, consuming memory for a property that is a function of the error category, not the error instance. With a centralized `ErrorType` enum, the migration module can derive the same information from the type without per-instance storage.

**Reason Rejected:**

* Migration is a consumer-side decision, not an error-side property. The centralized `ErrorType` enum with consumer-defined error sets (e.g., `MIGRATABLE_ERRORS` in `migration.rs`) provides the same functionality without the per-error overhead and without distributing migration knowledge across modules.

# Background

## References

* [Rust std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html)
* [anyhow crate](https://docs.rs/anyhow)
* [thiserror crate](https://docs.rs/thiserror)
* [RFC 2119 - Key words for use in RFCs](https://datatracker.ietf.org/doc/html/rfc2119)

## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **ErrorType** | A fixed enum of standardized error categories (e.g., `Unknown`, `Disconnected`, `CannotConnect`). All Dynamo errors are classified into one of these categories. |
| **DynamoError** | A serializable struct implementing `std::error::Error` that carries an `ErrorType`, a message, and an optional cause chain. The standardized error type for Dynamo. |
| **Migration** | The process of retrying a failed LLM request on a different engine worker. |
| **Error chain** | A linked sequence of errors where each error may reference an inner cause via `std::error::Error::source()`, forming a chain from the outermost wrapper to the root cause. |
| **Stream-level error** | An error carried inside an `Annotated` response as a `DynamoError`, detected by the migration module via `MaybeError::err()`. |
| **Result-level error** | An error returned as `Err(anyhow::Error)` from a pipeline `generate()` call, detected by the migration module via `downcast_ref::<DynamoError>()`. |
