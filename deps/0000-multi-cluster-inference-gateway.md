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

Route in two steps:

1. At the entry gateway, a **hub EPP** picks a cluster.

2. In that cluster, the gateway and the Dynamo EPP pick a worker, as they
   do today.

```
Client -> Gateway + hub EPP (picks a cluster)
       -> Gateway + Dynamo EPP in that cluster (picks a worker)
       -> Worker
```

Both options below need the entry gateway to forward requests to another
cluster's gateway. We need to confirm that agentgateway and others support this.

There are two ways to build the hub EPP.

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
