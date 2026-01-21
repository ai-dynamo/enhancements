# Rollout Support for DynamoGraphDeployments (DGDs)

**Status**: Under Review

**Authors**: Thomas Montfort

**Category**: Architecture

**Required Reviewers**: Biswa Panda, Neelay Shah, Maksim Khadkevich, Julien Mancuso, Itay Neeman, Rohan Varma

**Review Date**: January 14, 2026

**Pull Request**: https://github.com/ai-dynamo/enhancements/pull/49

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# 1. Summary

This enhancement proposal is meant to address the issues faced when performing a Rollout of a DGD.

A RollingUpdate refers to a specific type of Kubernetes resource Rollout where Pods owned by the resource are updated sequentially. A Rollout refers more broadly to the strategy in which a resource is updated.

To learn more about Rollout strategies, see the [Appendix](#appendix).

# 2. Scenarios

- **Update Dynamo Worker Version** - Updating the Dynamo framework runtime image (e.g., vllm-runtime:0.6.0 to 0.7.0) for a Worker service.
- **Update Dynamo Frontend Version** - Updating the Dynamo framework runtime image for the Frontend service.
- **Update Dynamo Worker Configuration** - Changing command, args, or environment variables passed to workers (e.g., block size, context length, KV cache connector).

# 3. Problems

These scenarios become problematic when old and new workers coexist during a RollingUpdate. The dynamo namespace groups workers together into a fungible pool, meaning the frontend and workers discover and communicate with each other regardless of version. This causes three distinct issues:

## 3.1 Different MDC Checksums

When configuration changes affect the ModelDeploymentCard (MDC) checksum (e.g., `--block-size 32`):

_Applies to both aggregated and disaggregated deployments._

```mermaid
sequenceDiagram
    participant OW as Old Workers
    participant NW as New Workers
    participant FE as Frontend
    participant CL as Client

    OW->>FE: Publish MDC (checksum A, block_size=16)
    FE->>FE: Store MDC for Model A

    Note over NW: Rolling update begins

    NW->>FE: Publish MDC (checksum B, block_size=32)
    FE--xFE: Discard! Engine exists for Model A with different checksum

    NW->>CL: Register (same EndpointId)
    Note over CL: Instance list now contains<br/>old AND new workers

    CL->>NW: Route request to new worker
    FE->>NW: Preprocess with OLD MDC (checksum A)

    Note over NW: ❌ block_size mismatch!<br/>Expected 32, got 16

    NW-->>CL: Request Error
```

_The current behavior silently discards the new MDC but the new workers are added to the client instance list. This results in new workers having requests preprocessed with the old MDC, which can result in request errors or unexpected behaviors._

_Why this behavior exists: The decision to discard conflicting MDCs was made at the runtime level to keep the logic simple—accepting only one MDC per model key avoided complexity around MDC versioning and selection. This works fine for initial deployments but breaks down during rolling updates when multiple versions coexist._

## 3.2 Incompatible KV Cache Transfer

When configuration changes affect KV cache transfer (e.g., `--tensor-parallel-size 2`):

_Applies to disaggregated deployments only._

```mermaid
sequenceDiagram
    participant OP as Old Prefill (TP=1)
    participant OD as Old Decode (TP=1)
    participant NP as New Prefill (TP=2)
    participant CL as Client

    OP->>CL: Register Prefill instance
    OD->>CL: Register Decode instance
    Note over CL: Instance list: [Old Prefill, Old Decode]

    Note over NP: Rolling update begins

    NP->>CL: Register Prefill instance
    Note over CL: Instance list: [Old Prefill, Old Decode, New Prefill]

    CL->>NP: Route prefill request to New Prefill
    NP->>NP: Complete prefill (TP=2 KV layout)

    NP->>OD: Transfer KV cache
    Note over OD: ❌ KV cache layout mismatch!<br/>Expected TP=1, got TP=2

    OD-->>CL: Communication Error
```

_The discovery of old prefill/decode workers by the new set of workers can result in KV block shape incompatibilities, KV transfer API breaking changes, etc. that result in request errors or unexpected behaviors._

## 3.3 Runtime API Incompatibilities

When upgrading the Dynamo framework runtime image (e.g., vllm-runtime:0.6.0 to 0.7.0), the internal APIs between frontend and workers may have breaking changes. During a RollingUpdate, an old frontend may route requests to new workers (or vice versa) using incompatible API versions. This can manifest as serialization errors, missing fields, or protocol mismatches that cause request failures.

_Applies to both aggregated and disaggregated deployments._

# 4. Solution: Isolate New/Old Worker Deployments

Currently during an in-place RollingUpdate, new and old workers coexist in the same dynamo namespace sharing the same frontend. The dynamo namespace groups these workers together, making them a fungible pool of workers. This results in the MDC checksum and KV cache incompatibility issues described above.

```mermaid
flowchart LR
    Client([Client])

    subgraph ns[" "]
        direction TB
        ns_label["<b>Dynamo Namespace A</b>"]

        FE[Frontend]

        subgraph model[" "]
            direction TB
            model_label["<b>Model A</b>"]

            subgraph old[" "]
                direction TB
                old_label["OLD WORKERS (config v1)"]
                OP1[Prefill-0]
                OP2[Prefill-1]
                OD1[Decode-0]
            end

            subgraph new[" "]
                direction TB
                new_label["NEW WORKERS (config v2)"]
                NP1[Prefill-2]
                ND1[Decode-1]
            end
        end
    end

    Client --> FE
    FE -.->|discovers| OP1
    FE -.->|discovers| OP2
    FE -.->|discovers| OD1
    FE -.->|discovers| NP1
    FE -.->|discovers| ND1

    style ns_label fill:none,stroke:none
    style model_label fill:none,stroke:none
    style old_label fill:#fdd,stroke:none,color:#933
    style new_label fill:#dfd,stroke:none,color:#393
    style old fill:#fdd,stroke:#c99
    style new fill:#dfd,stroke:#9c9
    style model fill:#eef,stroke:#99c
    style ns fill:#f8f8ff,stroke:#669
```

The proposed solution is to have the new and old worker deployments to be in separate dynamo namespaces, with isolated frontends.

```mermaid
flowchart LR
    Client([Client])

    subgraph ns_old[" "]
        direction TB
        ns_old_label["<b>Dynamo Namespace A</b>"]

        FE_old[Frontend]

        subgraph model_old[" "]
            direction TB
            model_old_label["<b>Model A</b>"]

            subgraph old[" "]
                direction TB
                old_label["OLD WORKERS (config v1)"]
                OP1[Prefill-0]
                OP2[Prefill-1]
                OD1[Decode-0]
            end
        end
    end

    subgraph ns_new[" "]
        direction TB
        ns_new_label["<b>Dynamo Namespace B</b>"]

        FE_new[Frontend]

        subgraph model_new[" "]
            direction TB
            model_new_label["<b>Model A</b>"]

            subgraph new[" "]
                direction TB
                new_label["NEW WORKERS (config v2)"]
                NP1[Prefill-0]
                NP2[Prefill-1]
                ND1[Decode-0]
            end
        end
    end

    Client --> FE_old
    Client --> FE_new
    FE_old -.->|discovers| OP1
    FE_old -.->|discovers| OP2
    FE_old -.->|discovers| OD1
    FE_new -.->|discovers| NP1
    FE_new -.->|discovers| NP2
    FE_new -.->|discovers| ND1

    style ns_old_label fill:none,stroke:none
    style ns_new_label fill:none,stroke:none
    style model_old_label fill:none,stroke:none
    style model_new_label fill:none,stroke:none
    style old_label fill:#fdd,stroke:none,color:#933
    style new_label fill:#dfd,stroke:none,color:#393
    style old fill:#fdd,stroke:#c99
    style new fill:#dfd,stroke:#9c9
    style model_old fill:#eef,stroke:#99c
    style model_new fill:#eef,stroke:#99c
    style ns_old fill:#f8f8ff,stroke:#669
    style ns_new fill:#f8f8ff,stroke:#669
```

**Solves**:

- **Different MDC Checksums**: having local frontends to new and old workers ensures that for the Model A, only one MDC checksum is being stored and used.
- **Incompatible KV Cache Transfer**: new and old workers do not discover each other since they are in separate dynamo namespaces.
- **Runtime API Incompatibilities**: frontends and workers within each isolated namespace are always at the same version, preventing cross-version API mismatches.

**Pros**:

- No need to modify the frontend logic, this would be purely handled via the Kubernetes controller.

## 4.1 Case A: Traditional Frontend and Worker Deployment

This is the most common case where a DGD is defined with a Frontend service and either an aggregated worker service or prefill and decode worker services.

For this example, there are 3 frontend replicas, 4 prefill workers, and 2 decode workers.

### 4.1.1 Solution A: Modify Worker Resource In-Place

In this solution, the controller will do the following:

- Create a new frontend resource (Deployment/PodClique), matching the replica count of the old frontend resource, with a new DYN_NAMESPACE B.
- Wait until at least one of, or all of, the new frontend replicas are ready.
- Apply user-defined updates to the worker resource (Deployment/PodClique/PodCliqueScalingGroup/LWS) along with updating the controller-determined DYN_NAMESPACE to B.
- This will trigger the worker resource's underlying RollingUpdate.
- As new workers are ready in DYN_NAMESPACE B, they will be registered with the new frontend resource in DYN_NAMESPACE B, and old workers will be deregistered from the old frontend resource in DYN_NAMESPACE A as they are terminated.

```mermaid
sequenceDiagram
    participant U as User
    participant C as Controller
    participant FE_old as Old Frontend<br/>(NS A)
    participant FE_new as New Frontend<br/>(NS B)
    participant W as Workers

    Note over FE_old,W: Initial State: Frontend + Workers in NS A

    U->>C: T0: Modify DGD worker config<br/>(triggers RollingUpdate)

    C->>FE_new: T1: Create new Frontend resource<br/>(3 replicas, NS B)

    FE_new-->>C: T2: Frontend replica(s) ready

    C->>W: T3: Update worker resources<br/>(set DYN_NAMESPACE = NS B)

    Note over W: T4: RollingUpdate in progress<br/>Workers transition from NS A → NS B

    W-->>C: T4: RollingUpdate complete

    C->>FE_old: T5: Delete old Frontend resource

    Note over FE_new,W: Final State: Frontend + Workers in NS B
```

**Pros:**

- Logic is much simpler. Controller just needs to account for new/old frontend resource groups. Takes advantage of the underlying worker resource's rolling update mechanism.

**Cons:**

- Adding configurability to RollingUpdate behavior in the future is beholden to the underlying worker resource. For instance, maxUnavailable, maxSurge, partition do not have full support within Grove but do in Deployments/LWS.
- The default RollingUpdate behavior differs between Deployment/LWS and PodClique/PodCliqueScalingGroup.
- Having pause functionality for the rolling update relies on the underlying worker resource supporting partition/controller revision.
- Ensuring maintenance of P/D ratio during the rolling update is not possible.

### 4.1.2 Solution B: Controller Manages New/Old Worker Resource

_Assumes default RollingUpdate behavior where maxSurge is 1 and maxUnavailable is 0_

In this solution, the controller will do the following:

- The same as Solution A, create a new frontend resource with a new DYN_NAMESPACE B.
- Instead of modifying the worker resources in-place, the controller will create new worker resources with each set to 1 replica.
- The controller will then iteratively scale down the old worker resource by 1 and scale up the new worker resource by 1 until the old worker resource has scaled to 0 and the new worker resource has scaled to the desired replica count.
  _The rate of scaling up and down could be exposed as a part of the DGD CRD_

```mermaid
sequenceDiagram
    participant U as User
    participant C as Controller
    participant FE_old as Old Frontend<br/>(NS A)
    participant FE_new as New Frontend<br/>(NS B)
    participant W_old as Old Workers<br/>(NS A)
    participant W_new as New Workers<br/>(NS B)

    Note over FE_old,W_old: Initial State: Frontend + Workers in NS A<br/>(Prefill: 4, Decode: 2)

    U->>C: T0: Modify DGD worker config<br/>(triggers RollingUpdate)

    C->>FE_new: T1: Create new Frontend resource<br/>(3 replicas, NS B)

    FE_new-->>C: T2: Frontend replica(s) ready

    C->>W_new: T3: Create new worker resources<br/>(Prefill: 1, Decode: 1)

    loop T4: Repeat until old replicas = 0, new replicas = desired
        W_new-->>C: New worker replica ready
        C->>W_old: Scale down old resource by 1
        C->>W_new: Scale up new resource by 1
    end

    Note over W_new: T4 complete:<br/>Prefill: 4, Decode: 2 in NS B

    C->>FE_old: T5: Delete old Frontend resource

    Note over FE_new,W_new: Final State: Frontend + Workers in NS B
```

**Pros:**

- Can expose an API for configuring things such as maxUnavailable, maxSurge
- Can maintain P/D ratio during the rolling update due to the controller's ability to synchronize the rolling update between old/new and prefill/decode.
- Can enable pause functionality for the rolling update easily.

**Cons:**

- Logic becomes more complex. Controller needs to account for new/old worker resources. Controller needs to synchronize the rolling update between old/new and prefill/decode.

### 4.1.3 Solution Selection

**Solution B** where Controller Manages New/Old Worker Resource is the preferred solution as it provides maximum flexibility to support the following:

- Configurability of the RollingUpdate behavior such as maxSurge, maxUnavailable, partition.
- P/D ratio maintenance during the RollingUpdate.
- Pause functionality for the RollingUpdate.

Furthermore, it will enable consistent behavior regardless of if the worker underlying resource is a Deployment, PodClique, PodCliqueScalingGroup, or LWS.

## 4.2 Case B: Shared Frontend Deployment

The problem here is that at the DGD level for one of the model deployments, the DGD will only contain workers. It doesn't have the ability to create a new frontend resource with a new DYN_NAMESPACE.

```mermaid
flowchart LR
    Client([Client])

    subgraph dgd_fe[" "]
        direction TB
        dgd_fe_label["<b>DGD: Global Frontend</b>"]

        subgraph ns_global[" "]
            direction TB
            ns_global_label["<b>Dynamo NS: dynamo</b><br/>(discovers all namespaces)"]
            FE[Frontend]
        end
    end

    subgraph dgd_a[" "]
        direction TB
        dgd_a_label["<b>DGD: Model A Workers</b>"]

        subgraph ns_a[" "]
            direction TB
            ns_a_label["<b>Dynamo NS A</b>"]

            subgraph model_a[" "]
                direction TB
                model_a_label["<b>Model A</b>"]
                PA1[Prefill-0]
                PA2[Prefill-1]
                DA1[Decode-0]
            end
        end
    end

    subgraph dgd_b[" "]
        direction TB
        dgd_b_label["<b>DGD: Model B Workers</b>"]

        subgraph ns_b[" "]
            direction TB
            ns_b_label["<b>Dynamo NS B</b>"]

            subgraph model_b[" "]
                direction TB
                model_b_label["<b>Model B</b>"]
                PB1[Prefill-0]
                PB2[Prefill-1]
                DB1[Decode-0]
            end
        end
    end

    Client --> FE
    FE -.->|discovers| PA1
    FE -.->|discovers| PA2
    FE -.->|discovers| DA1
    FE -.->|discovers| PB1
    FE -.->|discovers| PB2
    FE -.->|discovers| DB1

    style dgd_fe_label fill:none,stroke:none
    style dgd_a_label fill:none,stroke:none
    style dgd_b_label fill:none,stroke:none
    style ns_global_label fill:none,stroke:none
    style ns_a_label fill:none,stroke:none
    style ns_b_label fill:none,stroke:none
    style model_a_label fill:none,stroke:none
    style model_b_label fill:none,stroke:none
    style dgd_fe fill:#fff8e8,stroke:#c96
    style dgd_a fill:#fff8e8,stroke:#c96
    style dgd_b fill:#fff8e8,stroke:#c96
    style ns_global fill:#f8f8ff,stroke:#669
    style ns_a fill:#f8f8ff,stroke:#669
    style ns_b fill:#f8f8ff,stroke:#669
    style model_a fill:#eef,stroke:#99c
    style model_b fill:#eef,stroke:#99c
```

_The global frontend uses the special `dynamo` namespace keyword which allows it to discover workers and MDCs from all dynamo namespaces. Each model's workers are deployed in separate DGDs with their own unique dynamo namespace._

For rolling the workers, either **4.1.1 Solution A: Modify Worker Resource In-Place** or **4.1.2 Solution B: Controller Manages New/Old Worker Resource** can be used.

The problem with the shared frontend is that if we do a RollingUpdate of a DGD for Model A, the shared frontend would need to be able to support a router client for Model A in dynamo namespace X and Model A in dynamo namespace Y, with potentially differing MDC checksums, and load balance between the two clients.

**We should remove the shared frontend pattern. It creates additional complexity to support multiple MDCs per model key and load balancing effectively between them.**

## 4.3 Load Balancing

During the RollingUpdate, we will essentially have two deployments, each with a set of frontends, that need to be load balanced across. The question is what will effectively handle load balancing between these two sets of frontends? Naively configuring a Kubernetes Service for both the new and old set of frontends will result in a 50/50 split of traffic which does not represent the shifting ratio of old/new workers.

### 4.3.1 Solution A: Gateway Load Balancing

In order to load balance between the two frontends, the DGD controller will manage a top-level Gateway with an HTTPRoute that's dynamically updated based on the ratio of old/new workers.

#### How It Works

The controller maintains a Gateway resource that acts as the single entry point for client traffic. During a RollingUpdate, the controller watches the worker replica counts in both the old and new deployments and dynamically adjusts the HTTPRoute backend weights to reflect the actual capacity. As new workers come online and old workers are terminated, the traffic split shifts proportionally until 100% of traffic flows to the new deployment.

```mermaid
sequenceDiagram
    participant Client
    participant C as DGD Controller
    participant GW as Gateway
    participant HR as HTTPRoute
    participant FE_old as Old Frontend<br/>(NS A, 4 workers)
    participant FE_new as New Frontend<br/>(NS B, 0 workers)

    Note over GW,FE_old: Initial State: All traffic to NS A

    Client->>GW: Request
    GW->>HR: Route lookup
    HR->>FE_old: weight: 100%
    FE_old-->>Client: Response

    Note over C: User triggers RollingUpdate

    C->>FE_new: Create new Frontend (NS B)
    C->>HR: Update weights<br/>(A: 100%, B: 0%)

    Note over C: New workers scaling up, old scaling down

    C->>C: Watch worker counts<br/>(NS A: 3, NS B: 1)
    C->>HR: Update weights<br/>(A: 75%, B: 25%)

    Client->>GW: Request
    GW->>HR: Route lookup
    HR->>FE_new: 25% chance
    FE_new-->>Client: Response

    C->>C: Watch worker counts<br/>(NS A: 2, NS B: 2)
    C->>HR: Update weights<br/>(A: 50%, B: 50%)

    C->>C: Watch worker counts<br/>(NS A: 1, NS B: 3)
    C->>HR: Update weights<br/>(A: 25%, B: 75%)

    C->>C: Watch worker counts<br/>(NS A: 0, NS B: 4)
    C->>HR: Update weights<br/>(A: 0%, B: 100%)

    Note over C: RollingUpdate complete

    C->>FE_old: Delete old Frontend
    C->>HR: Remove old backend

    Note over GW,FE_new: Final State: All traffic to NS B

    Client->>GW: Request
    GW->>HR: Route lookup
    HR->>FE_new: weight: 100%
    FE_new-->>Client: Response
```

**Key Controller Logic:**

1. **Watch Loop**: Controller continuously monitors worker replica counts across both namespaces
2. **Weight Calculation**: `weight_new = new_ready_workers / (old_ready_workers + new_ready_workers)`
3. **HTTPRoute Update**: Controller patches the HTTPRoute `backendRefs` with calculated weights
4. **Cleanup**: Once old workers reach 0, controller removes old frontend and its backend reference

**Pros:**

- No changes needed at the runtime level. Just orchestration and resource management by the controller.
- Simple to implement.

**Cons:**

- Controller will now need to manage a Gateway resource. Will need to provide configurability of the Gateway resource via the DGD API (or another CRD).
- Will either need to:
  - Always have a Gateway resource to manage in front of a frontend. This introduces an extra hop. OR
  - Allow user to enable/disable Gateway. However, if Gateway is disabled, we'd prevent in-place RollingUpdates.
- Introduces additional dependencies for the Dynamo Kubernetes platform:
  - Gateway API
  - Gateway controller installed (Istio, Envoy Gateway, etc.)

### 4.3.2 Solution B: Frontend Proxy Based Load Balancing

In this solution, the old frontend deployment will receive all of the traffic. The old frontend deployment will then proxy a weighted percentage of the requests to the new frontend deployment.

The two main issues:

- **Discovery**: How does the old frontend deployment discover the new frontend deployment and route to it?
- **Proxying**: Where/how is the traffic being proxied from the old frontend(s) to the new frontend(s)?

#### 4.3.2.1 Discovery Option A: New DynamoUpdateConfig CRD

The controller can create a DynamoUpdateConfig CR that contains the new frontend service URL and traffic split percentage.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoUpdateConfig
metadata:
  name: my-dgd-update
  namespace: default
spec:
  # Target frontend service for traffic proxying
  newFrontendService: "frontend-new.default.svc.cluster.local:8000"

  # Traffic split percentage (0-100)
  trafficSplitPercent: 25

  # Dynamo namespace for the new deployment
  newDynamoNamespace: "dynamo-v2"

  # Source and target deployment info
  sourceDeployment: "dgd-old"
  targetDeployment: "dgd-new"
```

The old frontend(s) watch this resource and can then configure proxying at the HTTP layer.

**Pros:**

- Simple to implement. Controller has complete information of the new frontend service URL, amount of workers to determine the traffic split ratio.

**Cons:**

- Introducing a new CRD.

#### 4.3.2.2 Discovery Option B: Leverage the DynamoWorkerMetadata CRD

The new frontend deployment replicas will each publish a DWM that contains the amount of workers they have and models being served.

**TODO:** Need to evaluate the feasibility of this approach.

#### 4.3.2.3 Proxying Option A: HTTP Server Layer

At the Rust HTTP server layer, we can configure a handler to proxy requests to the new frontend deployment. The difficulty is that LLM inference is mainly SSE streaming, so the proxy implementation is more involved in this case.

#### 4.3.2.4 Proxying Option B: Frontend Envoy Sidecar

Each of the Frontend Pods would have a sidecar Envoy container that is configured to proxy requests to the new frontend deployment.

This bypasses the need for discoverability because the ConfigMap that the controller creates for the Envoy sidecar container will embed the traffic split and new frontend service URL.

### 4.3.3 Solution Selection

**Solution A** with Gateway Load Balancing is the preferred solution. While it introduces additional dependencies, the implementation and separation of concerns is vastly simpler. Having a decentralized frontend approach requires adding discovery logic and potentially hand-rolling proxying for SSE streaming within the Dynamo runtime itself. Envoy Gateway is a tried and true solution for proxying and load balancing.

### Future Considerations

- Adding ControllerRevisions for the ability to pause a deployment and rollback.
- Adding configurability to the RollingUpdate behavior such as maxSurge, maxUnavailable, partition.
- Coordinating the rollout of the prefill and decode workers to maintain the configured P/D ratio.

# Appendix

## 1 Rollout Strategies in K8s AI Model Deployment Ecosystem

### 1.1 Deployments

The core Kubernetes `Deployment` resource has [two strategies for rolling out updates:](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) `RollingUpdate` and `Recreate`.

- `RollingUpdate` is the default strategy where Pods are updated by gradually scaling down the old `ReplicaSet` and scaling up the new `ReplicaSet`.
  - Optional field `maxUnavailable` specifies the maximum number of Pods that can be unavailable during the update.
  - Optional field `maxSurge` specifies the maximum number of Pods that can be created above the desired number of Pods.
- `Recreate` strategy will delete all existing Pods before the new ones are created.

### 1.2 StatefulSets

The core [Kubernetes `StatefulSet` resource](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/stateful-set-v1/) has two `updateStrategy` options: `RollingUpdate` and `OnDelete`.

- `RollingUpdate` is the default strategy where Pods are updated in sequence (highest index -> lowest index) one Pod at a time.
  - Optional field `maxUnavailable` specifies the maximum number of Pods that can be unavailable during the update (default is 1).
  - Optional field `partition` indicates the ordinal at which the StatefulSet should be partitioned for updates (default is 0 - all Pods are updated).
- `OnDelete` is the legacy behavior where Pods are only updated when they are manually deleted.

_Note that there is no `maxSurge` for `RollingUpdate` strategy as stable Pod identity (index) is important for exclusive PVCs and deterministic startup/shutdown ordering_

### 1.3 LeaderWorkerSet

The `LeaderWorkerSet` resource has a single [`RolloutStrategy` `RollingUpdate`](https://github.com/kubernetes-sigs/lws/blob/main/api/leaderworkerset/v1/leaderworkerset_types.go)

- Optional field `partition` behaves the same as StatefulSet `RollingUpdate` `partition` field.
- Optional field `maxUnavailable` behaves the same as StatefulSet `RollingUpdate` `maxUnavailable` field.
- Optional field `maxSurge` behaves the same as Deployment `RollingUpdate` `maxSurge` field.

_Note that LWS does support `maxSurge`. Example of how it works [here](https://lws.sigs.k8s.io/docs/concepts/rollout-strategy/)_

### 1.4 Gateway API Inference Extension

The Gateway API Inference Extension (GAIE) does not control the `Model Server` itself (the piece that is actually running the model inference). Example of a [vllm Model Server Deployment](https://github.com/kubernetes-sigs/gateway-api-inference-extension/blob/main/config/manifests/vllm/gpu-deployment.yaml)

Since GAIE is just a Gateway that is routing between `InferencePool`s (`Model Server` grouping for a single model), they [expect the user to create another `Model Server` Deployment themselves and use an `HTTPRoute` to split traffic between the new and old `InferencePool`](https://gateway-api-inference-extension.sigs.k8s.io/guides/inferencepool-rollout/). This enables canary or blue/green rollout strategies but the way the `Model Server` is rolled out for updates is on the user.

_Note that llm-d uses GAIE and Deployment (single-node) and LeaderWorkerSet (multi-node) for model serving so it inherits both of their rollout strategies._

### 1.5 CentML Platform

The CentML Platform leverages [`Argo Rollouts`](https://argo-rollouts.readthedocs.io/en/stable/) to enable an [automated canary rollout strategy](https://github.com/CentML/platform/blob/main/catalog/src/catalog/inference_deployment_v3/templates/rollout.yaml).

Argo Rollouts are designed to be a drop in replacement of `Deployment` where the PodSpec is the same but the `.spec.strategy` has more powerful capabilities.

- An example is a canary style rollout where you can define an [`Analysis`](https://argo-rollouts.readthedocs.io/en/stable/features/analysis/) that can query custom metrics and define success/failure conditions, the step size, traffic weighting and rollout speed

_Future inspiration for how a user can perform a canary rollout that monitors SLOs (TTFT, ITL) of new set of workers during rollout._

### 1.6 Grove Rolling Update Strategy

The Dynamo Kubernetes Operator leverages the [Grove API](https://github.com/ai-dynamo/grove/tree/main) for gang-scheduling and topology awareness. It relies on Grove for the underlying rollout mechanism of `DGD` resources.

### 1.6.1 Single-Node Deployment

[Single-Node Deployment Grove Example](./dynamo-grove-single-node.png)

- `Frontend`, `Decode` and `Prefill` workers are each `PodClique` (`PC`) resources.
- Each `PC` follows a `RollingUpdate` rollout strategy, where it deletes a `Pod` index and recreates, waits until the new `Pod` is `Ready` before proceeding to the next `Pod` index.

### 1.6.2 Multi-Node Deployment

[Multi-Node Deployment Grove Example](./dynamo-grove-multi-node.png)

- `Frontend` is a `PC`, and `Decode` and `Prefill` workers are multiple `PC` resources managed by a `PodCliqueScalingGroup` (`PCSG`) (similar to `LeaderWorkerSet` group of Pods).
- Each `PCSG` also follows a `RollingUpdate` rollout strategy, where instead of deleting/recreating a single `Pod` it's deleting/recreating a single `PCSG` replica

**Notes**:

- This is the default rolling update strategy for Grove. There currently is no way to configure the rolling update strategy for a `PCSG` for things like `maxSurge`, `maxUnavailable`. Discussion [here](https://github.com/ai-dynamo/grove/issues/212) on supporting additional rolling update configuration/strategies.
- Grove's rollout strategy would be the same as `Deployment` `RollingUpdate` where `maxSurge` is 0 and `maxUnavailable` is 1.
- Need multiple replicas for standalone `PC` or `PCSG` to ensure availability during the rollout process.

### 1.7 AIBrix

### 1.8 OME

### 1.9 Canary Rollout

### 1.10 Blue/Green Rollout

