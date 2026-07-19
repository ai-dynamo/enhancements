# Dynamo Event Plane

**Status**: Under Review

**Authors**: [biswa](https://github.com/biswapanda)

**Category**: Architecture

**Sponsor**: Neelay Shah, Itay Neeman, Suman Tatiraju, Maksim Khadkevich

**Required Reviewers**: Neelay, Itay, Suman, Maksim

**Review Date**: 2026-01-22

**Pull Request**: TBD

** version 1**: 2026-01-22: initial version
** version 2**: 2026-01-26: added zmq based broker mode

## Summary

The Event Plane provides a transport-agnostic pub/sub interface for Dynamo components. It unifies NATS and ZMQ under a consistent API, exposes strongly scoped publisher/subscriber types, and supports dynamic discovery for ZMQ to automatically connect to publishers at runtime.

## Motivation

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

### REQ 4 Dynamic Discovery

Publishers/Subscribers **MUST** support dynamic discovery so new publishers or subscribers are connected automatically.

## Proposal


### Overview

The API is split into two primary entry points: `EventPublisher` and `EventSubscriber`. Each is scoped to a namespace or component and a specific topic.

Publishers auto-register with discovery; subscribers auto-discover publishers.

Transport selection is controlled by `DYN_EVENT_PLANE`.

### Event Plane Architecture

At a high level, the event plane is composed of four layers:

1. **Publisher and Subscriber layer**: `EventPublisher` and `EventSubscriber` provide the user-facing API. Publishers auto-register with discovery; subscribers auto-discover publishers.

2. **Envelope and Codec layer**: `EventEnvelope` adds topic and sequencing metadata; codecs serialize/deserialize (legacy JSON for NATS, MsgPack for ZMQ). The topic is included in the envelope to allow filtering when the transport does not provide native topic filtering.

3. **Transport Adapter layer**: transport-specific Tx/Rx implementations handle NATS Core subjects or ZMQ PUB/SUB sockets.

4. **Discovery Plane (control)**: registers event channels and publishes membership changes so subscribers can connect to new publishers dynamically.

Key points:
- Publishers/consumers are scoped by `namespace/component/topic`.
- The envelope carries topic and sequence metadata so filtering is consistent across transports.
- Discovery is used only for control-plane concerns (publisher registration and dynamic subscriber updates).

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
- Auto-registers with discovery plane on creation.
- Includes `topic` in `EventEnvelope` to allow filtering when the transport does not provide native topic filtering.
- Uses NATS subject prefix for NATS transport; uses ZMQ topic frame for ZMQ transport.

### Subscriber API

Subscribers are created with a topic and scope. They auto-discover publishers.

```rust
use dynamo_runtime::transports::event_plane::EventSubscriber;

let mut subscriber = EventSubscriber::for_component(&component, "kv-events").await?;
while let Some(result) = subscriber.next().await {
    let envelope = result?;
    // envelope.topic == "kv-events" and envelope.publisher_id is the id of the publisher
    // envelope.payload is the serialized event payload
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
    pub publisher_id: u64,       // unique id of the publisher
    pub sequence: u64,           // monotonically increasing sequence number per publisher
    pub published_at: u64,       // timestamp in milliseconds when the event was published
    pub topic: String,           // topic of the event
    pub payload: Bytes,          // serialized event payload
}
```

### ZMQ Dynamic Discovery and Pub/Sub

ZMQ uses the Discovery plane as a control channel to learn publisher endpoints and keep subscriptions current as publishers come and go.

**Registration Flow**

1. A publisher binds a ZMQ PUB socket on an OS-assigned port.
2. The publisher registers an `EventChannel` in discovery with:
   - `namespace`, `component`, and `topic`
   - transport details containing the ZMQ endpoint (e.g., `tcp://host:port`)
3. The discovery entry is stored under the key:
   - `namespace/component/topic/{instance_id}`

**Subscription Flow**

1. A subscriber constructs a discovery query for the topic:
   - `DiscoveryQuery::TopicEventChannels { namespace, component, topic }`
2. It calls `list_and_watch()` on the discovery client to:
   - get all current publishers (initial list)
   - receive future `Added`/`Removed` events for the same topic
3. For each `Added` event, the subscriber connects a ZMQ SUB socket to the endpoint.
4. Incoming messages from all publishers are merged into one stream.

**Runtime Behavior**

- The subscriber continuously watches discovery for changes. New publishers are connected without restart.
- When a publisher is removed from discovery, the subscriber currently logs the removal and relies on the stream to end naturally when the publisher disappears. This is safe but may leave a brief stale connection.
- Topic filtering is enforced at the envelope level (via `EventEnvelope.topic`) for consistency across transports.

**MsgPack Frame (ZMQ)**

ZMQ uses a compact binary frame format for high-performance transport. Each message is sent as a ZMQ multipart payload:

- Frame 0: topic (string)
- Frame 1: binary frame (8-byte header + MsgPack-encoded `EventEnvelope`)

The 8-byte header encodes protocol version, message type, codec type, flags, and payload length. This keeps framing overhead low while allowing future extension.

This design keeps discovery as the authoritative source of active publishers while allowing ZMQ to scale with dynamic process lifecycles.

**Discovery Plane Integration**

- **KV Store / etcd discovery**: entries are written under the `v1/event_channels` bucket using the topic-aware key format.
- **Kubernetes discovery**: event channel registrations are reflected in pod metadata and propagated via CR updates.

**DiscoverySpec::EventChannel**

`DiscoverySpec::EventChannel` is the discovery-plane record used to register event publishers. It includes:

- `namespace` and `component` scope
- `topic` name (e.g., `kv-events`, `kv-metrics`)
- `transport` details (NATS subject or ZMQ endpoint)

This record is written under the key `namespace/component/topic/{instance_id}` and is the authoritative source for dynamic subscriber discovery.

This design keeps discovery as the authoritative source of active publishers while allowing ZMQ to scale with dynamic process lifecycles.

### ZMQ Scaling: Direct Mode vs Broker Mode

ZMQ provides two deployment modes to address different scalability requirements: **Direct Mode** and **Broker Mode**. The choice depends on the number of publishers and reliability requirements.

#### Direct Mode

In direct mode, each publisher binds a ZMQ PUB socket, and subscribers connect directly to all publishers via discovery.

**Connection Model:** O(P × S) where P = publishers, S = subscribers
- Example: 100 publishers × 10 subscribers = 1,000 connections
- Example: 1,000 publishers × 10 subscribers = 10,000 connections

**Characteristics:**
- Lowest latency (direct publisher → subscriber path)
- No single point of failure or broker infrastructure
- Connection count grows multiplicatively with publishers and subscribers
- Best suited for < 100 publishers

#### Broker Mode

Broker mode introduces a centralized XSUB/XPUB relay that intermediates between publishers and subscribers, reducing connection count at scale.

**Connection Model:** O(P + S) where P = publishers, S = subscribers
- Example: 100 publishers + 10 subscribers = 110 connections (99% reduction)
- Example: 1,000 publishers + 10 subscribers = 1,010 connections (99% reduction)

**Activation:**

Broker mode is activated via environment variables:

```bash
# Explicit broker URL
export DYN_ZMQ_BROKER_URL="xsub=tcp://broker:5555 , xpub=tcp://broker:5556"

# Or discovery-based
export DYN_ZMQ_BROKER_ENABLED=true
```

No code changes required - the Event Plane automatically switches modes based on configuration.

**Characteristics:**
- Scales to 10,000+ publishers with zero message loss
- Centralized relay simplifies network topology
- Adds one network hop (slightly higher latency)
- Requires broker deployment
- Supports multi-broker HA for redundancy

#### Multi-Broker High Availability

For production deployments requiring high availability, broker mode supports multiple broker instances with automatic failover.

**Architecture:**

Multiple brokers operate independently. Publishers and subscribers connect to all brokers simultaneously:

```
Publishers → Broker 1 → Subscribers
          ↘ Broker 2 ↗
```

- Brokers can be shareded hash based assigned based on HRW hashing for load balancing in next phase.Each publisher can publish to 2 brokers based on stable HRW hash and subscribers subscribe to all brokers. This can help scale the brokers.

**Configuration:**

```bash
# Multiple brokers (semicolon-separated)
export DYN_ZMQ_BROKER_URL="xsub=tcp://broker1:5555;tcp://broker2:5555 , xpub=tcp://broker1:5556;tcp://broker2:5556"
```

**Behavior:**
- Publishers load-balance messages across all brokers (ZMQ built-in)
- Subscribers receive from all brokers for redundancy
- If one broker fails, messages continue through remaining brokers
- No coordinator needed - brokers are stateless and independent

**Deduplication:**

Since subscribers receive messages from multiple brokers, client-side deduplication ensures each message is processed once. The Event Plane uses an LRU cache with `(publisher_id, sequence)` tuples from the EventEnvelope to filter duplicates automatically.

**Validation:** Tested with 2,000 publishers and 2 brokers - received exactly 20,000 messages (not 40,000), confirming zero duplicates.

**Benefits:**
- No single point of failure
- Automatic failover without coordination
- Zero duplicate messages delivered to applications
- Transparent to application code

**Trade-offs:**
- Higher latency due to deduplication overhead
- Additional memory for deduplication cache
- More network connections (connects to each broker)

#### Selection Guidance

| Publishers | Recommended Mode | Rationale |
|------------|------------------|-----------|
| < 100 | Direct | Simple, low latency, manageable connections |
| 100-1,000 | 2 Brokers | Zero message loss, O(P+S) scaling |
| 1,000+ | Multi-Broker HA | High availability, load distribution |

**Migration:**

Switching modes requires no code changes - only environment variable configuration and process restart. Rollback is similarly simple by unsetting broker environment variables.

## Alternate Solutions

### Alt 1 Single Type for Publisher/Subscriber

**Pros:**
- Fewer types to learn and use
- Backward compatibility with older API

**Cons:**
- Always creates both PUB and SUB transports, even when unused
- Confusing lifecycle and higher resource usage (PUB and SUB sockets are created even when only one is needed)
- Inconsistent semantics when switching transports

**Reason Rejected:**
- Splitting publisher/subscriber provides clearer UX and avoids accidental side effects.

### Alt 2 Keep any transport-specific APIs or semantics

**Reason Rejected:**
- The event plane requires a unified interface to support runtime configuration and portability.
