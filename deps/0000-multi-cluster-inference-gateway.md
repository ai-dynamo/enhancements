# Multi-Cluster Routing with the Gateway API Inference Extension

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

Dynamo is adding routing across data centers
([DEP #11225](https://github.com/ai-dynamo/dynamo/issues/11225)).
Related work:

* A two-step Global Router
  ([roadmap #14124](https://github.com/ai-dynamo/dynamo/issues/14124),
  item 1.6). It first picks a pool, then that pool's router picks a
  worker. The same design works in one cluster or across many.
  Milestones are still being set.

* [Global Router proposal]( https://docs.google.com/document/d/1FYKvlsEnc6aMXgU_61RwtUJP3LWY7sQIEhOXjCFM2sk/edit?tab=t.bjkujx8pnylf)
On that Document the TODO is "K8s Native Global Router Solution: describes how Global Router fits into K8s Gateway Inference Extension"(under  LLD table lists). This DEP  is essentially that LLD.

* [DEP #11225](https://github.com/ai-dynamo/dynamo/issues/11225):
  Nikita Sukharev, a Software Engineer from @G-Core proposed the multi-DC Global Router. A draft design for a proxy reads the
  [KV DC Relay](https://github.com/ai-dynamo/dynamo/blob/main/docs/fern/pages/developer-guide/knowledge-base/modular-components/router/multi-dc-kv-routing.md).
  Only the Relay half is merged.

* [DEP #11403](https://github.com/ai-dynamo/dynamo/issues/11403):
  session-aware placement. Also a draft.

* Stats streams for the Relay
  ([PR #13187](https://github.com/ai-dynamo/dynamo/pull/13187)). Still
  open.

This DEP proposes how these users can
route requests across clusters into Dynamo deployments when using the [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/) .

# Motivation

Inside one cluster, Dynamo already works with GAIE. The gateway asks the
Dynamo EPP (Endpoint Picker) which worker should serve each request.

Across clusters, GAIE users have no Dynamo path yet:

* GAIE's own multi-cluster design
  ([InferencePoolImport](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/docs/proposals/1374-multi-cluster-inference))
  is still a draft, and not all gateways support it.

* Dynamo's multi-data-center routing uses its own proxy, not a gateway.

## Goals

* Route requests across clusters into Dynamo deployments through a GAIE
  gateway.

* Keep today's single-cluster setup unchanged.

# Proposal

The HLD doc suggest implementing the Global Router as a selector. So just like EPP it does not hold the stream (the FrontEnd does)
In the #11225 proposal the global router holds the stream, so this is not the pattern we will be adopting.
The difference between the local and the new global dynamo router is that the state will come from pool relays. 


Route in two steps:

1. At the entry gateway, a **Multi Cluster EPP** picks a pool. Region and DC are attributes of the pool. The "Hub EPP" makes the choice based ion the KV DC Relay service. The pool relay registers itself. 

2. In that pool (cluster), the gateway and the Dynamo EPP pick a worker, as they
   do today.

```
Client -> Gateway + hub EPP (picks a pool)
       -> Gateway + Dynamo EPP in that cluster (picks a worker)
       -> Worker
```

Both options below need the entry gateway to forward requests to another
cluster's gateway. We need to confirm that agentgateway and others support this.


## Gaps in design that this DEP will fulfill 

1. **Discovery.** The HLD does not say how discovery maps into Kubernetes
   1. How does the pool relay in cluster A learn about the global router
      address in cluster B?
      1. It seems that the [11225](https://github.com/ai-dynamo/dynamo/issues/11225)
         proposes that the Global Router calls the relays and asks them to send
         updates. (pull)
      2. HLD leans towards PUSH. Upon the start the relay calls the router to
         register itself.
      3. There could be a 3rd service with which the Global Router and Relays
         register.
   2. What creates a PoolRelay? (the operator?)
   3. The Regional Discovery Service implementation. Does this conflict with the
      no external store idea?
2. **EPP side**
   1. EPP returns `ip:port`. Cross-cluster destinations need the gateway-side
      support.
   2. Verify that GAIE EPP protocols enables retries by providing fallback
      endpoints.
   3. Make sure the tokenization is consistent across clusters
   4. Disag serving is a later milestone.


## Option A: Add a hub mode to the Dynamo Rust EPP

* The Dynamo EPP gets a new mode that picks a cluster.

* It reads each cluster's
  [KV DC Relay](https://github.com/ai-dynamo/dynamo/blob/main/docs/fern/pages/developer-guide/knowledge-base/modular-components/router/multi-dc-kv-routing.md),
  so it knows which cluster already has the prompt in its cache and how
  full each cluster is.

* It reuses the pool-selection code planned for Dynamo's Global Router
  ([roadmap](https://github.com/ai-dynamo/dynamo/issues/14124)) instead of
  duplicating it.

**Pros:** Dynamo controls more 

**Cons:** more work. Depends on the KV DC Relay, which is experimental.

## Option B: Use the llm-d hub EPP

* The llm-d EPP can already pick between clusters
  ([llm-d-router #2232](https://github.com/llm-d/llm-d-router/pull/2232)).

* The Dynamo EPP publishes two simple numbers per cluster for the hub to
  read: queue size and KV cache usage.

**Pros:** 

**Cons:** the hub only estimates cache hits from the requests it has sent.
The routing logic lives outside Dynamo.

The options can be combined: start with Option B, then add Option A if
tests show that it routes better.

# Alternate Solutions

N/A. Options A and B above are both still open.
