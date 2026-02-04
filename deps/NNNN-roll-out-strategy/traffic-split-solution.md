# Rolling Update: Proxy Solution vs. Single Frontend Multi-Pool Solution

## Current Traffic Proxying Implementation

### Architecture Overview

The current implementation uses HAProxy as a traffic proxy to split requests between old and new frontend instances during rolling updates:

```
                         ┌─────────────────────────────────────────────────────┐
                         │                  Old Dynamo Namespace               │
                         │  ┌──────────────┐      ┌──────────────────────────┐ │
                         │  │ Old Frontend │─────▶│ Old Workers (scale down) │ │
                         │  └──────────────┘      └──────────────────────────┘ │
┌────────┐   ┌─────────┐ │         ▲                                           │
│ Client │──▶│ HAProxy │─┤─────────┘                                           │
└────────┘   └─────────┘ │         │                                           │
                         │         ▼                                           │
                         │  ┌──────────────┐      ┌──────────────────────────┐ │
                         │  │ New Frontend │─────▶│ New Workers (scale up)   │ │
                         │  └──────────────┘      └──────────────────────────┘ │
                         │                  New Dynamo Namespace               │
                         └─────────────────────────────────────────────────────┘
```

### Resources Created

For each DGD, the operator creates:

1. **HAProxy Deployment** (`{dgd}-traffic-proxy`)
   - Runs HAProxy pods (configurable replicas)
   - Exposes 4 ports: HTTP (8000), Stats (8404), Runtime API (9999), Metrics (9998)

2. **HAProxy ConfigMap** (`{dgd}-traffic-proxy`)
   - Contains `haproxy.cfg` with backend definitions
   - Defines `old_frontend` and `new_frontend` servers for traffic splitting


### Traffic Shifting Mechanism

1. Controller detects worker spec change → triggers rolling update
2. New frontend + workers created in new dynamo namespace
3. Controller connects to HAProxy Runtime API via TCP
4. Updates backend server addresses (ClusterIPs of old/new frontend services)
5. Adjusts weights proportionally based on ready worker counts
6. Repeat until new=100%, old=0%
7. Delete old namespace resources

### Key Assumption

This implementation assumes **frontends and workers across versions are potentially incompatible**. This drives the need for:
- Isolated dynamo namespaces (workers only see compatible frontend)
- Dual frontends (each frontend talks only to its worker pool)
- External traffic proxy (HAProxy splits traffic at the request level)

---

## Problems and Concerns with HAProxy Approach

### Operational Complexity

| Problem | Description |
|---------|-------------|
| **New dependency** | Introduces HAProxy as a required component that must be maintained, upgraded, and secured |
| **New point of failure** | HAProxy becomes a SPOF for all traffic to the DGD; if HAProxy fails, the entire deployment is unreachable |
| **Configuration surface area** | Exposes new knobs: HAProxy replicas, resources, timeouts, health check intervals, connection limits |
| **Resource overhead** | HAProxy pods consume cluster resources even when no rollout is occurring |
| **Version management** | HAProxy image must be tracked, updated for CVEs, and tested for compatibility |

### Architectural Concerns

| Concern | Description |
|---------|-------------|
| **Latency overhead** | Additional network hop through HAProxy adds latency to every request |
| **Observability complexity** | Need to monitor HAProxy metrics separately; correlating issues across proxy and backends is harder |
| **Service mesh conflicts** | May conflict with Istio/Linkerd sidecars that expect direct pod-to-pod traffic |
| **Health check mismatch** | HAProxy health checks (`/health`) may not reflect actual frontend readiness to serve inference requests |

### Controller Complexity

| Concern | Description |
|---------|-------------|
| **Operator bloat** | ~450 lines for HAProxy client, ~390 lines for resource generation, ~300+ lines for rollout orchestration |
| **Runtime API fragility** | Controller-to-HAProxy communication can fail due to network issues, API errors, or timing |
| **Race conditions** | Controller reconcile loop and HAProxy state can get out of sync |
| **State reconciliation** | HAProxy runtime state (weights) is not persisted; ConfigMap and runtime can diverge |

### Rollout Strategy Questions

| Question | Description |
|---------|-------------|
| **When to use this vs in-place?** | No clear guidance on when HAProxy-based rollout is needed vs standard Kubernetes rolling update. For example, what if there is a frontend only change |
| **Continued support of Shared Frontend?** | No clear guidance on how to support shared frontend in the new approach. For example, this advanced rollout doesn't work if a frontend isn't defined in the DGD |

### Resource Duplication

| Resource | During Rollout | Steady State |
|----------|----------------|--------------|
| Frontend pods | 2x (old + new) | 1x |
| Worker pods | Up to 2x during transition | 1x |
| HAProxy pods | Always running | Always running |
| Services | 4 extra (2 frontends + 2 HAProxy) | 2 extra (HAProxy) |

---

## Alternative Approach: Single Frontend with Multi-Pool Support

### Key Assumption Change

Instead of assuming **frontends and workers are incompatible across versions**, assume they **are compatible**. This eliminates the need for:
- Dual frontends
- HAProxy traffic proxy
- External traffic splitting

### Architecture Overview

```
                    ┌──────────────────────────────────────────────────────────┐
                    │         Single Frontend (--namespace-prefix=default-myapp)│
                    │  ┌────────────────────────────────────────────────────┐  │
                    │  │    Multi-Pool Manager (discovers via prefix match) │  │
                    │  │                                                    │  │
┌────────┐          │  │   ┌─────────────────┐    ┌─────────────────┐      │  │
│ Client │─────────▶│  │   │ default-myapp-  │    │ default-myapp-  │      │  │
└────────┘          │  │   │ abc12345 (old)  │    │ def67890 (new)  │      │  │
                    │  │   │ 7 instances→70% │    │ 3 instances→30% │      │  │
                    │  │   └─────────────────┘    └─────────────────┘      │  │
                    │  │          │                        │               │  │
                    │  └──────────┼────────────────────────┼───────────────┘  │
                    └─────────────┼────────────────────────┼──────────────────┘
                                  ▼                        ▼
                    ┌──────────────────────┐  ┌──────────────────────┐
                    │    Old Workers       │  │    New Workers       │
                    │  (prefill, decode)   │  │  (prefill, decode)   │
                    └──────────────────────┘  └──────────────────────┘
```

**Key insights**:
- Prefix `default-myapp` matches both `default-myapp-abc12345` and `default-myapp-def67890`
- Weights derived from discovered instance counts - no explicit configuration needed
- New pools auto-discovered when operator creates them

### How It Works

1. **Single frontend** deployed (no duplication)
2. Frontend uses **namespace prefix matching** (e.g., `default-myapp`) to discover all pools
3. Workers automatically grouped by **full namespace** (e.g., `default-myapp-abc123`, `default-myapp-def456`)
4. **Weights derived from instance counts** - no explicit weight configuration needed
5. As operator scales down old / scales up new, weights automatically shift
6. When old pool reaches 0 instances, all traffic goes to new pool

**No configuration changes needed during rollout** - new pools are discovered automatically via prefix match.

## Comparison: HAProxy vs Single Frontend Multi-Pool

### Complexity

| Aspect | HAProxy Approach | Single Frontend Multi-Pool |
|--------|------------------|---------------------------|
| New external dependencies | HAProxy | None |
| Operator code added | ~1100+ lines | ~0 lines (just scale workers) |
| Frontend/router changes | None | ~200-300 lines |
| Resources to manage | 4 extra (Deployment, ConfigMap, 2 Services) | 0 extra |
| Points of failure | HAProxy + controller-to-HAProxy communication | Frontend pool logic |
| Weight management | Explicit via Runtime API | Implicit via instance counts |

### Operational

| Aspect | HAProxy Approach | Single Frontend Multi-Pool |
|--------|------------------|---------------------------|
| Latency | +1 network hop always | No overhead |
| Resource usage | HAProxy pods always running | No overhead |
| Observability | Separate HAProxy metrics | Unified frontend metrics |
| Service mesh compatibility | Potential conflicts | Native compatibility |
| Scaling | Must scale HAProxy separately | Frontend scales as usual |

### Rollout Behavior

| Aspect | HAProxy Approach | Single Frontend Multi-Pool |
|--------|------------------|---------------------------|
| Traffic splitting | L7 proxy (request-level) | Application-level (request-level) |
| Weight precision | HAProxy weights (0-256) | Instance count granularity |
| Weight update mechanism | Runtime API call from operator | Automatic via discovery |
| Weight update latency | Reconcile loop + API call | Discovery propagation (~seconds) |
| Rollback | Delete new namespace + reset HAProxy | Scale old pool back up |

### Trade-offs

| HAProxy Approach | Single Frontend Multi-Pool |
|------------------|---------------------------|
| ✅ No frontend code changes | ✅ No external dependencies |
| ✅ Works with any frontend version | ✅ Lower operational complexity |
| ❌ Always-on resource overhead | ✅ Zero overhead when not rolling |
| ❌ New SPOF | ✅ Simpler failure modes |
| ❌ Operator complexity explosion | ✅ No operator weight management |
| ❌ Dual frontend duplication | ✅ Self-healing (auto-adjusts to failures) |
| ❌ Explicit weight synchronization | ❌ Requires frontend changes |
| | ❌ Assumes FE-worker compatibility |

---

## Frontend-Worker Compatibility Surface Area

### Communication Layers

| Layer | Mechanism | Compatibility Impact |
|-------|-----------|---------------------|
| **Transport** | TCP / HTTP / NATS (configurable via `--request-plane`) | Low - abstracted, selectable |
| **Discovery** | Kubernetes CRDs (default) or etcd/KV store | Low - see below |
| **Request handling** | Async generator pattern | Medium - signature must match |
| **Streaming** | `AsyncGenerator[dict, None]` yields | Medium - dict format matters |

### Discovery Backends

| Backend | Default | How It Works |
|---------|---------|--------------|
| **Kubernetes** | ✅ Yes | `DynamoWorkerMetadata` CRD + EndpointSlices |
| **etcd/KV Store** | No | Key-value watches at `v1/mdc/...` paths |

### Data Contracts (Dynamo Runtime Level)

**PreprocessedRequest** - the core FE→Worker contract:
```python
class PreprocessedRequest(BaseModel):
    token_ids: List[int]           # Tokenized input (FE tokenizes)
    stop_conditions: StopConditions
    sampling_options: SamplingOptions
    eos_token_ids: List[int]
    mdc_sum: Optional[str]         # MDC checksum for validation
    annotations: List[str]
```

**ModelDeploymentCard (MDC)** - shared model configuration:
- `model_info`: HuggingFace config.json with checksum
- `tokenizer`: Tokenizer artifacts with checksum
- `prompt_formatter`: Chat template
- `runtime_config`: Tool/reasoning parsers
- Stored at: `v1/mdc/{namespace}/{component}/{endpoint}/{instance_id}`

**Key insight**: The frontend tokenizes the request using its MDC, then sends `token_ids` to workers. Workers use the `mdc_sum` to validate they have a compatible MDC.

### KV Events (Worker → Router Communication)

KV events are a **critical compatibility surface** for the KV-aware router. Workers emit events when KV cache blocks are allocated or evicted, enabling the router to make cache-aware routing decisions.

#### Event Flow

```
Worker (vLLM/SGLang/TRTLLM)
    │
    ├─► Option 1: Direct NATS publishing (KvEventPublisher)
    │       └─► NATS subject: "kv-events"
    │
    └─► Option 2: ZMQ → NATS bridge (ZmqKvEventPublisher)
            └─► ZMQ socket → Dynamo binding → NATS
                    │
                    ▼
            KV Router (KvIndexer)
                    │
                    ▼
            RadixTree (prefix matching for routing)
```

#### Event Types and Structure

**Defined in**: `lib/llm/src/kv_router/protocols.rs`

```rust
pub struct KvCacheEvent {
    pub event_id: u64,           // Monotonically increasing per worker
    pub data: KvCacheEventData,  // Stored | Removed | Cleared
    pub dp_rank: DpRank,         // Data parallel rank (0 if not DP)
}

pub enum KvCacheEventData {
    Stored(KvCacheStoreData),    // Blocks added to cache
    Removed(KvCacheRemoveData),  // Blocks evicted
    Cleared,                     // All blocks removed
}

pub struct KvCacheStoreData {
    pub parent_hash: Option<ExternalSequenceBlockHash>,
    pub blocks: Vec<KvCacheStoredBlockData>,  // token_ids, block_hash, num_tokens
}
```

### Breaking Change Risk Assessment

| Contract | Risk Level | What Breaks | Who Owns It |
|----------|------------|-------------|-------------|
| **PreprocessedRequest fields** | 🔴 High | Adding/removing/renaming fields | Dynamo |
| **MDC schema** | 🔴 High | Checksum validation fails | Dynamo |
| **KvCacheEvent schema** | 🔴 High | Router can't parse events, routing fails | Dynamo |
| **SamplingOptions/StopConditions** | 🟡 Medium | New sampling params not forwarded | Dynamo |
| **Framework SamplingParams** | 🟡 Medium | vLLM/SGLang version mismatch | Framework |
| **Handler async signature** | 🟡 Medium | Must yield dicts, not raw objects | Dynamo |
| **KV block hash algorithm** | 🟡 Medium | Cache overlap detection fails | Dynamo + Framework |
| **kv_block_size** | 🟡 Medium | Block boundaries don't align | Dynamo config |
| **Transport protocol** | 🟢 Low | Abstracted, backwards compatible | Dynamo |
| **Discovery mechanism** | 🟢 Low | K8s CRDs or etcd paths, same interface | Dynamo |

### What Dynamo Controls vs Framework

```
┌─────────────────────────────────────────────────────────────────┐
│                     DYNAMO RUNTIME LAYER                        │
│  • PreprocessedRequest/Response schemas                         │
│  • MDC format and checksum                                      │
│  • Discovery protocol (etcd paths)                              │
│  • Transport abstraction (TCP/HTTP/NATS)                        │
│  • Handler registration pattern                                 │
│  • KV router event format                                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FRAMEWORK LAYER                             │
│  • SamplingParams field names/types (vLLM vs SGLang differ)     │
│  • Engine initialization API                                    │
│  • RequestOutput structure                                      │
│  • KV cache block internals                                     │
│  • Multimodal input handling                                    │
└─────────────────────────────────────────────────────────────────┘
```
