# Dynamo Runtime: Event Plane API

**Status:** Draft  
**Authors:** [@biswapanda](https://github.com/biswaranjanp)  
**Category:** Architecture | Process | Guidelines  
**Replaces:** N/A  
**Replaced By:** N/A  
**Sponsor:** TBD  
**Required Reviewers:** TBD  
**Review Date:** 2026-01-21  
**Pull Request:** TBD  
**Implementation PR / Tracking Issue:** TBD  

## Summary

**[Required]**

The Event Plane provides a transport-agnostic pub/sub interface for Dynamo components. It unifies NATS and ZMQ under a consistent API, exposes strongly scoped publisher/subscriber types, and supports dynamic discovery for ZMQ to automatically connect to publishers at runtime.

## Motivation

**[Required]**

We need a single event interface that works across transports while providing the same developer experience. Prior iterations mixed pub and sub behaviors into one type, required manual discovery registration, and left topic semantics implicit or transport-specific. This led to confusion, coupling, and subtle failure modes when switching transports (e.g., ZMQ topic filtering).

This document establishes a high-level API and UX that makes transport selection explicit, auto-registers publishers, and ensures topic semantics are consistent across NATS and ZMQ. It also documents the dynamic discovery and multi-publisher fan-in pattern used by ZMQ.

## Goals

- Provide a clean publisher/subscriber API that does not require manual registration.
- Ensure topic semantics are consistent across NATS and ZMQ.
- Support discovery-based ZMQ fan-in that handles dynamic publisher lifecycles.
- Allow transport selection via an environment variable for deployments.

## Non Goals


- Providing durable message persistence (NATS JetStream remains out of scope).
- Guaranteeing exactly-once delivery semantics.
- Providing cross-topic multiplexing on a single subscriber instance.

## Requirements

### REQ 1 Transport Selection

The Event Plane **MUST** support selecting a transport via the `DYN_EVENT_PLANE` environment variable with `nats` as the default and `zmq` as an alternative.

### REQ 2 Consistent Topic Semantics

Publish and subscribe APIs **MUST** interpret the topic consistently across transports. Topics **MUST** be included in the event envelope to allow filtering when the transport does not provide native topic filtering.

### REQ 3 Auto-Registration

Publishers **MUST** automatically register with discovery on creation. Users **MUST NOT** be required to call a separate registration API.

### REQ 4 Dynamic Discovery (ZMQ)

ZMQ subscribers **MUST** support dynamic discovery so new publishers are connected without restarting the subscriber.

## Proposal

**[Required]**

### Overview

The API is split into two primary entry points: `EventPublisher` and `EventSubscriber`. Each is scoped to a namespace or component and a specific topic. Publishers auto-register with discovery; subscribers auto-discover publishers. Transport selection is controlled by `DYN_EVENT_PLANE`.

### Environment Variable

- `DYN_EVENT_PLANE`: controls transport selection.
  - `nats` (default): NATS Core pub/sub
  - `zmq`: ZMQ pub/sub with dynamic discovery

### Publisher API

Publishers are created with a topic and scope. They auto-register with discovery.

```rust
use dynamo_runtime::transports::event_plane::EventPublisher;

let publisher = EventPublisher::for_component(&component, "kv-events").await?;
publisher.publish(&event).await?;
```

Key behaviors:
- Auto-registers with discovery on creation.
- Includes `topic` in `EventEnvelope`.
- Uses NATS subject prefix for NATS transport; uses ZMQ topic frame for ZMQ transport.

### Subscriber API

Subscribers are created with a topic and scope. They auto-discover publishers.

```rust
use dynamo_runtime::transports::event_plane::EventSubscriber;

let mut subscriber = EventSubscriber::for_component(&component, "kv-events").await?;
while let Some(result) = subscriber.next().await {
    let envelope = result?;
    // envelope.topic == "kv-events"
}
```

Key behaviors:
- Uses discovery to find matching publishers.
- Filters by `EventEnvelope.topic` to ensure consistent semantics across transports.
- Supports typed subscriptions via deserialization helpers.

### Event Envelope

All events are wrapped in an `EventEnvelope` that includes a topic and sequencing metadata.

```rust
pub struct EventEnvelope {
    pub publisher_id: u64,
    pub sequence: u64,
    pub published_at: u64,
    pub topic: String,
    pub payload: Bytes,
}
```

### ZMQ Dynamic Discovery and Pub/Sub

ZMQ uses discovery to connect subscribers to publishers dynamically:

1. Publishers bind to an OS-assigned port and register their endpoint with discovery.
2. Subscribers query discovery for matching `namespace/component/topic`.
3. Subscribers connect to all discovered endpoints and keep watching for new ones.
4. New publishers are added without subscriber restart.

The dynamic subscriber merges multiple publisher streams into a single stream and applies topic filtering at the envelope level.

## Alternate Solutions

**[Required, if not applicable write N/A]**

### Alt 1 Single EventPlane Type for Pub/Sub

**Pros:**
- Fewer types to learn
- Backward compatibility with older API

**Cons:**
- Always creates both PUB and SUB transports, even when unused
- Confusing lifecycle and higher resource usage
- Inconsistent semantics when switching transports

**Reason Rejected:**
- Splitting publisher/subscriber provides clearer UX and avoids accidental side effects.

### Alt 2 Transport-Specific APIs

**Pros:**
- Maximum control and performance per transport
- No abstraction overhead

**Cons:**
- Duplicated code paths
- Inconsistent developer experience across transports
- Harder to switch environments

**Reason Rejected:**
- The event plane requires a unified interface to support runtime configuration and portability.
