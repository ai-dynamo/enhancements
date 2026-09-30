# Kubernetes Native Pool Membership for the Global Router

**Status**: Draft

**Authors**: Anna Tchernych

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**:

**Required Reviewers**:

**Review Date**:

**Pull Request**:

**Implementation PR / Tracking Issue**:

# Summary

The Global Router needs to know which pools exist and how to reach them.
This DEP proposes a Kubernetes native way to do that. Each pool writes a
small record into the Kubernetes API of the hub cluster where the Global Router resides. Every Global
Router replica watches those records.

This covers **membership only**: which pools exist, and their addresses.
It does not cover how KV and load state flows, or how requests reach a
pool.

This DEP is part of the "K8s Native Global Router Solution" LLD listed in
the
[Global Router HLD](https://docs.google.com/document/d/1FYKvlsEnc6aMXgU_61RwtUJP3LWY7sQIEhOXjCFM2sk/edit?tab=t.bjkujx8pnylf).

This DEP is the general membership layer. The GAIE-based multi-cluster
DEP ([0000-multi-cluster-inference-gateway.md](0000-multi-cluster-inference-gateway.md))
is one consumer: its hub EPP can read the same `DynamoPoolExport` records.

# Motivation

The HLD's "Global View" section says that each pool has a **PoolRelay**.
The PoolRelay registers its pool with the Global Router with a gRPC call
(`RegisterPool`), and then sends heartbeats. Pools dial out to the
Global Router, so workload clusters do not need an open inbound port for
registration.

The HLD leaves these questions open:

* Open question 1: push, pull, or discovery for registration?
* Open question 9: how does a PoolRelay learn the Global Router
  addresses?

It also does not say how any of this maps to Kubernetes.

The hard part of question 9 is that the HLD wants **every** Global Router
replica to see **every** pool, with no replica-to-replica syncing of pool
state. Say there are 3 router replicas behind one load balancer:

* A PoolRelay calls `RegisterPool` on the load balancer. The call lands
  on **one** replica. The other two never hear about the pool.
* To reach all three, the PoolRelay needs each replica's own address.
* When the router scales to 4 replicas, every PoolRelay in every cluster
  must notice replica 4 and register with it too.
* Replica 4 starts empty. It cannot tell when it is ready, because it
  does not know how many pools exist. Has it heard from 8 of 8 pools, or
  8 of 20?

What is missing is a **directory** that both sides can find.

## Goals

* A pool can join and leave without restarting the Global Router.
* Pools dial out. The hub never needs credentials for workload clusters.
* Every Global Router replica sees every pool.
* A new replica knows the full list of pools when it starts.
* A pool can only register or change **its own** record.
* Align with Kubernetes multi-cluster standards where we can.

### Non Goals

* How KV and load state flows from a pool to the Global Router.
* How requests reach a pool (network reachability, tunnels).
* Deployments without Kubernetes. The HLD's `RegisterPool` gRPC path
  stays the way to cover them (see
  [Pluggable Membership Sources](#pluggable-membership-sources)).
* Requiring GAIE or a Gateway with the Inference Extension. This DEP
  uses only Dynamo CRDs and built-in Kubernetes APIs. When GAIE is
  installed, it can be used as an extra input.

# Proposal

## Overview

The hub cluster's Kubernetes API is the directory. Pools write to it.
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

## Components to write
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

### Kubernetes APIs Used

* Workload cluster, the agent reads: `DynamoGraphDeployment`, `Service`.
* Hub, the agent writes: `DynamoPoolExport`, `Lease`. In Phase 2 also
  `CertificateSigningRequest`.
* Hub, the Global Router reads: `DynamoPoolExport`, `Lease`.
* GAIE and Gateway API: optional only.


## Steps

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

## The Record

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

## Who Can Write What

The hub holds **no** credentials for workload clusters. Each workload
cluster holds a credential for the hub that can write **only its own
namespace**. So a pool cannot register a fake pool or change another
pool's record.

* **Phase 1:** the hub admin creates the namespace, a ServiceAccount, and
  RBAC for each pool. They give the workload cluster a token for it.
  Submariner joins clusters to its broker in a similar way.
* **Phase 2:** a short-lived bootstrap token is swapped for a certificate
  through a CSR, and the hub admin must accept the new pool. This is how
  Open Cluster Management and Karmada pull mode register clusters.

## When Is a Pool Routable

A pool is routable only when all of these are true:

* its `DynamoPoolExport` record exists,
* its Lease is fresh,
* its state stream has delivered first state,
* and its readiness does not say "down"
  ([DEP #11225](https://github.com/ai-dynamo/dynamo/issues/11225)
  readiness gate).

The Lease is the slow check, about a minute. The state stream is the fast
check: when it drops, the pool stops getting traffic right away.

### Leases Do Not Expire on Their Own

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

## How This Answers HLD Open Question 9

* The export agent needs **one** address: the hub's API server. It does
  not know or care how many router replicas exist.
* The pool writes its record **once**. Every replica sees it by watching.
* When the router scales from 3 to 4 replicas, nothing changes for the
  pools.
* A new replica lists the records at startup and knows the full set, for
  example 20 pools. It becomes ready when it has state for all 20.
  Readiness is well defined.

If the state stream also dials out (pool to router), the pools need the
router replica addresses. The same directory works in that direction too:
each replica writes its address to the hub, and pools watch that list.
The design of the state stream is out of scope here.

## Pluggable Membership Sources

This DEP does not replace the HLD's `RegisterPool` gRPC call. The Global
Router's Pool Catalog should accept more than one source:

| Source | Use it when |
|---|---|
| Static file (ConfigMap) | Testing, and very small fixed setups |
| `RegisterPool` gRPC (HLD) | Deployments without Kubernetes |
| Kubernetes watch (this DEP) | Kubernetes deployments |

All sources fill the same Pool Catalog. The routing logic does not
care where a pool came from. llm-d's router already works this way: its
discovery is a plugin, and a file-based one exists today.

## Tradeoffs

| | gRPC `RegisterPool` (HLD) | Kubernetes API (this DEP) |
|---|---|---|
| Works outside Kubernetes | **Yes** | No |
| Matches the HLD wording | **Yes** | Bends the "no external store" rule (see below) |
| Pool reaches every router replica (open question 9) | Needs extra work. For example, the reply to `RegisterPool` returns the replica list | **Yes.** Write once, every replica watches |
| New replica knows which pools to wait for | No. It must ask its peers or wait for a timeout | **Yes.** It lists the records |
| Identity, and one pool cannot touch another | We must build it: certificates, pool binding, revocation | **Built in:** RBAC, one namespace per pool, CSR, audit log |
| See which pools exist | We must build `GET /pools` | **`kubectl get dynamopoolexports -A`** |
| Standards | None | **Same pattern** as GAIE 1374 Push/Pull, Open Cluster Management, and Karmada pull mode |
| What workload clusters must reach | **A small gRPC port** | **The hub's API server.** Some teams will not allow this |
| Parts to build | **Fewer.** One RPC and a heartbeat | CRD, operator reconciler, credentials, Leases |
| Time to notice a dead pool | **Fast.** The connection drops | Slow, about a minute. The state stream covers this |
| Sources of truth | **One** connection | Two, Lease and stream. Needs the routable rule above |
| Hub API server is down | Not affected | Known pools keep working from the watch cache. New pools cannot join |

## Why the Kubernetes Native Approach Is Better for Kubernetes

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

It is **worse** when there is no Kubernetes, when a site will not expose
its hub API server, or when the fewest moving parts matter most. That is
why the membership source should be pluggable.

## The "No External Store" Rule

The HLD says "No etcd, database or shared cache is required for the
Global View". The hub API server is backed by etcd, so this DEP bends
that rule. We think this is acceptable because:

* **It adds no new dependency.** Dynamo on Kubernetes already uses the
  Kubernetes API for discovery (the `DynamoWorkerMetadata` CRD). This DEP
  does the same one level up: pools instead of workers.
* **Only small, slow data goes there.** Pool records change when a pool
  joins, leaves, or scales. All KV and load state stays in router memory
  and is rebuilt from the PoolRelays, as the HLD requires.



# Proposed shape for the DynamoPoolExport CRD

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


# Alternate Solutions

## Alt 1 gRPC `RegisterPool` Only

**Pros:**

* Works everywhere, with or without Kubernetes.
* Fewer parts to build.

**Cons:**

* Every pool must find every router replica.
* A new replica does not know which pools to wait for.
* Identity and per-pool isolation must be built from scratch.

**Reason Rejected:**

* Not rejected. It remains a membership source for deployments without
  Kubernetes.

## Alt 2 The Hub Reaches Into Each Cluster (Hub/Spoke)

The hub holds a kubeconfig for every workload cluster and reads pools
from them. GKE's multi-cluster Inference Gateway works this way, using
Google IAM.

**Pros:**

* Nothing to install in workload clusters.

**Cons:**

* The hub holds credentials for every cluster.
* Every workload cluster needs an inbound path from the hub.

**Reason Rejected:**

* It goes against the HLD's dial-out direction.

## Alt 3 Static File Only

A list of pools in a ConfigMap. llm-d's multi-cluster router does this
today.

**Pros:**

* The simplest option.

**Cons:**

* A person must edit the file for every change.

**Reason Rejected:**

* Kept only as a source for tests and small setups (Phase 0).

# Open Questions

1. Should the export agent live in the Dynamo operator or in the
   PoolRelay? The HLD says the PoolRelay registers. The operator already
   has a Kubernetes client and knows the DGD.
2. Final field list for `DynamoPoolExport`.
3. One namespace per pool, or one per workload cluster?
4. Only when GAIE is installed: should we also sync to GAIE's
   `InferencePoolImport`, or wait for it to leave draft status?

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
