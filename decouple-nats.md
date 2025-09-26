# Nats decoupling: Transport-Agnostic pipeline

## Overview

The goal is to decouple the NATs transport from the dynamo runtime. 
Introduce abstractions for current NATs usages (e.g. KV router, event plane, request plane & object store, etc) which can be used to plug different implementations.

## Requirements
- Ability to deliver messages across dynamo instances with at least once delivery guarantee.
  Ensure serialized bytes can be reliably delivered in p2p and broadcast modes.

- Ability to switch between transports at runtime.

### Event Plane
- KV events needs to be delivered with at least once delivery guarantee. Currently KV events are handled in idempotent manner and redelivered messages are ignored.

#### Metrics 

### Object Store

### Request Plane
- need protocol for request context and control messages for request cancellation

### Message Delivery Guarantees

#### At least once delivery (preferred)
- No message loss is possible.
- Message is delivered at least once to the consumers
- consumers should be idempotent and be able to handle duplicate messages.

#### Exactly once delivery
- needs stateful tracking of messages and ack/nack coordination to ensure exactly once delivery.

#### At most once delivery
- Message loss is possible.

## Current NATs use cases

## NATs use cases

### 1. NatsQueue python binding
- **Location**: `lib/bindings/python/rust/llm/nats.rs` (`NatsQueue`)
- **Functionality**:
- Deprecated: We don't use `NatsQueue` python binding anymore. We use `NatsQueue` rust binding instead.
- We can remove the python binding and the associated tests to simplify the codebase.

#### 2. JetStream-backed Queue/Event Bus
- **Location**: `lib/runtime/src/transports/nats.rs` (`NatsQueue`)
- **Functionality**:
  - Stream creation per subject pattern `{stream_name}.*`
  - Publisher-only, worker-group, and broadcast consumer modes
  - Durable consumers with pull-based consumption
  - Administrative operations (purge, consumer management)

#### 3. Event Publishing for KV Router
- **Location**: `lib/llm/src/kv_router/publisher.rs`
- **Functionality**:
  - Publishes KV cache events from ZMQ or direct sources
  - Uses `EventPublisher` trait to send events

#### 4. Event Consumption for KV Router
- **Location**: `lib/llm/src/kv_router/subscriber.rs`
- **Functionality**:
  - Consumes `RouterEvent` messages via durable consumers
  - Handles state snapshots and stream purging

#### 5. Object Store (JetStream Object Store)
- **Location**: `lib/runtime/src/transports/nats.rs`
- **Functionality**:
  - File upload/download operations
  - Typed data serialization with bincode
  - Bucket management and cleanup

#### 6. Key-Value Store (JetStream KV)
- **Location**: `lib/runtime/src/storage/key_value_store/nats.rs`
- **Functionality**:
  - Implements `KeyValueStore` trait
  - CRUD operations with conflict resolution
  - Watch streams for real-time updates

#### 7. Request/Reply Pattern
- **Location**: `lib/runtime/src/transports/nats.rs`
- **Functionality**:
  - Service stats collection via broadcast requests
  - Each service responds once to stats queries

## Proposal

- Use named message bus to publish and subscribe to messages.
- Support different transports (e.g. Raw TCP, Nats) for request/reply pattern.
- Introduce abstractions for each NATs usage (e.g. KV router, Jet stream, object store, etc).

### Implementation 
- Phase 1
	* degraded feature set
		* not use KV router if they want. Best effort 
	* nats
		* No HA guarantees for router
		* Operate without high availability w/ single router
- Phase 2
   * explore transports (QUIC, Multicast)
	 * durability
	 * exactly once delivery


## Generic Messaging Protocol
Decouple messaging protocol from the underlying transport like Raw TCP, ZMQ or (HTTP, GRPC, and UCX active message).

Phase approach: start with ZMQ and Nats. Later, incrementally expand to support more advanced transports, ensuring that the protocol remains adaptable to requirements.

## Handshake and Closure Protocols: 
Robust handshake and closure protocols, including the use of sentinels or envelope structures to signal the end of streams or requests.
A common semantic for closing requests and handling errors, will be generalized across different transports.

## Multipart Message Structure
Use a multipart message structure, inspired by ZMQ's native multipart support, to encapsulate headers, body, and control signals (such as closure control signals or error notifications). 

Extend existing two-part message structure to support N-part messages, making the protocol more flexible and expressive.

handshake protocols and message flows for key transports (Raw TCP, HTTP SSE, ZMQ, GRPC, UCX), distilling a protocol that works across all. They emphasized the value of starting with simple transports and expanding to more complex ones, ensuring the protocol can accommodate future needs and additional transports.

## Python-Rust Interoperability and Data Class Generation
Strategies for improving Python-Rust interoperability, focusing on auto-generating Python data classes from Rust structs using Pydantic, and aligning message schemas to reduce manual coding and serialization errors.

### Support transports
 - Raw TCP
 - ZMQ
 - HTTP SSE
 - GRPC
 - UCX active messaging
 - Nats

## Milestones
1. Implement abstractions for each NATs usage

2. Implement different transports for request/reply pattern
a. Interface for Request Plane
b. sending requests over direct ZMQ 

3. Implement different transports for KV router

4. Implement different transports for event bus

5. Object store:
  a. interface for object store
  b. object store implementation using shared filesystem
  c. object store implementation using model express