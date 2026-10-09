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
3. A full Kubernetes approach for clusters on one trusted network that form a SIG Multicluster ClusterSet. It uses the standard Multi-Cluster Services API instead of a Dynamo-specific record.

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
* Requiring GAIE or a Gateway with the Inference Extension. The gRPC and
  Hybrid options use only Dynamo CRDs and built-in Kubernetes APIs. The
  Full Kubernetes option uses the SIG Multicluster APIs. When GAIE is
  installed, it can be used as an extra input.

# Proposal

## Overview
The proposal aims to define abstractions for discovery so that a client can decide. 

## Comparing the Three Options

| | gRPC Native | Hybrid | Full Kubernetes |
|---|---|---|---|
| Works outside Kubernetes | **Yes** | Pools: yes. The Global Router must run on Kubernetes | No |
| Matches the HLD wording | **Yes** | Partly. Same `RegisterPool`, but records are stored in etcd | Bends the "no external store" rule (see below) |
| Pool reaches every router replica (open question 9) | Needs extra work. The reply to `RegisterPool` returns the replica list, and pools connect to each replica | **Yes.** One regional address. Every replica watches | **Yes.** The router's headless export gives every cluster the replica list |
| New replica knows which pools to wait for | No exact answer. Ask a peer replica, or wait for a timeout | **Yes.** It lists the records | **Yes.** It lists the imported EndpointSlices |
| Identity, and one pool cannot touch another | We build it: mTLS, pool binding, revocation | We build it: mTLS in the Global Router | **Trusted network.** MCS does not authenticate. RBAC in each cluster decides who can export. Mesh mTLS is optional |
| See which pools exist | We build `GET /pools` | `kubectl get dynamopoolexports -A` | `kubectl get serviceimports -A` on the hub |
| Standards | None | Kubernetes store, custom registration | **SIG Multicluster:** MCS, About API, optional ClusterProfile. Same model as GAIE 1374 |
| What workload clusters must reach | Every router replica's gRPC address | The regional gRPC address | The router pods in the hub, over the trusted pod network. Not the hub's Kubernetes API |
| Extra infrastructure | An address per replica (a load balancer per replica, or a router like `stargate-k8s-router`), and a certificate authority for mTLS | A certificate authority for mTLS | An MCS implementation (Submariner, Cilium ClusterMesh, a cloud MCS service) and a pod network that reaches across clusters |
| Parts to build | RPCs, heartbeat, replica list, pool-side connection to every replica, mTLS | RPCs, heartbeat, mTLS, the router writes records and Leases, watch | `ServiceExport` in the operator, router export in Helm, EndpointSlice watchers in the router and the PoolRelay |
| Time to notice a dead pool | Fast. Missed heartbeats | Set by the Lease length. The state stream covers this | Readiness probe plus the MCS implementation's sync delay. The state stream covers this |
| Hub API server is down | Not affected | Known pools keep working from the watch cache. New pools cannot join | Known pools keep working from the watch cache. New pools cannot join |

## Which Option When

| Setup | Pick |
|---|---|
| No Kubernetes, or the Global Router does not run on Kubernetes | **gRPC Native** |
| The Global Router runs on Kubernetes, but workload clusters must not reach the hub's Kubernetes API (different owners, several clouds, strict security rules) | **Hybrid** |
| One owner and a trusted network between clusters, with an MCS implementation that joins them, the hub included, into a ClusterSet | **Full Kubernetes** |
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
| Full Kubernetes | The operator creates a `ServiceExport`. MCS imports it into the hub | Hub Kubernetes API (`ServiceImport` and EndpointSlices) | Every replica, by watching |
| Hybrid | PoolRelay calls `RegisterPool` | The receiving replica writes to the hub Kubernetes API | Every replica, by watching |
| Static file (possible start) | A person edits the file | ConfigMap | Every replica |

The interface itself is described in
[Implementing the Interface](#implementing-the-interface), after the
three approaches.

## Hybrid Approach (gRPC + Kubernetes)

Pools keep talking gRPC. The Global Router uses Kubernetes behind the
scenes to share what it hears. The discovery information is stored in the Hub's Kubernetes API.

1. A Global Router sits behind a load balancer with a fixed name.
2. A pool's relay is given that address as a config. The PoolRelay calls `RegisterPool` on this region's single
   Global Router address.
3. The load balancer sends the call to one GR replica.
4. That replica writes a small record for the pool into the Hub Cluster's 
   Kubernetes API Server.
5. Every GR replica watches those records, so every replica sees every
   pool, even though the pool talked to only one of them.
6. A new replica reads all the records at startup and knows which pools
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


**Cons:**

* The biggest issue with the gRPC Native approach is the absence of the shared list of pools.
  Each router replica only knows the pools that happened to call it. Three problems follow:

  1. Every GR replica must see every pool. There is a load balancer in the Gub in front of every GL replica. When a relay calls`/RegisterPool` the load balancer sends this to 1 replica only. To solve this the `/RegisterPool` can return the list of replicas. The PoolRelay would connect to each replica to refresh its list. To make sure each replica knows about another 2 approaches exist:
     1. Use a smart gateway like  stargate-k8s-router.
     2. Put an additional load balancer per replica. For the first call each Relay reaches the main load balancer and then it works with per replica load balancers.
  2. A new replica can't know when it's ready, because it doesn't know how many pools exist. A new replica only learns about the pool as the find it and call in. So it does not know how many to expect to mark itself ready. Ways to mitigate:
     1. A new replica asks a peer how many pools exist (snapshot). The issue that the  HLD does not want the replica-to-replica communication. If all replicas restart together then nobody knows.
     2. Just say ready when one pool is found
     3. Wait a fixed time. A relay sends heartbeats and in response the GR sends a list of replicas. When a relay finds a new replica in the list it is calls the `/RegisterPool` on it. The downside that this is eventually consistent.
     4. Re-implement the solution using a library with a gossip/swim protocol (@stefan Shumaksi)
  3. Extra infra is required. We need a load balancer per Global Router replica or a router that sends each connection to a named replica like stargate-k8-router.

* We need to re-implement some machinery Kubernetes gives us for free. For example, Kubernetes RBAC does not protect each pool's record. The Global Router must check who is calling, for example with mTLS.
* Teams where security is not an issue may be more comfortable reusing existing solutions. 


## Full Kubernetes Approach (SIG Multicluster MCS)

This option uses the SIG Multicluster standard APIs ([SIG Multicluster](https://multicluster.sigs.k8s.io/#approach)) and available only on clusters on one trusted network where MCS can skip authentication.
For V1 we also assume that pods in different cluster can reach each other. (i.e. Azure CNI with peered vNets, AWS VPC CNI with peered VPCs)
Otherwise something like Submariner is needed. 
For the V1 we will assume the use of an implementation of the Multi-Cluster Services (MCS) API with Karmada being the first choice. 

```
 Workload cluster A                      Hub cluster
 +----------------------------+          +--------------------------------+
 | DGD "qwen3-32b"            |          | ServiceImport "qwen3-32b"      |
 | PoolRelay Service          |   MCS    |   EndpointSlice per cluster    |
 |   + ServiceExport ---------|--------->|   (source-cluster: cluster-a)  |
 |                            |          |        ^ watch                 |
 | ServiceImport              |   MCS    |        |                       |
 |   "global-router" <--------|----------| Global Router replicas 1..N   |
 |                            |          |   headless Service             |
 | PoolRelay -----------------|- dials ->|   + ServiceExport              |
 +----------------------------+  each    +--------------------------------+
                                 replica
```

1. **The user marks a pool for export,** with a field on the DGD in the
   workload cluster. If GAIE is installed, the GAIE export annotation on
   the InferencePool
   ([1374](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/1374-multi-cluster-inference))
   can also trigger it.
2. **The operator creates a `ServiceExport`** for the pool's PoolRelay
   Service, in the DGD's namespace, named after the DGD.
3. **The MCS implementation imports it into the hub:** a `ServiceImport`
   and EndpointSlices. 
4. **Every Global Router replica watches** those EndpointSlices and
   builds its Pool Catalog from them. A pool is live while its PoolRelay
   endpoint is Ready.
5. **The PoolRelay finds every router replica.** The hub exports the
   Global Router's Service. Every workload cluster then has the
   router's EndpointSlices locally. The PoolRelay reads them and dials
   each replica.
6. **The pool leaves.** When export is turned off or the DGD is deleted,
   the operator deletes the `ServiceExport`, and the pool's endpoints
   disappear from the hub. If the PoolRelay dies, its endpoint stops
   being Ready.

**Pros:**

* Standard SIG Multicluster APIs. No Dynamo CRD on the hub. Same model as
  GAIE 1374.
* Every replica sees every pool, and a new replica lists the imported
  EndpointSlices to know which pools to wait for.
* Liveness comes from endpoint readiness. No Leases and no heartbeats.
* PoolRelays find every router replica from a local EndpointSlice (open
  question 9).
* Using a 3rd party provider saves us from giving every workload cluster credentials to the Kubernetes Server in the hub.  We could provide a basic solution for teams on a private network but the 3rd party solution provides a wider use case. 

**Cons:**

* Needs an MCS implementation and a pod network that reaches across
  clusters, the hub included. Many teams do not have this.
* MCS does not authenticate. It relies on the trusted network.
* Support for headless export and `exportedAnnotations` differs between
  MCS implementations.
* The MCS API is `v1alpha1`, and ClusterProfile is alpha.
* Imports are stored in etcd, so it bends the HLD's "no external store"
  rule, like Hybrid.

## Implementing the Interface

Related changes first:
1. In the [GlobalViewRuntime](https://github.com/ai-dynamo/dynamo/pull/15301/changes) a pool"s  `RelayDgdSource` changes. As before it holds data about the pool, but it no longer says how to reach the Relay. It will have an incoming connection from the Relay instead. 
2. The Global Router will run a gRPC server. Today the Relay is the server and the router subscribes. With this DEP the Relay will call into router ot we do the "reverse tunnel: the relay opens the connection and the router still subscribes over it. This is TBD.
3. Rolling updates is outside of this DEP and a TODO.


Implementations:

| Implementation | Source | Sink | `complete` |
|---|---|---|---|
| `InMemoryMembership` | Yes | Yes | `false` |
| `KubernetesMembership` (read and write) | Yes | Yes | `true` |
| `McsMembership` (reads imported EndpointSlices) | Yes | No (the operator exports) | `true` |
| `FileMembership` | Yes | No | `true` |

Wiring:

* **gRPC Native:** gRPC server + `InMemoryMembership`.
* **Full Kubernetes:** no `RegisterPool` server + `McsMembership`.
* **Hybrid:** gRPC server + `KubernetesMembership` (read and write).

One setting picks the combination, for example
`--membership=grpc|kubernetes|hybrid|file`.

### Pool Side

* `GrpcAnnouncer`: calls `RegisterPool` on the regional address and sends
  heartbeats. Used by gRPC Native and Hybrid.
* Full Kubernetes has no announcer in the PoolRelay. The operator
  creates the `ServiceExport`. The PoolRelay only reads the Global
  Router's imported EndpointSlices to find the replicas.

### Rules Shared by Every Option

These live in the Pool Catalog, not in the implementations, so behavior
is the same whichever option is chosen:

* **One record format.** The gRPC `RegisterPool` message and the Hybrid
  record carry the same fields as `PoolRecord`. In Full Kubernetes, the
  pool's name comes from the import, and the other fields come from the
  PoolRelay catalog (see [Pool Details](#pool-details)).
* **Expiry in one place.** The catalog removes a pool when it has not
  seen an update for the Lease or heartbeat duration, using its own clock.
  In Full Kubernetes, it removes a pool when the pool's endpoints
  disappear or stop being Ready.
* **Readiness from `complete`.** If `true`, wait for first state from
  every listed pool, with a timeout. If `false`, become ready after a
  fixed wait (soft readiness).
* **Same routable rule** as in
  [When Is a Pool Routable](#when-is-a-pool-routable).
* **Identity is checked where the write happens.** In the gRPC options,
  the gRPC handler checks the caller's mTLS identity and that the caller
  matches `pool_id` before calling the sink. In full Kubernetes, RBAC in
  each workload cluster decides who can create a `ServiceExport`, and the
  network is trusted.

### When Is a Pool Routable

A pool is routable only when all of these are true:

* its record exists,
* its Lease or heartbeat is fresh, or in Full Kubernetes its endpoint is Ready,
* its state stream has delivered first state,
* and its readiness does not say "down"
  ([DEP #11225](https://github.com/ai-dynamo/dynamo/issues/11225)
  readiness gate).

The Lease or heartbeat is the slow check. The state stream is the fast
check: when it drops, the pool stops getting traffic right away.

### Leases Do Not Expire on Their Own

This applies to Hybrid. Full Kubernetes uses endpoint readiness instead.
Kubernetes never deletes an old Lease, so the Global Router must decide
when a Lease is too old.

* The Global Router starts the timer when **it sees** a Lease update.
  It does not use `renewTime`, because the replica that wrote it used
  its own clock, which can differ from the reading replica's clock.
  Kubernetes checks node Leases the same way.
* If a pool disappears for good, its record and Lease stay on the hub.
  The Global Router ignores stale records and reports them. Something on
  the hub should also clean them up, for example a small cleanup job.

### The "No External Store" Rule

The HLD says "No etcd, database or shared cache is required for the
Global View". Hybrid stores pool records in the hub's Kubernetes API,
and Full Kubernetes stores imported Services there. Both are backed by
etcd, so they bend that rule. We
think this is acceptable because:

* **Dynamo already does this inside one cluster.** It uses the
  Kubernetes API for worker discovery (the `DynamoWorkerMetadata` CRD).
  These options do the same one level up: pools instead of workers.
* **Only small, slow data goes there.** Pool records change when a pool
  joins, leaves, or scales. All KV and load state stays in router memory
  and is rebuilt from the PoolRelays, as the HLD requires.

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

1. Which pool details move into the PoolRelay catalog, and which use
   `exportedAnnotations`?
2. Namespace sameness: a DGD's namespace must have one owner in every
   cluster, the hub included. How do teams name namespaces across
   clusters?
3. Only when GAIE is installed: should we also sync to GAIE's
   `InferencePoolImport`, or wait for it to leave draft status?
4. Who issues Relay Certificates?
5. How is each replica addressed in gRPC native?
6. Which MCS implementations support headless export across clusters
   well enough for per-replica dialing?
7. Read location from ClusterProfile, or configure it per cluster ID?

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
* [SIG Multicluster approach](https://multicluster.sigs.k8s.io/#approach):
  ClusterSet, namespace sameness, About API, MCS, ClusterProfile.
* [About API](https://multicluster.sigs.k8s.io/concepts/about-api/):
  `ClusterProperty` and the cluster ID.
* [Open Cluster Management: ManagedCluster registration](https://open-cluster-management.io/docs/concepts/cluster-inventory/managedcluster/).
* [Karmada pull mode registration](https://karmada.io/docs/userguide/clustermanager/cluster-registration/).
* [Submariner broker](https://submariner.io/getting-started/architecture/broker/).
* [llm-d file discovery](https://github.com/llm-d/llm-d-router/blob/main/pkg/epp/framework/plugins/datalayer/discovery/file/README.md).
* [GKE multi-cluster Inference Gateway](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/about-multi-cluster-inference-gateway).

# Appendix: Full Kubernetes Details

## Objects

| Object | Cluster | Created by | Read by | Purpose |
|---|---|---|---|---|
| `ServiceExport` for the PoolRelay Service | Workload | Dynamo operator | MCS implementation | Announces the pool |
| `ServiceImport` and EndpointSlices for each pool | Hub | MCS implementation | Global Router | Pool list and liveness |
| Headless Service and `ServiceExport` for the Global Router | Hub | Global Router Helm chart | MCS implementation | Announces the router replicas |
| `ServiceImport` and EndpointSlices for the Global Router | Every workload cluster | MCS implementation | PoolRelay | Replica addresses for the state stream |
| `ClusterProperty` `cluster.clusterset.k8s.io` | Every cluster | Cluster admin or cluster manager | MCS implementation | Cluster ID, used as `siteId` |
| `ClusterProfile` (optional) | Hub | Cluster manager (OCM, Karmada, a cloud fleet service) | Global Router | Cluster list and location |

## Where Each PoolRecord Field Comes From

| Field | Source |
|---|---|
| `siteId` | The EndpointSlice's `multicluster.kubernetes.io/source-cluster` label |
| DGD namespace | The `ServiceImport` namespace |
| DGD name | The `ServiceImport` name |
| Location | ClusterProfile properties, or Global Router configuration keyed by cluster ID |
| Model, frontend endpoint | The PoolRelay catalog on the state stream |
| Runtime namespace, Relay identity | The PoolRelay catalog (to add), or `exportedAnnotations` on the `ServiceExport` |

## Components to Write

1. **Export in the operator.** When the DGD's export field is set, the
   operator creates a `ServiceExport` for the PoolRelay Service, and
   deletes it when export is turned off.
2. **Router export in the Global Router Helm chart:** a headless Service
   and its `ServiceExport`.
3. **A watcher in the Global Router** for imported EndpointSlices, which
   builds the Pool Catalog.
4. **A watcher in the PoolRelay** for the Global Router's imported
   EndpointSlices, which gives the replica list.
5. **Optional:** a ClusterProfile reader in the Global Router, for
   location.

No Dynamo CRD, no Leases, and no per-pool hub credentials.

## Kubernetes APIs Used

* Workload cluster, the operator reads `DynamoGraphDeployment` and
  writes `ServiceExport` (`multicluster.x-k8s.io/v1alpha1`).
* Workload cluster, the PoolRelay reads `EndpointSlice`.
* Hub, the Global Router reads `ServiceImport` and `EndpointSlice`, and
  optionally `ClusterProfile` (`multicluster.x-k8s.io/v1alpha1`).
* Every cluster: `ClusterProperty` (`about.k8s.io`), read by the MCS
  implementation.
* GAIE and Gateway API: optional only.

## State Stream Direction

The PoolRelay dials out to every router replica. In Full Kubernetes it
finds them in the Global Router's imported EndpointSlices:

* Export the router as a **headless** Service. A ClusterSet IP would send
  each connection to only one replica.
* Use the EndpointSlice addresses. Per-pod DNS names
  (`<hostname>.<clusterid>.<svc>.<ns>.svc.clusterset.local`) exist only
  when pods have hostnames, for example in a StatefulSet.

The design of the state stream protocol is out of scope here.
