# KV Cache Controller: Design and API

**Status**: Draft

**Authors**: Moein Khazraee, Harry Xie, Lin Hu, Olga Andreeva, Adit Ranadive, Chang Guo, Ziqi Fan, Ryan Olson, Omri Kahalon

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

## Contents

- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
- [Proposal](#proposal)
  - [Principles](#principles)
  - [Component Roles](#component-roles)
  - [Memory Management: Enabling Principles 1 and 4](#memory-management-enabling-principles-1-and-4)
  - [Router–KV Cache Controller Dynamics: Enabling Principles 2 and 5](#routerkv-cache-controller-dynamics-enabling-principles-2-and-5)
  - [KV Cache Controller Policy and Interface: Enabling Principle 3](#kv-cache-controller-policy-and-interface-enabling-principle-3)
  - [State Model and Operation Lifecycle](#state-model-and-operation-lifecycle)

---

# Summary

This document describes a plan to improve the KV Block Manager (KVBM) by making better use of capabilities already present in the KV router and by simplifying the cache-management boundary. The resulting component is called the **KV Cache Controller** throughout this document.

Existing systems such as the Dynamo KV Router demonstrate that global cache awareness and request routing can be achieved efficiently. Because the router already tracks KV cache placement and uses it in routing decisions, it is a natural component to work hand in hand with the cache controller rather than duplicate that global view elsewhere. The router can also incorporate harness-level context, such as relationships among sub-agent requests, and pass relevant hints to the KV Cache Controller. Continued improvements to this coordination layer—such as the [Flash Indexer](https://docs.nvidia.com/dynamo/dev/digest/flash-indexer#7-benchmarks)—make that boundary increasingly practical at scale. The KV Cache Controller can therefore focus on local cache policy, residency, and data movement while reusing the router's global inventory and request-scoped hints.

The KV Cache Controller will be developed in the open as part of Dynamo. A public repository will be available soon, with the design and implementation evolving through community collaboration.

---

# Motivation

Inference engines managing KV caches face a hierarchy of storage tiers — HBM (G1) → Host DRAM / controller-attached pool (G2) → Local SSD (G3) → Object Store (G4) — across both single nodes and large clusters. Current approaches suffer from one of two failure modes: tight GPU coupling, where external entities manage GPU-side KV blocks and compete for kernel launch bandwidth, requiring continuous synchronization with the engine; or naive simplicity, where eviction happens reactively under peak memory pressure with no admission filtering and no cross-node coordination. Neither scales cleanly. In multi-node disaggregated serving, the problem compounds: prefix caches are distributed, routing decisions affect hit rates, and coordination without a clear authority creates either duplicated state or missed reuse.

---

## Goals

**Non-Intrusiveness:** The KV Cache Controller must not put blocking work on the engine's hot path or interfere with memory the framework uses for execution. Cache data that the framework depends on must retain a local, fast-access guarantee, whether the framework holds the bytes directly or delegates that guarantee to the cache layer. Framework-facing calls either read local metadata, enqueue work, or complete asynchronously, while internal caching remains transparent to the engine.

**Performance:** The KV Cache Controller should not add avoidable overhead to framework scheduling or GPU memory management. Its value comes from opening reuse and overlap opportunities outside the engine hot path: serving cache blocks from local or remote G2, G3, or G4 when cheaper than recompute, and using router hints for early staging before the framework needs the data. The primary value is the separation of dependencies and control paths; latency benefits come from using existing routing and scheduling windows for cache preparation.

**Scalability:** Only the KV router's global inventory should scale with the cluster. It receives partial, eventually-consistent information from each KV Cache Controller — enough to make good decisions without requiring a complete, strongly consistent global view. A controller's role and behavior are the same on a single node and in a thousand-node cluster; adding a node adds its inventory to the router without changing existing controllers.

**Extensibility:** Frameworks can plug in caching policies through the standard control-plane interface without knowing or modifying the underlying data-movement mechanism. Framework hints bridge the gap between scheduling context known to the engine and storage policy owned by the controller.

**Adoptability:** Frameworks can adopt the KV Cache Controller incrementally, with each integration level delivering independent value. Minimal integration improves routing through controller inventory, while deeper integration adds storage, tiering, and cross-node reuse. A framework that already uses a KV router can add the controller as an inventory source without replacing its routing or scheduling layers.

**Resilience through hot start and optional side process:** The normal KV Cache Controller remains in the inference-engine process for efficiency. Deployments that need controller-owned memory to survive an engine or GPU failure can place it in an optional side process that also hosts a backup controller. The active controller maps the pool directly; after fencing a failed owner, the backup can take ownership and serve committed KV data. A restarted engine hot-starts from preserved metadata and takes ownership back through a fenced handoff. This applies only to controller-owned memory, not framework-owned G1 or G2 memory.

---

# Proposal

## Principles

**Principle 1 — Non-Intrusiveness: The framework owns and orchestrates GPU memory; it delegates transfer execution to purpose-built tools.**

Adding GPU management to the KV Cache Controller introduces coordination on the hot path, shared GPU state between two entities, and the need to continuously synchronize scheduling context. The framework already has all the context needed: scheduling state, active requests, decode step count. Any external entity would need to be told this continuously.

The framework's orchestration role extends across both caching and disaggregated serving. For prefill/decode (P/D) transfers, it delegates the physical DMA to NIXL. For KV caching, it delegates host-side movement and cross-node transfers to the KV Cache Controller — providing GPU pointers when needed so the controller can execute GPU↔host DMA via NIXL on its behalf. Framework-owned memory movement and GPU scheduling remain framework decisions; the controller decides controller-owned tier movement in response to framework calls, router hints, and policy. GPU-bound bandwidth is not exclusively local: in disaggregated serving, remote writes into GPU memory are already in the picture. The invariant that matters is GPU memory ownership and scheduling, not who physically issues the DMA.

**Principle 2 — Scalability: separating cross-node coordination as a global inventory where controllers have no peer block inventory.**

Each KV Cache Controller manages only its own local G2 and G3 tiers and has no knowledge of peer block inventories. The KV router holds the cross-node mapping — kept as minimal as possible, keyed by `BlockKey` and optional prefix-path metadata — and provides request-scoped source hints. Controllers execute transfers directly via NIXL.

**Principle 3 — Separation of Control and Data Plane: Mechanism is stable, policy is pluggable.**

All host-side data movement — slot management, G3 writes, G4 operations, cross-node transfers — is the controller's data plane and does not change across deployments. The policy governing admission, eviction, placement, and early-staging prioritization is the control plane and is fully replaceable. Framework hints are the interface between the engine's scheduling context and the policy, carrying intent that access-pattern inference cannot provide.

**Principle 4 — Resilience: Controller-owned memory can outlive the in-process controller without moving the normal path out of process.**

The normal KV Cache Controller remains in-process and manages cache semantics over its attached pool. When engine-failure resilience is needed, an optional side process can own controller-owned memory and host a backup controller without changing the normal cache-management interface. Framework-owned memory remains owned by the framework and is outside this mechanism. Attachment, takeover, and hot-start details are defined in the Memory Management section.

**Principle 5 — Locality without extra hierarchy: same-node and cross-node reuse use the same source model.**

Consistent with Principle 2, this design uses the same router-directed source model for same-node and cross-node reuse. A shared node-level pool would add another ownership, coherence, isolation, and recovery boundary. Instead, the router identifies the source and NIXL selects the appropriate transport—for example, directly from one process's host memory into another process's GPU. Colocation changes the transport, not the ownership or coordination model.

---

## Component Roles

Figure 1 summarizes the component boundaries and an example destination-initiated peer transfer; dashed green outlines denote optional deployment components.

![KV Cache Controller component overview](./0016_images/component_overview.svg)

**Framework / Engine**
- Sole owner of GPU memory: all block allocation, eviction decisions, scheduling
- Decides which blocks enter or leave framework-owned memory and when; performs transfers directly or delegates to the controller for caching and cross-node serving
- During framework initialization, supplies the runtime-provided peer control endpoint address to the controller and registers the address with the router
- Registers its G1 and G2 memory with its own NIXL agent at init; shares the agent metadata blob with the local controller, which may further share it with peer controllers
- Generates the KV layout manifest needed to interpret cached data
- Reports framework-owned memory (`HBM` for G1 and `FW_DRAM` for G2) to the controller via `tier_enter`/`tier_exit`, or directly to the router when the integration already provides that path; reporting is batched and non-blocking
- Can provide a pinning interface to the controller, to share its G2 with other instances
- Handles P/D KV transfer between prefill and decode instances

**Optional memory-owner / backup process**
- Owns the controller-owned G2 (`DRAM`) allocation when engine-failure resilience is enabled
- Exposes the pool for direct access by the active controller
- Hosts a backup controller with its own NIXL agent; after fencing the failed active owner, the backup may serve already committed controller-owned KV data
- Preserves committed metadata for hot start and transfers ownership back to a restarted in-process controller through a fenced handoff
- Does not preserve framework-owned memory or decide framework scheduling, routing, or GPU allocation

**KV Cache Controller (host-only)**
- Main runtime component linked directly into the engine as an in-process library
- Manages controller-owned G2 and G3 residency, plus configured G4 object storage
- Reports controller-tier inventory to the router and, when framework inventory flows through the controller, forwards framework-reported G1/G2 working-set state with tier labels
- Executes all host-side data movement via NIXL: local tier transfers, cross-node transfers, direct deliveries, and G4 fetches
- Owns its NIXL agent for controller-owned memory; coordinates with the framework's separate NIXL agent for framework-owned sources and destinations
- **Transport abstraction:** Controller-issued data movement goes through NIXL, which provides a common interface across GPU-host DMA, NVMe, RDMA, and object-store access. This minimizes controller-logic changes across supported hardware configurations, although deployments still require backend-specific configuration, credentials, and capability handling.
- Enforces storage policy: admission, eviction, tier placement, early-staging prioritization
- Responds to framework queries about block availability
- If G4 is enabled, the controller can query or fetch from it directly; G4 is equidistant to all controllers, so it does not require a per-node ownership lookup
- Avoids blocking the engine's hot path

**KV Router**
- Maintains eventually-consistent global `BlockKey` inventory from all controllers, with tier labels and optional prefix-path metadata
- Makes cache-aware routing decisions based on KV overlap score and node load
- Sends hints to destination controllers — for same-node G3/G4 staging or with a source node and peer control endpoint for cross-node retrieval
- Optionally tracks G4 presence for early-staging hints; because G4 is equidistant, it does not create cache locality for any particular node
- Does not manage P/D KV transfer — that is the engine's concern

---

## Memory Management: Enabling Principles 1 and 4

The KV Cache Controller manages its attached G2 pool, moves KV data across tiers, reports inventory, and enforces policy. Controller-owned memory placement is NUMA-aware and should remain near the GPUs it serves when the topology permits. Deployments that do not need engine-failure preservation may allocate the pool in the inference-engine process. The cache API and policy semantics are the same either way; only the pool lifetime and recovery guarantee differ.

**Two mechanisms, any mix:**

`deposit`/`deliver` — framework initiates both; `deposit` pushes a block into the controller-owned pool (callback fires when the source pointer is safe to reuse) and may retain an evictable controller copy; `deliver` requests that the controller place a block into a framework-provided G1 or G2 destination (callback fires when the destination is ready); without `no_evict`, neither call leaves a framework-visible release handle or non-evictable claim open.

`fetch`/`release` — framework asks the controller to make a block resident in its controller-owned G2 pool and pin it there. On success, `fetch` returns the resident pointer and a release handle; the framework calls `deliver` to place the block into a framework-provided destination and calls `release` when the pool claim is no longer needed.

`deposit` also accepts a `no_evict` flag (list-level, applies to all entries in the batch): when set, the controller keeps every completed slot non-evictable and returns the pointer plus release handle per entry — for when the framework hands a batch of blocks to the controller for cache management and cross-node serving while retaining local references. The framework calls `release` with that handle to clear the no-evict claim. Any mix of the two mechanisms is valid; pool size and deposit/deliver ratio are deployment knobs, not protocol choices.

This is opt-in for frameworks. A framework that wants guaranteed local DRAM residency behavior for selected KV blocks can get that behavior through `no_evict`, while the controller still handles sharing, routing visibility, transfer setup, and tiering policy. The tradeoff is backpressure: if a framework overuses no-evict claims and the controller runs short on capacity, it must respond to `capacity_needed` by releasing enough claims to free the requested number of slots. If a deployment uses only framework-owned memory, it does not use `deposit`, `fetch`, or `release`; `search_and_pin`/`unpin` together with `deliver` is its single mechanism.

**NIXL agent ownership:**

NIXL agents follow memory ownership. The framework has its own agent for framework-owned GPU and host memory. The active in-process controller has a separate agent for controller-owned memory and uses the framework's agent for operations involving framework-owned memory. All controller data movement uses asynchronous NIXL operations. When resilience is enabled, the backup controller has its own agent and uses it only after a fenced takeover; only the active controller accepts new operations for the pool.

**Framework–KV Cache Controller Interface:**

The framework–KV Cache Controller boundary is a small set of non-blocking calls. Movement into or out of framework-owned memory belongs to the framework: it provides the source or destination descriptors, timing intent, and scheduling context. The controller may still stage, tier, retain, or move data among its own managed tiers according to policy and request-time router hints.

`KVLayoutManifest` identifies the framework, model, KV layout, and host representation needed to interpret cached data. `G2ResidentMap` records committed entries in the attached G2 pool for hot start. The pool may contain multiple internal pools to support different attention-head requirements. On restart, the controller validates pool and layout compatibility before reconstructing committed inventory; without a valid map, the pool is treated as a cold start.

Framework-owned G1 and G2 memory may be reported through the controller or directly to the router when an existing integration already provides that path. Framework-owned memory is sourceable through the `search_and_pin`/`unpin` adapter when the framework exposes a NIXL-accessible memory location. Controller-owned G2 is under controller control and requires no framework pinning API. The controller reports its managed G2, G3, G4, and other tiers to the router, along with any framework-owned residency reported through it.

```python
# Python-style pseudocode
# Framework → KV Cache Controller
kv_cache_controller.init(pools=[PoolSpec(...)], layout_manifest=layout_manifest, resident_map=None, peer_control_endpoint=None, config=None) # runtime supplies pools, peer endpoint, and optional hot-start map/config

kv_cache_controller.tier_enter(block_key_list, tier)                              # framework inventory add
kv_cache_controller.tier_exit(block_key_list, tier)                               # framework inventory remove

kv_cache_controller.deposit([(block_key, ptr, mem_type, hints=None), ...], no_evict=False, callback=None)  # completion value is None or (ptr, release_handle)
kv_cache_controller.query(block_key_list, request_id=None) -> List[Status]         # HIT/MISS/FETCHING/FETCHABLE
kv_cache_controller.fetch(block_key_list, request_id=None, hints=None, callback=None) -> OperationHandle # completion value is (ptr, release_handle)
kv_cache_controller.deliver([(block_key, dst_ptr, mem_type), ...], request_id=None, callback=None) -> OperationHandle # place blocks at framework destinations
kv_cache_controller.release(release_handle_list) -> List[Result[None, Error]]      # release fetch/no-evict claims

kv_cache_controller.poll_completed() -> List[Completion]                          # drain individual completions
kv_cache_controller.abort(operation_handle, block_key_list=None)                  # cancel this fetch/deliver operation, optionally for selected keys

# KV Cache Controller → Framework
framework.capacity_needed(num_slots)                               # last-resort capacity pressure signal

framework.search_and_pin(block_key_list) -> descriptors+pin_handle | UNAVAILABLE # one pin may cover the full list
framework.unpin(pin_handle)                                        # release a framework-owned source pin
```

**Operating flow:**

From the framework's perspective, a controller-owned memory entry is either under the controller's control (it may evict, move, or use the entry for internal staging) or protected by a claim. `fetch` returns a resident pointer and release handle; `deposit(no_evict=True)` likewise returns a protected controller pointer and release handle. Each successful handle must be released once. Stale or duplicate handles return an error without changing residency state.

`deposit` copies from framework-owned memory into the controller's pool, while `deliver` places data into a framework-provided destination. `deliver` does not name a source; source selection remains with the controller and router. The controller does not allocate or free framework memory. A `deposit` admission failure completes the affected entries with errors and emits no inventory update. Data-movement calls are non-blocking and complete through callbacks or polling. `query` only reports current knowledge and may enqueue a configured G4 existence check; it does not allocate memory, acquire pins, change residency, or start a transfer.

To serve data, the controller may use a claimed controller residency, framework-owned memory obtained through `search_and_pin`, or data staged from G3 or G4. A framework pin may cover multiple block keys and be shared by multiple operations. The controller reuses covered keys from previous requests and asks the framework to pin only the remaining keys. The framework must keep the pin valid until every dependent operation finishes and the controller calls `unpin`.

When an operation times out or is cancelled, safe release starts immediately. Caller-visible completion may precede physical release, but NIXL-visible descriptors and framework-owned pins remain protected until NIXL can no longer access them. Expired pins are not reused for new work; any still-needed keys may be acquired again after release is safe. Delays beyond the deadline therefore come from NIXL quiescence or safe release of framework-owned memory, not controller scheduling.

`fetch`, `deposit`, and `deliver` accept lists and report per-entry outcomes. Batch callbacks fire when the batch is complete, while `poll_completed` exposes individual completions as they arrive. The framework must call `release` for every successful release handle it consumes.

`abort` is best effort and may target selected entries. Shared acquisition or transfer work continues while other operations still depend on it, and completed claims still require `release`. `deposit` is not normally cancelled through `abort`. If capacity cannot be recovered through policy, the controller completes committed work with an explicit error and may use `capacity_needed` as a last-resort pressure signal.

Hints, including framework `retention_hint` and router `no_retain`, are advisory inputs to scheduling and policy. They cannot override a `no_evict` claim or silently discard committed `fetch`, `deliver`, or `deposit` work. For example, a `fetch` deadline can tell the controller to report an error early when the block is unlikely to arrive within the framework's scheduling budget.

The framework can retrieve data in two steps (`fetch` followed by `deliver`) or request direct `deliver` into a framework-provided destination. Cross-node movement is destination-initiated. Stale router metadata or an unavailable source causes the operation to fail cleanly, allowing framework recomputation and an opportunistic inventory refresh.

**Side-process resilience:**

Without side-process resilience, the inference-engine process allocates and owns the controller-owned pool itself. With resilience enabled, an optional per-instance side process instead allocates and owns that pool, preserves its committed metadata, and hosts a backup controller. The active controller still runs inside the inference-engine process. It maps the side-process-owned pool through shared memory, `mmap`, or an equivalent mechanism and reads or writes the bytes directly; normal cache operations do not make an IPC round trip through the side process.

If the engine or GPU fails, the side process first fences the failed owner and then activates its backup controller. The backup can serve committed KV data from the preserved pool. When a replacement inference-engine instance starts, its in-process controller maps the preserved pool, reconstructs committed residencies from the preserved metadata, resynchronizes inventory, and becomes active through a fenced handoff. Within a compatible framework/model/layout, the controller stores a tensor-parallelism (TP)-independent host representation. This is not a universal cross-framework KV format. It is the framework-declared offload/cache layout, with TP partitions packed in a stable way when needed. Partial writes and in-flight operations are not recovered. Framework-owned G1 and G2 memory are not part of this mechanism. Recovery and handoff must preserve committed-data integrity and prevent concurrent ownership.

---

## Router–KV Cache Controller Dynamics: Enabling Principles 2 and 5

The router maintains an eventually-consistent global prefix inventory for cache-aware request routing. That same inventory directs cross-controller transfers through request-scoped source hints sent to the destination controller before or during request setup. The router is never on the data path: controllers execute transfers peer-to-peer via NIXL. Source endpoints remain request-scoped rather than becoming part of global block state, so this coordination adds no controller-side peer inventory. The controller does not query the router during framework lookup, and the incremental load on the router is small because it already tracks who has what.

Cross-controller block transfers require a metadata layer: given a `BlockKeyList`, which node holds the requested blocks? A `BlockKey` identifies one cacheable KV unit in the public controller key space: usually one logical KV block/page, but it may also name a slot bundle or a layout-specific cache key when the framework groups multiple physical components under one cache identity. A `BlockKeyList` is an ordered exact path for a request prefix, such as `[k1, ..., k10]`; `k10` alone is not a request for all prior blocks, and `[k5, k10]` is an exact sparse request, not a longest-prefix lookup. The router supplies this metadata.

Same-node reuse follows the same rule as cross-node reuse: the router identifies a source and the destination controller initiates the transfer. The readable source may be controller-owned memory, framework-owned memory protected by a pin, or storage data staged by the source controller. Local and remote reuse therefore have the same delivery semantics without introducing a shared node-level pool with separate ownership, coherence, isolation, and policy concerns. Such a pool would also need compatibility boundaries across tenants, models, inference engines, and KV layouts.

For bounded shared inputs such as repository or harness context, the router can choose one controller as the preferred owner for the node and route later consumers to it. Consumers may retain local copies when replication is worthwhile. Otherwise, the router includes `no_retain` in `submit_hint`. The consuming controller still satisfies any committed framework request; `no_retain` only advises it not to keep an additional controller-owned copy after the framework no longer needs it.

Router-directed copy or move rebalancing based on a `utilization_update` from the controller, as well as explicit controller requests for router guidance, may be added in the future if needed; neither API is part of the initial design.

**Router API:**

The KV Cache Controller uses the router interface for inventory and request-scoped source hints. Inventory updates and heartbeats reuse the existing router control path. Request-time `submit_hint` may reach the controller directly or be forwarded through the framework, depending on the integration. Controller-to-controller transfer control uses a separate peer control channel from the data path. The runtime supplies the peer control endpoint to `kv_cache_controller.init`. During framework initialization, the framework also registers that endpoint with the router alongside the node metadata; the router includes it with the source node in request-scoped hints.

Inventory events use the same `BlockKey` values as fetch and `submit_hint` in the normal case. `BlockInventoryEntry` carries the block key, tier, and optional relationship metadata; optional fields let the router maintain a prefix index without adding controller-side lookup semantics. If a deployment temporarily has different router-side and framework/controller-side hashes, the framework reports both to the router in inventory metadata. The router indexes by its routing key, but includes the controller-local key in source hints so the controller that owns the local record can resolve it without recomputing or guessing the other hash. A route-time `submit_hint` may carry a whole `BlockKeyList` from one source. In the future, if necessary, multi-source assembly can be added by splitting the list into multiple hinted operations.

```python
# Python-style pseudocode
class BlockInventoryEntry:
    block_key: BlockKey                   # router KV block/page identity
    tier: str                             # G1 HBM, G2 FW_DRAM/DRAM, G3 SSD, G4, ...
    parent_key: BlockKey | None = None    # previous block key in the same prefix path
    scope: Scope | None = None            # model/layout/tenant/key-space namespace
    local_key: BlockKey | None = None     # optional framework/controller-local key when it differs from block_key

# Framework → Router during framework initialization
router.node_register(node_id, peer_control_endpoint, capacity)

# KV Cache Controller → Router
router.kv_tier_add(entries: List[BlockInventoryEntry], gen)    # inventory add
router.kv_tier_remove(entries: List[BlockInventoryEntry], gen) # inventory remove
router.heartbeat(node_id)                                      # liveness signal

# Router → KV Cache Controller, directly or through the framework
kv_cache_controller.submit_hint(request_id, block_key_list, src=None, mode=copy|move, hints=None) # src includes node and peer control endpoint; default mode=copy
```

Inventory is eventually consistent and may be batched. Updates are tier-specific because one block may be present in several tiers at once. Inventory generations expose missed updates; after reconnect or hot start, the controller republishes its current inventory to the router. `heartbeat` reports liveness. `submit_hint` is advisory and may start movement when policy and capacity allow; framework `fetch` and `deliver` are committed calls that must complete or return an error.

The initial design does not prescribe a transfer-versus-recompute cost model. Router staging hints are advisory and may be declined by controller policy. The framework decides whether to issue a committed `fetch` or `deliver` request or recompute based on its scheduling budget. More sophisticated cost policies may be added later.

Controller-to-controller movement is destination-initiated. A router hint gives the destination the source node and peer control endpoint. `mode=copy` retains the source copy, while `mode=move` permits source eviction only after successful completion. Timeout or cancellation keeps the source copy. When the router sends `src=None`, the destination uses its local storage information or checks G4 when enabled. It may stage the result in controller-owned memory or deliver directly into framework memory when the transport supports it.

**Controller-to-controller peer communication:**

Peer controllers communicate over two separate paths. A peer-to-peer control channel establishes the connection and exchanges NIXL metadata, acknowledgements, and transfer-control messages; it never carries KV payload bytes. The control endpoint is separate from the NIXL agent identity. The data transfer is direct peer-to-peer through asynchronous NIXL WRITE operations and does not pass through the router or control channel. As shown in Figure 1, either endpoint in a peer transfer may use controller-owned or framework-owned memory through its owning NIXL agent.

Eager connection and early remote pinning are optional configuration choices, not protocol requirements. Controllers may remember established peer connections to avoid repeated setup. The control protocol must define compatibility and versioning so incompatible or late messages fail explicitly.

**Object store (G4) and key mapping:**

G4 is equidistant to all controllers and is not a node-local ownership tier. A controller can fetch from G4 directly when enabled. The router does not need G4 inventory for locality decisions, but may track G4 presence to issue early staging hints. Object identity should be deterministic from the block key or maintained by the router rather than duplicated across controllers.

**Block identity and key authority:**

The router is the routing-key authority: it assigns the request's content-derived `BlockKey` path before execution, and the framework normally passes those keys unchanged to the controller. During a migration with distinct local hashes, the explicit `local_key` mapping described above preserves that authority without requiring the controller to translate keys. The key space may be namespaced by model version, quantization, KV layout, tenant, or hash algorithm; the controller treats each full key as opaque. Blocks are reusable only when their model, adapter, KV representation, and cache-sharing boundaries are compatible.

---

## KV Cache Controller Policy and Interface: Enabling Principle 3

The data plane (memory management, NIXL transfers, inventory reporting) is fixed. The policy is replaceable without touching the mechanism. Policy calls must complete quickly and never block.

Policy scope is local to controller-owned G2 and downstream tiers. It does not schedule framework-owned G1 or G2 memory, choose request placement, or decide global source selection; the router supplies source hints, and controller policy decides whether to stage, retain, move, or drop within the controller. A committed framework `fetch` or `deliver` cannot be silently dropped by policy; it either completes or returns an error. Fetch and `no_evict` deposit claims are hard mechanism constraints: while they exist, policy cannot evict the entry. `no_retain` is a router retention hint; if it conflicts with a framework `no_evict` claim, `no_evict` wins until the framework calls `release` with the returned handle, after which `no_retain` takes effect and the controller should not retain another pool copy.

**Policy interface:**

```python
# Python-style pseudocode
class KVCachePolicy:
    def prioritize(self, queue: EvictQueue, src: Tier) -> None: ...  # score evictable entries
    def decide(self, meta: BlockMeta, src: Tier) -> Tier: ...       # choose downstream controller tier or retention target
    def on_ingest(self, meta: BlockMeta) -> IngestAction:           # placement after deposit
        return IngestAction.KEEP
    def on_remove(self, block_key: BlockKey) -> None: ...           # called when block is permanently gone

class Tier:  # policy-visible choices; framework-owned G1 and G2 are not policy-controlled
    DRAM = "dram"  # controller-owned G2
    SSD = "ssd"    # G3
    G4 = "g4"
    REMOTE_CONTROLLER = "remote_controller"  # source only
    DROP = "drop"  # do not retain or admit into a controller-managed tier

class IngestAction:
    KEEP = "keep"  # retain in DRAM; eviction path decides downstream tiers (default)
    DROP = "drop"  # discard immediately without caching

def copy_to(tier: Tier) -> IngestAction: ...  # also write now and retain the DRAM copy
def move_to(tier: Tier) -> IngestAction: ...  # write now and free DRAM after completion

# Policy can read queue entries and update scores only; content is read-only.
class EvictQueue:
    def __iter__(self) -> Iterator[tuple[SlotId, BlockMeta, Score]]: ...
    def update_score(self, slot_id: SlotId, score: Score) -> None: ...
```

`BlockMeta` is owned by the mechanism and passed read-only to the policy:

```python
# Python-style pseudocode
class BlockMeta:
    block_key: BlockKey
    size: int
    access_count: int       # incremented by mechanism on cache accesses
    last_access: Instant    # updated by mechanism on each access
    hints: list[HintType]   # framework hints plus router hints such as NO_RETAIN
    eviction_blocked: bool  # fetch or no_evict deposit claim is active
```

`BlockMeta` provides the common access information and hints needed by policy. Custom hints come from framework calls, while router hints such as `NO_RETAIN` arrive through `submit_hint`. The policy receives the controller-managed tiers enabled for the deployment and may return only one of them or `DROP`; `REMOTE_CONTROLLER` may appear only as a source. Framework-owned destinations are supplied separately by the framework and are not policy return values.

`decide(meta, src)` is constrained by the source tier: local memory may move to a lower controller tier, storage may move farther downstream or be dropped, and remote data may be admitted to controller memory or skipped. A direct cross-node `deliver` may still target framework-owned memory without treating it as a controller cache tier. Policy may decline advisory hints, but committed framework requests must complete or return an explicit error.

**Proactive eviction:** The controller invokes `prioritize` before staging capacity is exhausted. Only ready, unclaimed controller-owned residencies are eligible; framework-owned memory and storage locations are outside this eviction queue.

**Ingest-time decisions:** `on_ingest` chooses whether newly deposited data remains in controller-owned G2, is copied or moved to another controller tier, or is not retained. Active claims take precedence over policy decisions that would remove the data.

**Policy expressiveness:** The four methods cover admission and ingest-time placement (`on_ingest`), eviction ordering (`prioritize`), downstream tier routing (`decide`), and stateful cleanup (`on_remove`). Policies may use access history, tenant priority, or prefix relationships supplied through framework hints without changing the data plane.

**Block availability query from framework:**

When the framework asks the KV Cache Controller whether a block is available, the controller responds with one of:
- `HIT(tier, location)` — locally available at the time of the query
- `FETCHING(estimated_time)` — fetch in progress (from G3, G4, or via a cross-node transfer); estimated time is optional
- `FETCHABLE(tier, estimated_time)` — not started, but the controller knows the location; `tier` distinguishes `REMOTE_CONTROLLER`, `LOCAL_SSD`, and `G4` in that order of expected latency; estimated time is optional
- `MISS` — not found in any tier the controller currently knows about; may enqueue a background G4 check when configured

These statuses describe current controller knowledge, not a reservation or guarantee. A local G2 `HIT` may be evicted before a later operation claims it, and a remote G2 `FETCHABLE` result based on a router hint remains provisional until the operation succeeds. Framework-owned `FW_DRAM` can be returned as sourceable when the router hint identifies it and the framework pin path is available; the actual readability is confirmed by `search_and_pin` before transfer. Framework-owned `HBM` follows the same design if a deployment exposes GPU KV through the pin API, but that is not an initial target. To obtain the data, the framework calls `fetch` or `deliver` and waits for completion; either operation may return an error.

`query` is fast and non-blocking because it reads local controller knowledge rather than calling the router. Remote source information arrives through `submit_hint`; optional G4 discovery may update the result of a later query. After a `MISS`, the framework may issue a follow-up `query` when its scheduling budget allows; no separate asynchronous query callback is required.

---

## State Model and Operation Lifecycle

The preceding memory, routing, and policy flows share a node-local state and operation model. Its central lookup structure is the `block_index`, which maps each `BlockKey` to a `BlockRecord`. The `BlockRecord` is the authority for that block's known locations and residency state. A block may be present in several locations at once, for example controller-owned G2 and G3, or controller-owned and framework-owned memory while a transfer is in flight. Because this map contains only local state, it does not grow with the cluster; the router owns the global inventory.

Framework-owned memory and controller-owned memory use different residency types. Framework residency carries the framework descriptor and pin. Controller residency carries the controller allocation, readiness, claims, and eviction state. Storage tiers likewise use location types suited to their own behavior. Remote endpoints and request-scoped hints belong to operation state rather than the block record.

Residency state and operation state are deliberately disjoint. A `BlockRecord` does not embed or own operation lifecycle state; it only keeps references to related operation IDs when needed. The separate `in_flight_ops` collection owns those operations and their lifecycle. This lets block lookup remain stable while operations are created, coalesced, completed, cancelled, or removed independently.

**Controller tracking objects:**
- `block_index`: the authoritative node-local map from block keys to `BlockRecord` residency state.
- `in_flight_ops`: a separate polymorphic collection for every operation kind. All operations share identity, lifecycle state, deadline, related blocks, and completion information. Each operation implementation defines how success, failure, timeout, cancellation, and completion affect its resources; block records hold only references to related operation IDs.
- `evictable_priority_queue`: tracks controller-owned residencies that policy may evict; insertion order is the default priority until policy updates it.
- `free_list`: available controller-owned memory slots.

These four structures are the primary state model. Auxiliary structures may support completion polling, pins that cover multiple keys, and reuse of established peer connections, but do not replace the primary model.

Claims are attached to concrete residencies. They prevent controller-owned memory from being evicted while in use and keep framework-owned memory pinned until all dependent operations finish. Concurrent operations may share existing residencies, pins, or in-flight work; cancellation of one caller must not invalidate resources still needed by another. When an operation needs a controller residency that is currently evictable, the controller removes it from the evictable queue and acquires a claim before using it, making it non-evictable. After the last claim is released, a ready residency that is still retained returns to the evictable queue; otherwise its controller-owned memory is freed.

The controller does not maintain another per-block state machine alongside the authoritative `BlockRecord`. Transient operations remain separate and are connected to a record only by their references. The event loop owns controller metadata mutations and never performs blocking external calls. Dedicated NIXL I/O and progress threads submit and poll transfers, then post completions back to the event loop; framework-agent and storage work follow the same asynchronous completion pattern.

### Telemetry

Controller-level telemetry covers:

- operation outcomes and end-to-end latency;
- control-channel setup latency, message counts, acknowledgements, and failures;
- framework pin acquisition latency and acquisition/unpin outcomes;
- NIXL setup and submission latency, transfer time, notification, and failure;
- blocks and bytes transferred by bounded source and destination tier;
- failures such as unavailable sources, timeouts, capacity errors, and incompatible peer metadata; and
- current resource counts, including known blocks, in-flight operations, pins, claims, peer connections, and available capacity.

Telemetry is optional and cheap when disabled. Labels use bounded categories rather than block keys, request IDs, or raw endpoints.
