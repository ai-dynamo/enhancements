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

**Implementation PR / Tracking Issue**: the DEP issue is
[ai-dynamo/dynamo#17600](https://github.com/ai-dynamo/dynamo/issues/17600).
The PRs are under [Implementation](#implementation).

# Summary

The Global Router needs to know which pools exist and how to reach them.
Each PoolRelay needs to know the Global Router replicas, to send them its
pool's state. This DEP covers only this membership. It does not cover how
KV and load state flows, or how requests reach a pool.

A **Global Discovery Service** answers both questions over HTTP. The Global
Router and the PoolRelay make the same calls in every mode. Two knobs choose
what is behind the service:

1. `--pool-discovery`: where the pool list comes from.
   * `registrations`: from the PoolRelays' own `POST /pools` calls.
   * `karmada` or `mcs`: from the pools that the Multi-Cluster Services API
     (MCS) imports, with Karmada or with a standard MCS implementation.
2. `--registration-storage`: where registrations are kept. `memory` now;
   Kubernetes or DynamoDB later.

This DEP is part of the "K8s Native Global Router Solution" LLD listed in
the
[Global Router HLD](https://docs.google.com/document/d/1FYKvlsEnc6aMXgU_61RwtUJP3LWY7sQIEhOXjCFM2sk/edit?tab=t.bjkujx8pnylf).

# Motivation

The HLD's "Global View" section says that each pool has a **PoolRelay**.
The PoolRelay registers its pool and then sends heartbeats. Pools dial out,
so workload clusters need no open inbound port for registration.

The HLD left these questions open:

* Open question 1: push, pull, or discovery for registration?
* Open question 9: how does a PoolRelay learn the Global Router replica
  addresses?

It also did not say how this maps to Kubernetes.

The hard part is that every Global Router replica must see every pool,
with no replica-to-replica syncing. If PoolRelays register with router
replicas behind one load balancer, each call reaches one replica, and the
others never hear about the pool. A new replica also cannot tell when it is
ready, because it does not know how many pools exist.

The Global Router POC ([#15338]) avoids this with a static JSON list of
pools, read at startup.

## Goals

* The Global Router and the PoolRelay work the same in every mode.
* The deployer picks the mode with two flags.

The modes are compared on these properties:

* A pool can join and leave without restarting anything.
* Pools dial out.
* Every Global Router replica sees every pool.
* A reader knows when the pool list is complete.
* A pool can only register or change its own record.
* Kubernetes multi-cluster standards are used where possible.

Not every mode has every property. See
[Comparing the Modes](#comparing-the-modes).

### Non Goals

* How KV and load state flows from a pool to the Global Router, and which
  side opens that connection.
* How requests reach a pool (network reachability, tunnels).
* Requiring GAIE or a Gateway with the Inference Extension. When GAIE is
  installed, it can be an extra input.

# Proposal

## The Global Discovery Service

| Call | Called by | What it does |
|---|---|---|
| `POST /pools` | Each PoolRelay, on a timer | Registers its pool: key (`cluster_id/namespace/dgd_name`), location, model, capabilities, frontends and capacity. Each call is also the heartbeat. The reply is a lease: call again every 5 s, and the registration lasts 15 s (the defaults) |
| `GET /pools` | Global Router replicas | The live pools, and whether the list is `complete` |
| `GET /routers` | PoolRelays | The Global Router replicas to send state to |
| `POST /routers` | Nobody yet | Reserved. Answers `501` until router registration is designed ([#17333]) |
| `GET /registrations` | Other discovery service instances | The registrations this instance received, so that instances can share them (`--peer-service`, [#17332]) |
| `GET /healthz` | Kubernetes | Readiness |

Every router replica reads the same pool list, and every PoolRelay finds
every router replica with one call (open question 9). To register, a
PoolRelay needs only the discovery service's address.

## The Two Knobs

### `--pool-discovery`: where the pool list comes from

* **`registrations`** (the default). A registration is one `POST /pools`
  call. A pool is listed while its PoolRelay keeps registering within the
  lease, and dropped when the lease runs out. Router replicas come from
  `--router <id>=<url>`, because routers cannot register yet.
* **`karmada`** or **`mcs`**. MCS decides which pools exist: those whose
  PoolRelay Service (`<DGD name>-pool-relay`) is imported with a Ready
  endpoint. Registrations still add what MCS cannot know: the model,
  capabilities, capacity and location ([#17332]). A pool is listed only
  while both agree. Router replicas are the Ready endpoints of the router Service
  that MCS imports. The two values differ in how the imported
  EndpointSlices are named and labeled, and they match the operator's
  `mcsImplementation`.

A flag that works with only some values fails at startup when it is given
with another.

### `--registration-storage`: where registrations are kept

* **`memory`** (the default, and the only value today). Each instance keeps
  the registrations it receives. With `registrations`, run one instance.
  With `karmada` or `mcs`, `--peer-service` lets the instances in every
  cluster share them. After a restart, the list fills again within one
  heartbeat.
* **Kubernetes** (later). Registrations and their Leases in the hub's
  Kubernetes API. Every instance reads the same records, and they survive
  restarts.
* **DynamoDB** (later). Global tables that every region reads, so instances
  do not need to ask each other.

The two knobs are independent. Every `--pool-discovery` value takes
registrations, so each storage works with each of them.

### Modes

| Mode | `--pool-discovery` | `--registration-storage` | Code |
|---|---|---|---|
| Registrations (was "gRPC Native") | `registrations` | `memory` | [#17331], [#17371] |
| Hybrid | `registrations` | Kubernetes | Not built |
| MCS (was "Full Kubernetes") | `karmada` or `mcs` | `memory` | [#17331], [#17371], [#17332] |

## Comparing the Modes

| | Registrations | Hybrid (not built) | MCS |
|---|---|---|---|
| Needs Kubernetes | No | The discovery service runs on the hub | Yes, plus an MCS implementation and a pod network across clusters |
| Discovery service instances | One | Many. They read the same records | Many. They share registrations (`--peer-service`) |
| Router replicas (`GET /routers`) | Set with `--router` | Set with `--router` | Found from the router Service that MCS imports |
| A pool leaves when | Its lease runs out | Its lease runs out | Its lease runs out, or MCS no longer shows a Ready PoolRelay |
| The list is complete | One lease after the service starts | Once the records are read | After the first MCS sync and one lease |
| Who can add a pool | Anyone who can reach the service. V1 trusts the network | The same. mTLS can come later | Only a pool whose PoolRelay Service is exported. RBAC in each cluster decides who can export |
| The Kubernetes API is down | Not affected | Known pools stay. New pools cannot join | Known pools stay, marked incomplete. New pools cannot join |
| Standards | None | Kubernetes storage, custom registration | SIG Multicluster MCS, the same model as GAIE 1374 |

## Which Mode When

| Setup | Pick |
|---|---|
| No Kubernetes, or tests | **Registrations** |
| Kubernetes, but no trusted network between clusters | **Registrations** now. **Hybrid** or DynamoDB storage later |
| One owner and a trusted network, with an MCS implementation (Dynamo's standard one, or Karmada) | **MCS** |
| First tests, or a few pools that rarely change | A static file, as in the POC |

### Standard MCS or Karmada

Use the [standard implementation](#the-standard-mcs-implementation) in
`deploy/global-discovery-mcs` for new fleets, and whenever clusters span
regions. It needs no MCS hub: each cluster runs `mcs-controller`, and
`mcs-registry` with a DynamoDB Global Table serves every region. It
imports each exported Service into every cluster on its own. It is a
standard KEP-1645 implementation: its slices carry the KEP labels, and
its exports report conflicts. Karmada's MCS is partial. Its imported
slices have no KEP labels, so the source cluster is parsed from the slice
name, which limits a Service name to 55 characters. Each Service needs a
manual `ServiceImport` and `PropagationPolicy` per destination. Its CRDs
serve `v1alpha1` only, it does not handle conflicts, and it has one hub,
whose hostPath etcd loses its state when the hub's pod moves. Karmada
still fits a fleet that already runs it: set `--pool-discovery karmada`
and the operator's `mcsImplementation` to `karmada`.

## Registrations

1. Each PoolRelay calls `POST /pools` on the discovery service every
   heartbeat.
2. The service keeps each registration in memory until its lease runs out.
3. Router replicas read `GET /pools`. PoolRelays read `GET /routers`, which
   lists the `--router` flags.

**Pros:** no Kubernetes needed. One address for PoolRelays. Every router
replica sees the same list.

**Cons:** one instance, since each keeps only what it receives. A restart
empties the list until PoolRelays register again. Router replicas are set
by hand. No caller check yet.

## Hybrid (not built)

Like Registrations, but the service writes each registration and its Lease
to the hub's Kubernetes API.

**Pros:** many instances share the list, and it survives restarts.
Workload clusters never reach the hub's Kubernetes API.

**Cons:**

* The service must run on the hub and needs write access there.
* Kubernetes RBAC does not protect each record, because the service writes
  it. The service must check who is calling.
* Kubernetes never deletes an old Lease, so the reader decides when one is
  too old. It starts the timer when it sees a Lease update, not from
  `renewTime`, which another instance wrote with its own clock. Kubernetes
  checks node Leases the same way. A cleanup job on the hub can delete
  stale records.

## MCS (Karmada or Standard)

This mode uses the SIG Multicluster APIs
([SIG Multicluster](https://multicluster.sigs.k8s.io/#approach)). It is only
for clusters on one trusted network, where MCS can skip authentication. V1
also assumes that pods in different clusters can reach each other (for
example, peered VPCs with the AWS VPC CNI, or peered VNets with Azure CNI).
Otherwise something like Submariner is needed.

Dynamo provides a standard MCS implementation
([KEP-1645](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api)),
shipped in `deploy/global-discovery-mcs` ([#17601]). Karmada is an
alternative. The discovery service reads the slices of either:
`--pool-discovery mcs` for the standard implementation, `karmada` for
Karmada. See [Standard MCS or Karmada](#standard-mcs-or-karmada).

```
 Workload cluster                          Hub cluster
 +------------------------------+          +--------------------------------+
 | DGD "qwen3-32b"              |          | Global Discovery Service       |
 |   qwen3-32b-pool-relay       |   MCS    |   reads the imported           |
 |     + ServiceExport ---------|--------->|   EndpointSlices of the        |
 |   qwen3-32b-frontend         |          |   PoolRelay, frontend and      |
 |     + ServiceExport ---------|--------->|   router Services              |
 |                              |          |                                |
 | PoolRelay                    |          | Global Router replicas 1..N    |
 |   POST /pools, GET /routers -|--------->|   Service + ServiceExport      |
 |   dials each router replica -|--------->|   read GET /pools              |
 +------------------------------+          +--------------------------------+
```

1. **Mark the pool.** Annotate the DGD with
   `nvidia.com/global-router-pool: "true"`, and set the operator's
   `multiCluster.mcsImplementation` to `karmada` or `mcs`.
2. **The operator exports it.** It creates `ServiceExport`s for
   `<DGD name>-pool-relay` and `<DGD name>-frontend`. Removing the
   annotation withdraws them.
3. **MCS imports them** as EndpointSlices, in the cluster where the
   discovery service runs, or on the Karmada control plane. The standard
   implementation imports them into every cluster.
4. **The discovery service lists the pool** while its PoolRelay endpoint is
   Ready and its registration is live. The frontend addresses come from
   MCS. The model, capabilities, capacity and location come from the
   registration.
5. **PoolRelays find the routers.** An admin exports the router Service
   once. `GET /routers` lists its Ready endpoints, so PoolRelays read no
   EndpointSlices.
6. **The pool leaves** when its export is withdrawn, its PoolRelay stops
   being Ready, or its registrations stop.

**Pros:**

* Standard SIG Multicluster APIs, and no Dynamo CRD on the hub. The same
  model as GAIE 1374.
* A registration alone cannot add a pool. MCS must show it too.
* Router replicas are found without a `--router` list.
* Instances in every cluster share registrations, so every region lists
  every pool.
* Workload clusters need no credentials for the hub's Kubernetes API.

**Cons:**

* Needs an MCS implementation and a pod network across clusters. Many
  teams do not have this.
* MCS does not authenticate. It relies on the trusted network.
* Karmada names an imported Service `derived-<name>`, which must fit in 63
  characters. So a Service name can have at most 55 characters, and a DGD
  name at most 44. Karmada also watches a member's `ServiceExport`s only
  while it has propagated at least one there. The standard
  implementation names it `derived-<hash>`, whose length does not depend
  on the Service name.
* If every discovery instance in a cluster is down, that cluster's pools
  drop out of every region within a lease. Storage that every region reads,
  such as DynamoDB, would remove this.
* Karmada's MCS CRDs serve `v1alpha1` only. The `sigs.k8s.io/mcs-api`
  v0.5.2 CRDs serve `v1beta1` too. Imports are stored in etcd.

### The Standard MCS Implementation

`deploy/global-discovery-mcs` ([#17601]) has two parts:

* **`mcs-registry`**, one service for the ClusterSet. It keeps a lease for
  each cluster, each cluster's Service exports and endpoint sets, and the
  ServiceImports merged from them. It stores them in memory, or in a
  DynamoDB Global Table when clusters span regions.
* **`mcs-controller`**, one in each cluster. It exports every Service that
  has a `ServiceExport`, or the label
  `dynamo.nvidia.com/mcs-export: "true"`. In every cluster it creates the
  `ServiceImport`, a `derived-<hash>` Service, and EndpointSlices labeled
  `multicluster.kubernetes.io/service-name` and
  `multicluster.kubernetes.io/source-cluster`. When clusters export the
  same Service in conflicting ways, the oldest export wins (the KEP-1645 rule),
  and the `ServiceExport` reports `Valid`, `Ready` and `Conflict`
  conditions. With `--write-cluster-properties` it also creates the
  `ClusterProperty` objects `clusterset.k8s.io` and
  `cluster.clusterset.k8s.io`.

The discovery service reads these slices unchanged, with
`--pool-discovery mcs`. Pool liveness is the same as with Karmada: a pool
is live while its PoolRelay endpoint is Ready. `mcs-controller` can also
stamp `topology.kubernetes.io/region` on the slices it imports, so a
pool's region can come from that label instead of the cluster name.

## Implementing the Interface

The library ([#17331]) puts the contract first and the backends below it:

* **Contract** (`pool_discovery`): `PoolDirectory` lists and watches pools.
  `PoolRegistrar` registers a pool and returns its `Lease`.
  `ReplicaDirectory` lists and watches router replicas.
* **Backends** (`backends`): memory (`MemoryPoolStore`), MCS
  (`McsPoolDirectory`, `McsReplicaDirectory`), and a static replica list
  (`StaticReplicaDirectory`).
* **Shared types** (`lib/global-common`): `PoolKey`, `PoolLocation`,
  `PoolRegistration`, `PoolListing`, `Lease`, `RouterReplica`, `WatchEvent`,
  `Snapshot`, `Completeness`.

The HTTP service ([#17371]) builds its backend, `PoolDiscoveryBackend`, from
the two knobs. A PoolRelay will call it through `lib/global-discovery-client`.

| Function | `--pool-discovery registrations` | `--pool-discovery karmada` or `mcs` |
|---|---|---|
| Lists and watches the pools: `PoolDirectory` | The live registrations | The pools whose PoolRelay MCS shows Ready, joined with their live registrations |
| Keeps a registration: `PoolRegistrar` | Until its lease runs out | The same |
| Lists the router replicas: `ReplicaDirectory` | The `--router` flags | The Ready endpoints of the router Service that MCS imports |
| Admits a PoolRelay's state stream (router) | Not designed yet. Listings name no PoolRelay | `relay_identities`: the PoolRelay pod's addresses, matched against the caller's source IP |
| Registers the pool (PoolRelay) | `POST /pools` on a timer | `POST /pools` on a timer. The operator exports its Services |
| Finds the router replicas (PoolRelay) | `GET /routers` | `GET /routers` |
| Shares the exports between clusters (MCS implementation) | Not used | `mcs`: `mcs-registry`, in memory or a DynamoDB Global Table. `karmada`: the Karmada control plane |
| Creates the `ServiceImport` (MCS implementation) | Not used | `mcs`: `mcs-controller`, in every cluster. `karmada`: an admin, with a `PropagationPolicy` per Service and destination |
| Creates the derived Service (MCS implementation) | Not used | `mcs`: `mcs-controller`, named `derived-<hash>`. `karmada`: Karmada, named `derived-<name>` |
| Creates the imported EndpointSlices (MCS implementation) | Not used | `mcs`: `mcs-controller`, labeled `multicluster.kubernetes.io/service-name` and `multicluster.kubernetes.io/source-cluster`. `karmada`: Karmada, without these labels |
| Names the ClusterSet and the cluster: `ClusterProperty` (MCS implementation) | Not used | `mcs`: `mcs-controller` creates `clusterset.k8s.io` and `cluster.clusterset.k8s.io` when `--write-cluster-properties` is set. Otherwise, and with `karmada`, a cluster admin or manager |

The rows marked "MCS implementation" are not part of the discovery
service. Their `mcs` cells describe the standard implementation that
Dynamo ships in `deploy/global-discovery-mcs`.

### When Is a Pool Routable

A pool gets traffic only when all of these are true:

* `GET /pools` lists it,
* its state stream has delivered first state,
* and its readiness does not say "down" (DEP [#11225] readiness gate).

The heartbeat is the slow check. The state stream is the fast check: when
it drops, the pool stops getting traffic right away.

### The "No External Store" Rule

The HLD says "No etcd, database or shared cache is required for the Global
View". Registrations in memory need no store. Hybrid and MCS keep small
records in the hub's etcd, and DynamoDB is a database. We think this is
acceptable because:

* **Dynamo already does this inside one cluster.** It uses the Kubernetes
  API for worker discovery (the `DynamoWorkerMetadata` CRD). These modes do
  the same one level up: pools instead of workers.
* **Only small, slow data goes there.** Pool records change when a pool
  joins, leaves, or scales. All KV and load state stays in router memory
  and is rebuilt from the PoolRelays, as the HLD requires.

# Implementation

| PR | Title | State |
|---|---|---|
| [#17330] | feat(global-router): add a crate for types the Global Router components share | Merged |
| [#17208] | feat(router): add a Global Discovery Service client crate | Merged |
| [#17331] | feat(global-router): add the Global Discovery Service library (contract and backends) | In review |
| [#17371] | feat(global-router): add the Global Discovery HTTP service | Draft, on #17331 |
| [#17332] | feat(global-router): take MCS pool models from registrations, not a file | Draft, on #17371 |
| [#17333] | feat(global-router): reserve POST /routers until router registration is designed | In review, on #17371 |
| [#17412] | feat(operator): export Global Router pools through the Multi-Cluster Services API | Draft |
| [#17615] | feat(global-router): add the Multi-Cluster Services registry and controller (`mcs-registry` and `mcs-controller` in `deploy/global-discovery-mcs`, closes [#17601]) | Draft, on #17371 |

Not built yet: a PoolRelay that calls the client, router registration,
Kubernetes or DynamoDB storage, and caller checks.

# Alternate Solutions

## Alt 1: PoolRelays Register With Each Router Replica

The first version of this DEP had each PoolRelay call `RegisterPool` on the
Global Router itself ("gRPC Native"), and each router replica kept the pools
that called it.

**Reason Rejected:** a call through a load balancer reaches one replica.
So every PoolRelay needs every replica's address (for example, a load
balancer per replica), and a new replica cannot tell how many pools to wait
for. The discovery service keeps one list instead.

## Alt 2: The Hub Reaches Into Each Cluster (Hub/Spoke)

The hub holds a kubeconfig for every workload cluster and reads pools from
them. GKE's multi-cluster Inference Gateway works this way, using Google
IAM. SIG Multicluster's ClusterProfile KEP
([KEP-4322](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/4322-cluster-inventory))
recommends this direction, with cloud identity federation instead of
stored keys.

**Pros:** nothing to install in workload clusters. With identity
federation, no long-lived credentials are stored anywhere.

**Cons:** without identity federation, the hub holds credentials for every
cluster. Identity federation is easiest when all clusters are in one cloud.
Every workload cluster needs an inbound path from the hub.

**Reason Rejected:** it goes against the HLD's dial-out direction. It can
still fit single-cloud fleets.

## Alt 3: Static File Only

A list of pools in a file or ConfigMap. llm-d's multi-cluster router does
this, and so does the Global Router POC ([#15338]). [#17276] reads MCS pool
models from such a file; [#17332] takes them from registrations instead.

**Pros:** the simplest option.

**Cons:** a person must edit the file, and copy it to every cluster, for
every change.

**Reason Rejected:** pools cannot join or leave on their own. It is still a
fine start for tests.

# Open Questions

1. **Router registration.** How does a router prove it is a router: mesh
   mTLS (a SPIFFE ID), a token, or something else? What does it send, and
   is it a heartbeat with a lease? `POST /routers` is reserved for it.
2. **Caller checks.** V1 trusts the network. How is a pool bound to its
   PoolRelay later, and who issues PoolRelay certificates?
3. **The next registration storage:** Kubernetes (Hybrid) or DynamoDB
   global tables?
4. **Cluster names.** A PoolRelay's `--cluster-id` must equal its cluster's
   name in the ClusterSet, or MCS and registrations do not join (the
   service logs a warning). How do we keep them equal?
5. **Namespace sameness.** A DGD's namespace must have one owner in every
   cluster, the hub included. How do teams name namespaces across clusters?
6. **Export field.** The operator uses a DGD annotation for now. Should the
   DGD get a field?
7. **Upgrades.** Old PoolRelays must keep working with a new discovery
   service, and the reverse.
8. Only when GAIE is installed: should we also sync to GAIE's
   `InferencePoolImport`?
9. **Several PoolRelay pods.** A PoolRelay with more than one replica
   ([#17347]) registers the same PoolKey from each pod. This is a known
   ambiguity in the registration model: it does not say whether these are
   one registration or several, or whose fields win.

# References

* [Global Router HLD](https://docs.google.com/document/d/1FYKvlsEnc6aMXgU_61RwtUJP3LWY7sQIEhOXjCFM2sk/edit?tab=t.bjkujx8pnylf):
  "Global View" section and open questions.
* DEP [#11225]: Multi-DC KV-aware routing and the KV DC Relay.
* DEP [#15742]: PoolRelay naming and compatibility contract.
* DEP [#16027]: Operator-managed Pool Relay.
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

# Appendix: MCS Details

## Objects

| Object | Cluster | Created by | Read by | Purpose |
|---|---|---|---|---|
| `ServiceExport` for `<DGD name>-pool-relay` and `<DGD name>-frontend` | Workload | Dynamo operator | MCS implementation | Announces the pool and its frontends |
| Their imported EndpointSlices | Where the discovery service runs, or the Karmada control plane | MCS implementation | Global Discovery Service | Pool list, liveness, frontend addresses |
| `ServiceImport` for each exported Service | Every cluster (standard), or where an admin propagates it (Karmada) | `mcs-controller` (standard). An admin, with a `PropagationPolicy` (Karmada) | Not read by Dynamo | The Service as the ClusterSet sees it |
| Its derived Service | The same | `mcs-controller`, named `derived-<hash>` (standard). Karmada, named `derived-<name>` | Not read by Dynamo | The local Service for the import |
| `ServiceExport` for the Global Router Service | Hub | An admin, once | MCS implementation | Announces the router replicas |
| Its imported EndpointSlices | Where the discovery service runs | MCS implementation | Global Discovery Service | `GET /routers` |
| `ClusterProperty` `cluster.clusterset.k8s.io` | Every cluster | Cluster admin or manager, or `mcs-controller` with `--write-cluster-properties` | MCS implementation | The cluster's name in the ClusterSet |
| `ClusterProperty` `clusterset.k8s.io` | Every cluster | Cluster admin or manager, or `mcs-controller` with `--write-cluster-properties` | MCS implementation | The ClusterSet's name |

## Where Each Pool Field Comes From

| Field | Source |
|---|---|
| `cluster_id` | Standard MCS: the EndpointSlice's `multicluster.kubernetes.io/source-cluster` label. Karmada: the slice name, `imported-<cluster>-<slice>` |
| `namespace`, `dgd_name` | The PoolRelay Service's namespace, and its name without `-pool-relay` |
| Model, capabilities, capacity, location | The PoolRelay's registration |
| Frontends | The Ready endpoints of `<DGD name>-frontend` |
| `relay_identities` | The addresses of a Ready PoolRelay pod |

## Kubernetes APIs Used

* Workload cluster: the operator reads `DynamoGraphDeployment` and writes
  `ServiceExport` (`multicluster.x-k8s.io/v1beta1`, with `v1alpha1` also
  served by the `sigs.k8s.io/mcs-api` v0.5.2 CRDs; Karmada's own CRDs
  serve `v1alpha1` only). The operator ([#17412]) writes `v1alpha1`
  objects, which conversion accepts: the v0.5.2 CRDs store `v1beta1`.
* Every cluster, with the standard implementation: `mcs-controller` reads
  exported `Service`s and their endpoints, and writes `ServiceImport`, the
  `derived-<hash>` Service, `EndpointSlice`, the `ServiceExport`
  conditions, and, with `--write-cluster-properties`, `ClusterProperty`.
* Hub: the Global Discovery Service lists and watches `EndpointSlice`. It
  writes nothing.
* Every cluster: `ClusterProperty` (`about.k8s.io`), read by the MCS
  implementation.
* GAIE and Gateway API: optional only.

PoolRelays connect to each router replica at the address `GET /routers`
gives, not to a ClusterSet IP, which would send every connection to one
replica. The state stream protocol is out of scope.

[#11225]: https://github.com/ai-dynamo/dynamo/issues/11225
[#15338]: https://github.com/ai-dynamo/dynamo/pull/15338
[#15742]: https://github.com/ai-dynamo/dynamo/issues/15742
[#16027]: https://github.com/ai-dynamo/dynamo/issues/16027
[#17208]: https://github.com/ai-dynamo/dynamo/pull/17208
[#17276]: https://github.com/ai-dynamo/dynamo/pull/17276
[#17330]: https://github.com/ai-dynamo/dynamo/pull/17330
[#17331]: https://github.com/ai-dynamo/dynamo/pull/17331
[#17332]: https://github.com/ai-dynamo/dynamo/pull/17332
[#17333]: https://github.com/ai-dynamo/dynamo/pull/17333
[#17347]: https://github.com/ai-dynamo/dynamo/pull/17347
[#17371]: https://github.com/ai-dynamo/dynamo/pull/17371
[#17412]: https://github.com/ai-dynamo/dynamo/pull/17412
[#17601]: https://github.com/ai-dynamo/dynamo/issues/17601
[#17615]: https://github.com/ai-dynamo/dynamo/pull/17615
