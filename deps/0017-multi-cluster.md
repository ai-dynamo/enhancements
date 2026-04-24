# Multi-Cluster Deployments in Dynamo

**Status**: Draft

**Authors**: Anna Tchernych, Thomas Montfort

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**:

**Required Reviewers**:

**Review Date**:

**Pull Request**:

**Implementation PR / Tracking Issue**:

# Summary

This proposal describes how Dynamo can perform multi-cluster routing in a
manner that respects session affinity and is KV-cache-aware.

# Motivation

Increasingly, Dynamo customers such as Baseten, Prime Intellect, and Amazon
Ads are deploying models across Kubernetes clusters in different regions.
Traditional or simple off-the-shelf load-balancing solutions are suboptimal at
maximizing global cluster utilization and throughput and minimizing latency.

## Traditional Load-Balancing Solutions

### Region-Local Traffic Handling

Uses DNS GLB which always routes users to the closest datacenter.

**Pros:**

* KV-cache affinity naturally emerges because of sticky sessions to the same
  datacenter.
* No single point of failure.

**Cons:**

* No resource sharing: if one datacenter is overloaded while another has
  capacity to service requests, a user in the same region as the overloaded
  datacenter will still be routed there. The user would be serviced faster
  (lower latency) if routed to the other datacenter, even accounting for
  cross-region latency.
* DNS TTL can cause slow failover (Anycast IP routing overcomes this).
* Assumes that every model is served in every region.

### Global Load Balancer

Simple routing logic:

* **Least Loaded**: could be defined by `inflight_requests`, `queue_depth`,
  or `gpu_utilization`.
* **Consistent Hashing / User Pinning**: map users to the closest or least
  loaded datacenter and then pin requests to this datacenter for a set TTL.
  This is an easy way to take advantage of KV-cache affinity because prefix
  reuse is very high within a user. Consistent hashing provides a cleaner
  algorithm for when datacenters leave or join.

**Pros:**

* Easy to take advantage of KV-cache affinity due to sticky sessions.

**Cons:**

* With a single global LB ingress point, you introduce a multi-regional RTT
  latency for each request (up to 200 ms), which can significantly impact TTFT
  (prefill time is normally on the order of hundreds of milliseconds).
* Least Loaded does not account for KV-cache affinity, which can result in
  expending more resources and additional latency for a request. Consistent
  hashing, on the other hand, is not load-aware.

# Proposal

## Scale KV Router / Indexer to Multiple Regions

TODO
If the KV Router / Indexer can performantly operate over multiple regions
without limitations, this could provide the KV-/load-aware solution to solve
the problem.

**Concerns:**

* What are the current bottlenecks of the frontend / router?
  * Tokenization, sequence tracking, indexing
  * Networking and batching
* If we have a KV Router in each region, how can we discover workers in
  different clusters?

## Two-Tiered Routing

### Custom Cluster Picker via `ext_proc`

This approach is similar to how
[EPP](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/pkg/epp)
is designed. `ext_proc` lets you make a routing decision on every request based
on the latest metrics, providing per-request routing.

This approach also aspires to be consistent with the SIG-proposed
[Multi-Cluster Service APIs](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api).
There is currently no standard way to connect beyond the cluster boundary. The KEP proposes a new API to extend the
service concept across multiple clusters. GKE already has a custom implementation compliant with this approach.

The Cluster Picker will be a gRPC service similar to EPP but much simpler. We
only need to parse the headers and do not need to worry about streaming.

**Architecture:**

1. Deploy an Envoy-compatible load balancer in a dedicated tiny cluster (no
   GPU). This way, if one of the GPU clusters goes down, the LB still works.
2. In the same cluster, deploy a small `ext_proc` service written in Go. It
   will:
   1. Scrape the cluster-level aggregate metrics exposed by the FrontEnds in
      clusters A and B.
   2. Build an in-memory map of these metrics per cluster so that it remains
      stateless.
   3. On each request, receive the request headers, look at its in-memory
      metrics cache, pick the better cluster, and tell Envoy where to send
      it via a response header (e.g., `x-route-to-cluster: cluster-a`). It
      respects session affinity using headers:

      ```
      cluster = hash(request.header["x-session-id"]) % num_clusters
      ```

      and overrides affinity when the target cluster is overloaded. TTL-based
      affinity can also be used.

3. Envoy is configured with routes that match the header set by the Cluster
   Picker:

   ```yaml
   routes:
     - match:
         prefix: /v1
         headers:
           - name: x-route-to-cluster
             exact_match: cluster-a
       route:
         cluster: cluster-a-frontend

     - match:
         prefix: /v1
         headers:
           - name: x-route-to-cluster
             exact_match: cluster-b
       route:
         cluster: cluster-b-frontend
   ```

   Envoy sends the request to the appropriate cluster based on the header.

**Scaling:** Multiple replicas of the Cluster Picker are deployed. Each
replica builds the metrics map independently. If all replicas die, Envoy can
be configured to fall back to round-robin.

### Discovery

**Stage 0 — Hard-coded FrontEnd addresses.**

**Stage 1 — Static config with optional CRDs.** The Cluster Picker service
discovers FrontEnds via static configuration. If there are many clusters and
they are added/removed regularly, a new CRD `ServiceImport` can be
added/deleted in the gateway cluster. The Cluster Picker watches these CRDs
(it is not a controller) and discovers service clusters from them. This CRD
is suggested by the
[K8s SIG Multi-Cluster Services API](https://github.com/kubernetes-sigs/mcs-api).

The advantages:

* We can manually install `ServiceImport` to extract clusters from the
  `status.cluster` field.
* We can manually install `EndpointSlice` CRDs to extract endpoint IPs per
  `source-cluster` label.
* Later we can decide to add an mcs-controller (or deploy with GKE Fleet);
  everything already speaks the standard API. The Cluster Picker's code does
  not change (see Stage 2).

**Stage 2 — Comply with the full K8s Multi-Cluster API.** Google uses the
standard
[K8s Multi-Cluster Services API](https://github.com/kubernetes/enhancements/tree/master/keps/sig-multicluster/1645-multi-cluster-services-api)
(alpha implementation). For this, we would need to turn the Cluster Picker
into a controller or use existing open-source implementations.

### Extension to Reinforcement Learning

In this case, Dynamo deployments in different clusters become a rollout fleet.

**Assumptions:**

* Each Dynamo deployment has some sort of a weight updater.
* FrontEnd exposes new metrics:

  ```
  policy_version{cluster="a"} 142            # the version DGD is serving
  weight_update_in_progress{cluster="a"} 0   # whether weight update is in progress
  ```

The Cluster Picker algorithm is extended with the following steps on each
request:

1. **Read request headers**
   * `x-session-id` (for affinity)
   * `x-policy-version` (optional: trainer can request a specific version)
   * `x-rl-batch-id` (optional: for batch tracking)

2. **Check policy versions across clusters**
   * If the trainer requests a specific version: filter to clusters running
     that version (or close enough).
   * If no version is requested: all clusters are candidates.

3. **Check cluster health**
   * Filter out clusters in the middle of a weight update.
   * Filter out clusters with an unhealthy FrontEnd (`HTTP /health`).

4. **Apply session affinity**
   * If a session exists and the preferred cluster is healthy and running the
     right version: route to preferred.
   * Otherwise, fall through to load-based selection.

5. **Pick least loaded among remaining candidates**
   * Same as before: `inflight` + `queued` metrics.

6. **Set routing header**
   * `x-route-to-cluster: cluster-a`

**Strategies for the weight-update window** (when weights are being swapped):

* **Strategy 1 — Passive update.** The picker stops routing to clusters with
  `weight_update_in_progress = 1`. The problem is that if all clusters are
  undergoing an update, traffic cannot be sent.

* **Strategy 2 — Staggered rollout.** The Cluster Picker allows only one
  cluster to update at a time. This is slower:
  `total_time = N × per_cluster_update_time`. It can be implemented using a
  K8s ConfigMap deployed in the gateway cluster. The DGDs in the cluster would
  try to acquire the lock by patching it. The weight updaters in DGDs will
  need cross-cluster access (via ServiceAccount token).

* **Strategy 3 — Async RL.** The picker does not coordinate at all. The
  trainer does not stop and wait for the fleet to be on the right version. It
  continuously consumes rollouts and applies corrections for the fact that
  some were generated by a slightly older policy.

### Simplified Variant: Controller with HttpRoute

A simpler but less accurate alternative is to build a small controller and
deploy an `HttpRoute` resource. The controller scrapes metrics from the
FrontEnd and updates the weights in the `HttpRoute`. The global LB forwards
requests to each FrontEnd based on the weight distribution.

This solution is Kubernetes-native. The weight updates introduce a lag, and
routing is statistical (e.g., a 70/30 split applies across requests rather
than per-request). The advantage is that it works with any Gateway API
implementation, though in practice users would likely use an Envoy-based
gateway.

# Alternate Solutions

N/A
