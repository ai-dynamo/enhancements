# Pluggable Pool Discovery for the Global Router

**Status**: Draft

**Authors**: Anna Tchernych

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**:

**Required Reviewers**: Sachal Malick, Neelay Shah

**Review Date**:

**Pull Request**:

**Implementation PR / Tracking Issue**:

# Summary

The Global Router needs to know which pools exist and how to reach them.
This DEP proposes ways to do that. This covers **membership only**: which pools exist, and their addresses.
It does not cover how KV and load state flows, or how requests reach a pool.
The DEP proposes an abstract interface covering 3 implementations.
The deployer can choose: 
1. The gRPC native approach when each cluster relay calls `/RegisterPool` API and does not use any Kubernetes APIs.
2. A Hybrid Approach when each cluster Relay calls `/RegisterPool` API. The Global Router stores the list of pools in its own Cluster's Kubernetes API. 
3. A full Kubernetes approach aimed at deployments where the worload clusters can safely call into the Hub's Kubernets API server.

This DEP is part of the "K8s Native Global Router Solution" LLD listed in
the
[Global Router HLD](https://docs.google.com/document/d/1FYKvlsEnc6aMXgU_61RwtUJP3LWY7sQIEhOXjCFM2sk/edit?tab=t.bjkujx8pnylf).


# Motivation

The HLD's "Global View" section says that each pool has a **PoolRelay**.
The PoolRelay registers its pool with the Global Router with a gRPC call
(`RegisterPool`), and then sends heartbeats. Pools dial out to the
Global Router, so workload clusters do not need an open inbound port for
registration.

The HLD leaves these questions open:

* Open question 1: push, pull, or discovery for registration?
* Open question 9: how does a PoolRelay learn the Global Router replica addresses?

It also does not say how any of this maps to Kubernetes.

Today's Global Router POC
([#15338](https://github.com/ai-dynamo/dynamo/pull/15338)) reads a
static JSON list of pools at startup, and the router connects to each
Relay. In this DEP's terms, that is the static file source
(`FileMembership`). The interface below lets the other options replace
it without changing the router.

The hard part of question 9 is that the HLD wants every Global Router
replica to see every pool, with no replica-to-replica syncing of pool
state. Say there are 3 router replicas behind one load balancer:

* A PoolRelay calls `RegisterPool` on the load balancer. The call lands
  on one replica. The other two never hear about the pool.
* To reach all three, the PoolRelay needs each replica's own address.
* When the router scales to 4 replicas, every PoolRelay in every cluster
  must notice replica 4 and register with it too.
* Replica 4 starts empty. It cannot tell when it is ready, because it
  does not know how many pools exist. Has it heard from 4 of 4 pools, or
  4 of 20?

## Goals

* The same Global Router code works with every discovery option.
* The deployer picks the option that fits their setup.

The options are compared against these properties:

* A pool can join and leave without restarting the Global Router.
* Pools dial out.
* Every Global Router replica sees every pool.
* A new replica knows the full list of pools, so it knows when it is
  ready.
* A pool can only register or change its own record.
* Align with Kubernetes multi-cluster standards where possible.

Not every option has every property. For example, gRPC Native needs
extra work so that every replica sees every pool, and a new replica can
only estimate when it is ready. See
[Comparing the Three Options](#comparing-the-three-options).

### Non Goals

* How KV and load state flows from a pool to the Global Router.
* How requests reach a pool (network reachability, tunnels).
* Requiring GAIE or a Gateway with the Inference Extension. This DEP
  uses only Dynamo CRDs and built-in Kubernetes APIs. When GAIE is
  installed, it can be used as an extra input.

# Proposal

## Overview
The proposal aims to define abstractions for discovery so that a client can decide. 

## Comparing the Three Options

| | gRPC Native | Hybrid | Full Kubernetes |
|---|---|---|---|
| Works outside Kubernetes | **Yes** | Pools: yes. The Global Router must run on Kubernetes | No |
| Matches the HLD wording | **Yes** | Partly. Same `RegisterPool`, but records are stored in etcd | Bends the "no external store" rule (see below) |
| Pool reaches every router replica (open question 9) | Needs extra work. The reply to `RegisterPool` returns the replica list, and pools connect to each replica | **Yes.** One regional address. Every replica watches | **Yes.** Write once. Every replica watches |
| New replica knows which pools to wait for | No exact answer. Ask a peer replica, or wait for a timeout | **Yes.** It lists the records | **Yes.** It lists the records |
| Identity, and one pool cannot touch another | We build it: mTLS, pool binding, revocation | We build it: mTLS in the Global Router | **Built in:** RBAC, one namespace per pool, CSR, audit log. |
| See which pools exist | We build `GET /pools` | `kubectl get dynamopoolexports -A` | `kubectl get dynamopoolexports -A` |
| Standards | None | Kubernetes store, custom registration | **Same pattern** as GAIE 1374 Push/Pull, Open Cluster Management, and Karmada pull mode |
| What workload clusters must reach | Every router replica's gRPC address | The regional gRPC address | **The hub's Kubernetes API server.** Some teams will not allow this |
| Extra infrastructure | An address per replica (a load balancer per replica, or a router like `stargate-k8s-router`), and a certificate authority for mTLS | A certificate authority for mTLS | Hub API server reachable from workload clusters, and credentials per pool |
| Parts to build | RPCs, heartbeat, replica list, pool-side connection to every replica, mTLS | RPCs, heartbeat, mTLS, the router writes records and Leases, watch | CRD, export agent in the operator, credentials, Leases, watch |
| Time to notice a dead pool | Fast. Missed heartbeats | Set by the Lease length. The state stream covers this | Slow, about a minute. The state stream covers this |
| Hub API server is down | Not affected | Known pools keep working from the watch cache. New pools cannot join | Known pools keep working from the watch cache. New pools cannot join |

## Which Option When

| Setup | Pick |
|---|---|
| No Kubernetes, or the Global Router does not run on Kubernetes | **gRPC Native** |
| The Global Router runs on Kubernetes, but workload clusters must not reach the hub's Kubernetes API (different owners, several clouds, strict security rules) | **Hybrid** |
| One owner and a private network between clusters, or a cluster manager (OCM, Karmada, a cloud fleet service) already connects clusters to a hub | **Full Kubernetes** |
| First tests, or a few pools that rarely change | **Static file** as a start |

## Membership Interface

We will abstract pool membership behind an interface, so that all three
options work with the same Global Router code:

* gRPC Native: gRPC `RegisterPool`, as in the HLD
* Full Kubernetes
* Hybrid: gRPC registration, stored in the hub's Kubernetes API

Membership is split into two questions:

1. **How does a pool announce itself?** (the write side)
2. **Where does the pool list live, and how does every replica read
   it?** (the store and read side)

Each option is one combination of the two.

| Option | How pools announce | Where the list lives | Who sees it |
|---|---|---|---|
| gRPC Native | PoolRelay calls `RegisterPool` | In memory, per replica | Only the replica that got the call |
| Full Kubernetes | Export agent writes a record and Lease to the hub | Hub Kubernetes API | Every replica, by watching |
| Hybrid | PoolRelay calls `RegisterPool` | The receiving replica writes to the hub Kubernetes API | Every replica, by watching |
| Static file (possible start) | A person edits the file | ConfigMap | Every replica |

The interface itself is described in
[Implementing the Interface](#implementing-the-interface), after the
three approaches.

## Hybrid Approach (gRPC + Kubernetes)

Pools keep talking gRPC. The Global Router uses Kubernetes behind the
scenes to share what it hears. The discovery information is stored in the Hub's Kubernetes API.

1. A pool's PoolRelay calls `RegisterPool` on the region's single
   Global Router address.
2. The load balancer sends the call to one replica.
3. That replica writes a small record for the pool into the Hub Cluster's 
   Kubernetes API Server.
4. Every replica watches those records, so every replica sees every
   pool, even though the pool talked to only one of them.
5. A new replica reads all the records at startup and knows which pools
   to wait for.

**Pros:**

* Pools need only one address, the regional one.
* Every replica sees every pool, and a new replica knows when it is
  ready.
* Workload clusters never reach the hub's Kubernetes API. Only the
  Global Router does, inside its own cluster.
* A small change to the HLD design: the same `RegisterPool` call.

Compared with gRPC Native, the shared list of pools saves us:

1. **Pools do not have to find every replica.** With gRPC Native, a
   pool's call reaches only one replica, so every pool needs every
   replica's address and a connection to each one. With Hybrid, a pool
   calls one regional address, once.
2. **No extra network setup per replica.** gRPC Native needs a load
   balancer per replica, or a router like `stargate-k8s-router`. Hybrid
   needs one normal load balancer.
3. **A new replica knows when it is ready.** With gRPC Native it can only
   guess, with a timeout or by asking another replica. With Hybrid it
   reads the shared list and knows exactly which pools to wait for.
4. **The pool list survives restarts.** With gRPC Native, the list lives
   only in router memory and is lost if all replicas restart together.
   With Hybrid, it stays in the hub's Kubernetes API.

**Cons:**

* Kubernetes RBAC does not protect each pool's record, because the
  router writes it. The Global Router must check who is calling, for
  example with mTLS, and needs a certificate authority for the Relays.
* The Global Router must run on Kubernetes and needs permission to
  write records and Leases in its own cluster.
* Records are stored in etcd, so it bends the HLD's "no external store"
  rule, like Full Kubernetes.

## gRPC Native Approach

1. A pool's PoolRelay calls `RegisterPool` on the region's single
   Global Router address, as the HLD describes.
2. The discovery information is kept in the Global Router's memory

**Pros:**

* Can be deployed without Kubernetes
* On Kubernetes, the workload clusters never reach the hub's Kubernetes API. 


**Cons:**

* The biggest issue with the gRPC Native approach is the absence of the shared list of pools. 
Each router replica only knows the pools that happened to call it. Two problems follow:                                                                       
  1. Every replica must see every pool. A registration through the load balancer reaches only one replica. To solve this the `/RegisterPool` can return the list of replicas. The PoolRelay would connect to each replica to refresh its list.                                                                  
  2. A new replica can't know when it's ready, because it doesn't know how many pools exist. A new replica only learns about the pool as the find it and call in. So it does not know how many to expect to mark itself ready. If HLD does not want the replica-to-replica communication to solve this problem then gRPC can't give an exact answer. If the replica requests a snapshot then it works unless all replicas restart at once. 
  3. Extra infra is required. We need a load balancer per Global Router replica or a router that sends each connection to a named replica like stargate-k8-router. 
* We need to re-implement some machinery Kubernetes gives us for free. For example, Kubernetes RBAC does not protect each pool's record. The Global Router must check who is calling, for example with mTLS.
* Teams where security is not an issue may be more comfortable reusing existing solutions. 


## Full Kubernetes Approach
Each pool writes a small record into the Kubernetes API of the hub cluster where the Global Router resides. Every Global
Router replica watches those records. The hub cluster's Kubernetes API is the directory. Pools write to it.
Global Router replicas read from it.

```
 Workload cluster A              Hub cluster                    
 +--------------------+          +------------------------------+
 | Dynamo operator    |  dials   | Kubernetes API server        |
 | (export agent)     |--------->|  ns: pool-a                  |
 +--------------------+   out    |    DynamoPoolExport "pool-a" |
                                 |    Lease "pool-a"            |
 Workload cluster B              |  ns: pool-b                  |
 +--------------------+  dials   |    DynamoPoolExport "pool-b" |
 | Dynamo operator    |--------->|    Lease "pool-b"            |
 | (export agent)     |   out    |                              |
 +--------------------+          |        ^ watch   ^ watch     |
                                 |        |         |           |
                                 |  Global Router  Global Router|
                                 |  replica 1      replica 2 ...|
                                 +------------------------------+
```

The biggest con of this approach is security. The workload clusters can reach the Hub's cluster Kubernetes API server.
The biggest pro is relying on existing APIs, and reliable way to report the Global Router Replica's readiness as well as robust discovery which we do not need to implement ourselves. 

#### How This Answers HLD Open Question 9

* The export agent needs **one** address: the hub's API server. It does
  not know or care how many router replicas exist.
* The pool writes its record **once**. Every replica sees it by watching.
* When the router scales from 3 to 4 replicas, nothing changes for the
  pools.
* A new replica lists the records at startup and knows the full set, for
  example N pools. It becomes ready when it has state for all N.
  Readiness is well defined.

If the state stream also dials out (pool to router), the pools need the
router replica addresses. The same directory works in that direction too:
each replica writes its address to the hub, and pools watch that list.
The design of the state stream is out of scope here.

#### Kubernetes Native Approach makes the items below easy

1. **Every replica sees every pool, for free.** A watch gives every
   replica the same list. There is no replica list to hand out and no
   replica-to-replica sync.
2. **A new replica knows what to wait for.** It lists the records at
   startup, so "ready" has a clear meaning.
3. **Security comes built in.** Kubernetes already has identity, RBAC,
   namespaces, certificate signing, and an audit log. With `RegisterPool`
   we would build and maintain all of that.
4. **It follows a known pattern.** GAIE 1374 ("Push/Pull"), Open Cluster
   Management, and Karmada pull mode all have members dial out and write
   their own record to a hub. GAIE 1374 describes it as "Typical when you
   want no hub-stored member credentials."
5. **It is easy to operate.** Admins use `kubectl` to see, debug, and
   remove pools.



#### Components to write
1. Kubernetes Reconciler = Export Agent. Sees a pool, write its record and Lease to the hub. Deletes the record when a pool goes away. 
The agent watches DGDs. If GAIE is installed, it can also watch InferencePools.
   * The agent needs **two** Kubernetes clients: one for its own cluster
     and one for the hub. Today the operator talks to one cluster only.
     controller-runtime supports a second cluster client.
   * Only **one** operator replica may write the record and renew the
     Lease. The operator already uses leader election, so the agent runs
     only on the leader.
2. The agent will write a new CRD "DynamoPoolExport" installed in the Hub Cluster. It will use the existing K8 Lease (coordination.k8s.io/v1)
We can avoid the CRD in favor of ConfigMap but the CRD is preferable.     
3. The global Router needs a watcher for leases (watch, lease K8 APIs). It needs to build the Pool Catalog. 
4. Helm Chart to create a namespace, ServiceAccount and RBAC per pool.
5. Optional: if GAIE is installed, the pool can also be exported through its InferencePool.
6. The agent reads the pool's Frontend Service (built in) to find the pool's address.

#### Kubernetes APIs Used

* Workload cluster, the agent reads: `DynamoGraphDeployment`, `Service`.
* Hub, the agent writes: `DynamoPoolExport`, `Lease`. In Phase 2 also
  `CertificateSigningRequest`.
* Hub, the Global Router reads: `DynamoPoolExport`, `Lease`.
* GAIE and Gateway API: optional only.


#### Steps

1. **The user marks a pool for export.** In the workload cluster, the
   user marks the **DGD** for export, with a field or annotation on the
   DGD. If GAIE is installed, the standard GAIE export annotation on the
   InferencePool (`inference.networking.x-k8s.io/export: "ClusterSet"`,
   from GAIE's multi-cluster proposal
   [1374](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/1374-multi-cluster-inference))
   can be a second trigger.

2. **The export agent dials out.** A new reconciler in the Dynamo
   operator in the workload cluster sees the marked DGD. It connects to
   the hub's API server and writes a `DynamoPoolExport` record into the
   pool's own namespace on the hub.

3. **The export agent keeps a heartbeat.** It renews a Kubernetes
   `Lease` in the same namespace. If the Lease expires, the pool leaves
   the catalog. Similar systems renew about every 40 to 60 seconds.

4. **The Global Router watches.** Every replica watches all
   `DynamoPoolExport` records and Leases, and builds its Pool Catalog from
   them. The replicas do not talk to each other about pools.

5. **The pool leaves.** When export is turned off on the DGD, or the DGD
   is deleted, the export agent deletes the record. If the agent dies, the
   Lease expires and the pool is removed.

#### The Record

A small, namespaced custom resource on the hub. It holds data that
changes rarely.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoPoolExport
metadata:
  name: pool-a
  namespace: pool-a            # one namespace per pool
spec:
  poolId: pool-a
  frontendAddress: https://pool-a.us-east-1.example.com:443  # for requests
  relayIdentity: spiffe://example.com/relay/pool-a            # for the state stream
  location:
    region: us-east-1
    dc: dc-1
    cluster: cluster-a
  hardware: B200-IB
  models:
    - Qwen/Qwen3-32B
  workerCount: 16
```

The field list is a starting point for review.

#### Proposed shape for the DynamoPoolExport CRD

`kind: DynamoPoolExport`, one per pool, in that pool's namespace on the
hub. Main fields only:

| Field | Required | Example | Where the agent gets it | Why the Global Router needs it |
|---|---|---|---|---|
| `spec.poolId` | Yes | `pool-a` | DGD name (or a DGD field) | Names the pool. Matches the pool to its Lease and its state stream |
| `spec.frontendAddress` | Yes | `https://pool-a.us-east-1.example.com:443` | A manual override on the DGD first. Otherwise the Frontend Service's `status.loadBalancer.ingress`. Otherwise, if GAIE is installed, the Gateway's `status.addresses` | The pool's entry point. Where to send requests |
| `spec.relayIdentity` | Yes | `spiffe://example.com/relay/pool-a` | Relay configuration | Checks that a state stream really comes from this pool |
| `spec.location.region` | Yes | `us-east-1` | Agent configuration | Locality in cost functions |
| `spec.location.cluster` | Yes | `cluster-a` | Agent configuration | Locality, and to group pools by cluster |
| `spec.location.dc` | No | `dc-1` | Agent configuration | Locality |
| `spec.models` | Yes | `[Qwen/Qwen3-32B]` | DGD | Which pools can serve a request |
| `spec.hardware` | No | `B200-IB` | DGD or node labels | Answers "all InfiniBand pools" (HLD) |
| `spec.workerCount` | No | `16` | DGD | Rough pool size |

No `status` is needed. The Global Router only reads these records.

**The Lease.** The agent writes a standard `Lease`
(`coordination.k8s.io/v1`) with the **same name and namespace** as the
record. It sets `spec.holderIdentity` to the agent's ID and
`spec.leaseDurationSeconds` (for example `60`), and updates
`spec.renewTime` on each heartbeat. The Lease has an `ownerReference`
to its `DynamoPoolExport`, so deleting the record also deletes the
Lease.

#### When Is a Pool Routable

A pool is routable only when all of these are true:

* its `DynamoPoolExport` record exists,
* its Lease is fresh,
* its state stream has delivered first state,
* and its readiness does not say "down"
  ([DEP #11225](https://github.com/ai-dynamo/dynamo/issues/11225)
  readiness gate).

The Lease is the slow check, about a minute. The state stream is the fast
check: when it drops, the pool stops getting traffic right away.

#### Leases Do Not Expire on Their Own

Kubernetes never deletes an old Lease. The Global Router must decide
when a Lease is too old.

* The Global Router starts the timer when **it sees** a Lease update.
  It does not use `renewTime`. `renewTime` comes from the clock in the
  workload cluster, and that clock can differ from the hub's clock.
  Kubernetes checks node Leases the same way.
* If a workload cluster disappears for good, its record and Lease stay
  on the hub. The Global Router ignores stale records and reports them.
  Something on the hub should also clean them up, for example a small
  cleanup job or a hub-side controller.


#### The "No External Store" Rule

The HLD says "No etcd, database or shared cache is required for the
Global View". The hub API server is backed by etcd, so this DEP bends
that rule. We think this is acceptable because:

* **It adds no new dependency.** Dynamo on Kubernetes already uses the
  Kubernetes API for discovery (the `DynamoWorkerMetadata` CRD). This DEP
  does the same one level up: pools instead of workers.
* **Only small, slow data goes there.** Pool records change when a pool
  joins, leaves, or scales. All KV and load state stays in router memory
  and is rebuilt from the PoolRelays, as the HLD requires.

## Implementing the Interface

### Global Router Side

The Pool Catalog only reads. It depends on one interface:

```rust
/// Where the Pool Catalog gets the pool list. The catalog does not care
/// which implementation is behind it.
trait MembershipSource {
    /// Full list of pools right now, plus a version to watch from.
    async fn snapshot(&self) -> Result<Snapshot>;

    /// Changes after that version: Added, Updated, Removed.
    fn watch(&self, from: Revision) -> BoxStream<'static, MembershipEvent>;
}

struct Snapshot {
    pools: Vec<PoolRecord>,
    revision: Revision,
    /// true  = this is the full set (a new replica can wait for exactly these)
    /// false = only pools that happened to call this replica (use a timeout)
    complete: bool,
}
```

The gRPC handler writes through a second interface:

```rust
/// Where RegisterPool calls are stored. Only used by the gRPC options.
trait MembershipSink {
    async fn register(&self, rec: PoolRecord, caller: PoolIdentity) -> Result<()>;
    async fn heartbeat(&self, pool: &PoolId, caller: PoolIdentity) -> Result<()>;
    async fn deregister(&self, pool: &PoolId, caller: PoolIdentity) -> Result<()>;
}
```

Implementations:

| Implementation | Source | Sink | `complete` |
|---|---|---|---|
| `InMemoryMembership` | Yes | Yes | `false` |
| `KubernetesMembership` (read and write) | Yes | Yes | `true` |
| `KubernetesMembership` (read only) | Yes | No (the export agents write) | `true` |
| `FileMembership` | Yes | No | `true` |

Wiring:

* **gRPC Native:** gRPC server + `InMemoryMembership`.
* **Full Kubernetes:** no gRPC server + `KubernetesMembership` (read only).
* **Hybrid:** gRPC server + `KubernetesMembership` (read and write).

One setting picks the combination, for example
`--membership=grpc|kubernetes|hybrid|file`.

### Pool Side

* `GrpcAnnouncer`: calls `RegisterPool` on the regional address and sends
  heartbeats. Used by gRPC Native and Hybrid.
* `KubernetesExportAgent`: the reconciler in the Dynamo operator that
  writes the record and Lease to the hub. Used by full Kubernetes.

### Rules Shared by Every Option

These live in the Pool Catalog, not in the implementations, so behavior
is the same whichever option is chosen:

* **One record format.** `PoolRecord` has the same fields as the
  `DynamoPoolExport` spec. The gRPC `RegisterPool` message carries the
  same fields.
* **Expiry in one place.** The catalog removes a pool when it has not
  seen an update for the Lease or heartbeat duration, using its own clock.
* **Readiness from `complete`.** If `true`, wait for first state from
  every listed pool, with a timeout. If `false`, become ready after a
  fixed wait (soft readiness).
* **Same routable rule** as in
  [When Is a Pool Routable](#when-is-a-pool-routable).
* **Identity is checked where the write happens.** In the gRPC options,
  the gRPC handler checks the caller's mTLS identity and that the caller
  matches `pool_id` before calling the sink. In full Kubernetes, RBAC
  and a CEL validation policy on the hub do it.

# Alternate Solutions

## Alt 1 The Hub Reaches Into Each Cluster (Hub/Spoke)

The hub holds a kubeconfig for every workload cluster and reads pools
from them. GKE's multi-cluster Inference Gateway works this way, using
Google IAM. SIG Multicluster's ClusterProfile KEP
([KEP-4322](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/4322-cluster-inventory))
recommends this direction, with cloud identity federation instead of
stored keys.

**Pros:**

* Nothing to install in workload clusters.
* With identity federation, no long-lived credentials are stored anywhere.

**Cons:**

* Without identity federation, the hub holds credentials for every
  cluster.
* Identity federation is easiest when all clusters are in one cloud.
* Every workload cluster needs an inbound path from the hub.

**Reason Rejected:**

* Not rejected for single-cloud fleets. It goes against the HLD's
  dial-out direction.

## Alt 2 Static File Only

A list of pools in a file or ConfigMap. llm-d's multi-cluster router does
this today, and it can reload the file when it changes. Dynamo's Global
Router POC does it too
([#15338](https://github.com/ai-dynamo/dynamo/pull/15338)): a JSON file
lists each pool and its Relay and Frontend addresses, and the router
reads it once at startup.

**Pros:**

* The simplest option.

**Cons:**

* A person must edit the file for every change.
* If the file is read only at startup, every change also means
  restarting the routers.

**Reason Rejected:**

* Every change needs a person, so pools cannot join or leave on their
  own. It can still be a possible start (Phase 0), as in the POC, and a
  source for tests.


# Open Questions

1. Should the export agent live in the Dynamo operator or in the
   PoolRelay? The HLD says the PoolRelay registers. The operator already
   has a Kubernetes client and knows the DGD.
2. Final field list for `DynamoPoolExport`.
3. One namespace per pool, or one per workload cluster?
4. Only when GAIE is installed: should we also sync to GAIE's
   `InferencePoolImport`, or wait for it to leave draft status?
5. Who issues Relay Certificates?
6. How is each replica addressed in gRPC native?

# References

* [Global Router HLD](https://docs.google.com/document/d/1FYKvlsEnc6aMXgU_61RwtUJP3LWY7sQIEhOXjCFM2sk/edit?tab=t.bjkujx8pnylf):
  "Global View" section and open questions.
* [DEP #11225](https://github.com/ai-dynamo/dynamo/issues/11225):
  Multi-DC KV-aware routing and the KV DC Relay.
* [GAIE proposal 1374](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/1374-multi-cluster-inference):
  multi-cluster InferencePools, Hub/Spoke and Push/Pull topologies.
* [KEP-1645](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api):
  Multi-Cluster Services API.
* [ClusterProfile API (KEP-4322)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/4322-cluster-inventory).
* [Open Cluster Management: ManagedCluster registration](https://open-cluster-management.io/docs/concepts/cluster-inventory/managedcluster/).
* [Karmada pull mode registration](https://karmada.io/docs/userguide/clustermanager/cluster-registration/).
* [Submariner broker](https://submariner.io/getting-started/architecture/broker/).
* [llm-d file discovery](https://github.com/llm-d/llm-d-router/blob/main/pkg/epp/framework/plugins/datalayer/discovery/file/README.md).
* [GKE multi-cluster Inference Gateway](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/about-multi-cluster-inference-gateway).


