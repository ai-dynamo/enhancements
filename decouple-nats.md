# Nats decoupling: Transport-Agnostic pipeline

## Overview

The goal is to decouple the NATs transport from the pipeline.

## Requirements

- Ability to deliver messages across dynamo instances with at least once delivery guarantee.
- Ability to switch between transports at runtime.


##  Durability guarantees

### At least once delivery
- No message loss is possible.
- Message is delivered at least once to the consumers

### Exactly once delivery
- needs ack/nack coordination and stateful tracking of messages to ensure once delivery.

### At most once delivery
- Message loss is possible.


## Current NATs use cases
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
- Generalize ingress/egress components to support different transports 


## Alternative solutions

### Message queue transports
 - ZeroMQ
 - Redis
 - Kafka
 - SQS
 - GCP PubSub
 - Azure Service Bus

### Object store
 - S3
 - Redis
 - Shared filesystem 

### Ideas

1. Use RocksDB / local storage to persist messages on producer side to guarantee at least once delivery.
