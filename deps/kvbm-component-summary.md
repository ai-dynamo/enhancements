# KVBM (KV Block Manager) Component Summary

## Overview

The KV Block Manager (KVBM) manages KV cache memory allocation and lifecycle for LLM inference. It provides efficient block-based memory management, supports distributed KV cache across workers, handles KV migration/offloading, and integrates with NIXL for high-performance GPU-to-GPU transfers.

## Location

- **Rust Core**: `lib/llm/src/block_manager/`
- **Python Bindings**: `lib/bindings/python/rust/llm/kv.rs` (partial)
- **Package**: Part of `dynamo-llm` crate

## Internal Dependencies

### Rust Crates

| Crate | Purpose |
|-------|---------|
| `dynamo-runtime` | Runtime coordination |
| `tokio` | Async runtime |
| `cuda-runtime-sys` | CUDA bindings |

### External Dependencies

| Dependency | Purpose |
|------------|---------|
| CUDA | GPU memory management |
| NIXL | High-performance GPU transfers |
| NATS | Event distribution |

## Module Structure

### Core Modules (`lib/llm/src/block_manager/`)

| Module | File | Purpose |
|--------|------|---------|
| `block` | `block.rs`, `block/` | Block data structures |
| `config` | `config.rs` | Configuration |
| `connector` | `connector.rs`, `connector/` | Backend connectors |
| `controller` | `controller.rs`, `controller/` | Block allocation controller |
| `distributed` | `distributed.rs` | Distributed coordination |
| `events` | `events.rs` | KV events (create, delete) |
| `kv_consolidator` | `kv_consolidator/` | KV consolidation |
| `layout` | `layout.rs` | Memory layout |
| `metrics_kvbm` | `metrics_kvbm.rs` | KVBM-specific metrics |
| `numa_allocator` | `numa_allocator.rs` | NUMA-aware allocation |
| `offload` | `offload.rs` | KV offloading to CPU/disk |
| `pool` | `pool.rs`, `pool/` | Block pool management |
| `state` | `state.rs` | Block state tracking |
| `storage` | `storage.rs` | Storage backends |
| `v2` | `v2/` | Next-generation KVBM |

## Public Interface

### Core Types

```rust
// Block management
pub struct BlockManager { ... }
pub struct Block { ... }
pub struct BlockPool { ... }

// Configuration
pub struct KvBlockManagerConfig {
    pub block_size: usize,
    pub num_blocks: usize,
    pub num_layers: usize,
    pub // ...
}

// Events
pub enum KvEvent {
    BlockCreated { block_id: u64, worker_id: String },
    BlockDeleted { block_id: u64, worker_id: String },
    // ...
}
```

### Key Functions

```rust
// Block allocation
pub fn allocate_blocks(num_blocks: usize) -> Result<Vec<Block>>
pub fn free_blocks(blocks: Vec<Block>) -> Result<()>

// KV operations
pub fn store_kv(block: &Block, layer: usize, data: &[u8]) -> Result<()>
pub fn load_kv(block: &Block, layer: usize) -> Result<Vec<u8>>

// Events
pub fn publish_event(event: KvEvent) -> Result<()>
pub fn subscribe_events() -> impl Stream<Item = KvEvent>
```

## User/Developer Interaction

### 1. Backend Integration

KVBM is typically used internally by backend workers:

```python
# Example: vLLM integration
# KVBM is managed by the engine, not directly by user code
```

### 2. Configuration

Via backend CLI arguments:
```bash
python -m dynamo.vllm \
    --block-size 64 \
    --kv-cache-free-gpu-mem-fraction 0.9
```

### 3. KV Events

Workers publish KV events for router awareness:
```python
from dynamo.llm import ZmqKvEventPublisher, ZmqKvEventPublisherConfig

publisher = ZmqKvEventPublisher(config)
# Events published automatically by engine
```

## Packaging & Containers

### Build

Part of `dynamo-llm` crate:
```bash
cargo build --release -p dynamo-llm
```

### Container Integration

- Included in all runtime containers
- Requires CUDA for GPU memory management
- NIXL for disaggregated KV transfer

## Service Interface & I/O Contract

### KV Event Schema

```rust
pub struct KvEvent {
    pub event_type: KvEventType,  // Created, Deleted, Migrated
    pub block_id: u64,
    pub worker_id: String,
    pub timestamp: u64,
    pub metadata: HashMap<String, String>,
}
```

### Event Transport

| Transport | Use Case | Notes |
|-----------|----------|-------|
| NATS JetStream | Persistent events | Default for router sync |
| ZMQ | Low-latency | vLLM native |
| NATS Core | Non-persistent | Local indexer mode |

### Memory Layout

```
┌─────────────────────────────────────────┐
│              GPU Memory                  │
├─────────────────────────────────────────┤
│  Block Pool (KV Cache)                  │
│  ┌─────┬─────┬─────┬─────┬─────┐       │
│  │ B0  │ B1  │ B2  │ ... │ Bn  │       │
│  └─────┴─────┴─────┴─────┴─────┘       │
├─────────────────────────────────────────┤
│  Per-Layer Layout                       │
│  ┌─────────────────────────────────┐   │
│  │ Key [head_dim × num_heads]      │   │
│  │ Value [head_dim × num_heads]    │   │
│  └─────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

## Observability

### Prometheus Metrics (prefix: varies)

| Metric | Type | Description |
|--------|------|-------------|
| `kvbm_blocks_allocated` | Gauge | Currently allocated blocks |
| `kvbm_blocks_free` | Gauge | Free blocks in pool |
| `kvbm_allocation_latency` | Histogram | Block allocation time |
| `kvbm_eviction_count` | Counter | Block evictions |

### Logging

Structured logging for:
- Block allocation/deallocation
- Migration events
- Offload operations
- Error conditions

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| Configuration | Various sources | Unified config schema |
| Event format | Protobuf | Version, document schema |
| Metrics | Basic | Comprehensive metrics |
| Error handling | Various | Unified error types |
| Memory layout | Internal | Document for interop |

## Customization & Extension

### Extension Points

1. **Custom Allocators** - Implement allocator traits
2. **Custom Storage** - Add storage backends
3. **Custom Offload** - Implement offload strategies

### Configuration Options

| Option | Description | Default |
|--------|-------------|---------|
| `block_size` | Tokens per block | 64 |
| `num_layers` | Model layers | Model-specific |
| `gpu_memory_fraction` | GPU memory for KV | 0.9 |
| `enable_offload` | CPU offloading | false |
| `offload_path` | Offload directory | `/tmp` |

### Current Limitations

| Area | Limitation | Workaround |
|------|------------|------------|
| Multi-GPU | Single GPU per worker | Use multiple workers |
| Dynamic sizing | Fixed block size | Choose size at startup |
| Cross-node | NIXL required | Use NIXL-enabled containers |

## Related Documentation

- [Block Manager Design](lib/llm/src/block_manager.md)
- [KV Cache Routing](docs/router/kv_cache_routing.md)
- NIXL documentation

## Tests

- Unit tests in each module
- Integration tests: `tests/kvbm/`
- Performance benchmarks
