# Background

This enhancement proposal is meant to address two things:

1. Building on @biswapanda [Dynamo namespaces based isolation DEP](https://github.com/ai-dynamo/enhancements/pull/28) to propose a model worker group isolation mechanism within a DynamoGraphDeployment (DGD).
2. Proposing enhancements to Dynamo/Grove's roll out scenario/strategy support for DGDs.

## Model Deployment Cards (MDC)

In Dynamo, each model is represented by a [**ModelDeploymentCard (MDC)**](https://github.com/ai-dynamo/dynamo/blob/0028cdf43dd583356139669f33aecbb743fd13be/lib/llm/src/model_card.rs#L165). The MDC comprises of the model name (display name), tokenizer config, context length, model input/type specification, among other fields that defines the model inference behavior.

When a model worker starts up, it publishes its MDC to the KvStore with the key `v1/mdc/{namespace}.{component}.{endpoint}/{instance_id}`.

The Frontend has a `ModelWatcher` that watches the KvStore PUT events for keys with the prefix `v1/mdc`.

The `ModelWatcher.watch` method first determines if the [Frontend should process the MDC or discard it](https://github.com/ai-dynamo/dynamo/blob/6deeecb1d6a9f4eb1770b4272bfa85a4b6226e0a/lib/llm/src/discovery/watcher.rs#L138). If the frontend is deployed in the global namespace (`dynamo`), then all MDCs observed will be processed. If the Frontend is deployed in a non-global namespace, than only MDCs with keys that match the namespace will be processed (`v1/mdc/{namespace}.*`)

If the MDC should be processed, `ModelWatcher` will extract the model name (display name) from the MDC and check if the `ModelManager` already contains an engine for that model. If there is an engine that exists but the MDC checksum (generated from select MDC fields) does not match, [the `ModelWatcher` will discard the instance and log an error](https://github.com/ai-dynamo/dynamo/blob/6deeecb1d6a9f4eb1770b4272bfa85a4b6226e0a/lib/llm/src/discovery/watcher.rs#L156). Currently, the worker with discarded MDC will continue to run and actually receives traffic (registered under the same `EndpointId`) but frontend pre-processing will use the old MDC (undesirable). If there is no engine for that model, an engine is created and stored on the `ModelManager`.

The model engine contains a `Client` constructed for the MDC `EndpointId`. The MDC `EndpointId` is extracted from the MDC key (`{namespace}.{component}.{endpoint}`), which is the address of the model worker. This `Client` will watch the KvStore PUT events for keys with the prefix `v1/instance/{namespace}/{component}/{endpoint}`. When a PUT event is received, the `Client` will update the instance list.

### DynamoGraphDeployment (DGD) interaction with MDC

A **DynamoGraphDeployment (DGD)**, is the Dynamo Kubernetes CRD for deploying a distributed inference graph using the Dynamo distributed runtime and pre-built components.

The CRD spec contains a list of services which define the components that make up an inference graph. For instance, in the [vllm `disagg.yaml` file](https://github.com/ai-dynamo/dynamo/blob/main/components/backends/vllm/deploy/disagg.yaml), the services are: `Frontend`, `VllmDecodeWorker`, and `VllmPrefillWorker`. Each component has a field `dynamoNamespace` which defines the `namespace` the component will be logically grouped in. The `component` and `endpoint` parts of the worker `EndpointId` are not derived from the DGD but rather statically defined within the backend component itself. For instance, the `VllmDecodeWorker` component `EndpointId` has `component` part `backend` and `endpoint` part `generate`. `VllmPrefillWorker` component `EndpointId` has `component` part `prefill` and `endpoint` part `generate`.

## Model Deployment Roll Outs

A model deployment roll out is the process in which the deployment of a model is updated in production. Different roll out scenarios are listed below. For the purposes of this proposal, we will only focus on how these out scenarios occur via updates to a DGD.

### Common Roll Out Scenarios

#### Update Dynamo Worker Version

##### Use Cases

- Bug/vulnerability fix in newer version of Dynamo framework runtime
- New feature in the Dynamo framework runtime that the user wants to leverage
- Performance improvement in newer version of Dynamo framework runtime
- Changing the Dynamo framework that is being used (i.e. vllm -> sglang)

##### Example

An example of a how a user would update the Dynamo framework runtime version from `0.6.0` to `0.7.0`.

1. User has already deployed the initial vllm `disagg.yaml` DGD manifest with the image for each service set to `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0`.
2. The user updates the DGD manifest to use the image `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.7.0` for `VllmDecodeWorker` and `VllmPrefillWorker`.
3. User applies the updated DGD manifest.

##### Expected Behavior

- In-flight requests will be properly served (through graceful shutdown or request migration)
- Complete uptime of the model inference deployment is maintained

#### Update Frontend Worker Version

Same use cases, example and expected behavior as **Update Dynamo Worker Version**. Instead of updating the image for the workers, the user would update the image for the `Frontend` service.

#### Different MDC Checksums during Roll Out

As explained in the **Model Deployment Cards (MDC)** section above, the MDC is published from the model worker on startup. A DGD does not directly specify an MDC (not directly apparent to Operator/User how changes to `extraPodSpec` updates the MDC - would have to look at KvStore).

For any of the following use cases listed below, a user would want to update their model deployment via their DGD manifest.

##### Use Cases

- Update model revision
- Update model weight configuration such as quantization
- Update model context length
- Update tokenizer config
- Update chat template
- ...

##### Example

An example of a how a user would update the vllm `disagg.yaml` DGD manifest to update the `block_size` for the `VllmDecodeWorker` and `VllmPrefillWorker`.

1. User applies the initial vllm `disagg.yaml` DGD manifest.
2. User updates the DGD manifest by adding the flag `--block-size 32` in the `args` array in the `extraPodSpec` for the `VllmDecodeWorker` and `VllmPrefillWorker`.
3. User applies the updated DGD manifest.
4. The new Worker Pods will publish the new MDC with the updated `block_size` configuration, resulting in a different MDC checksum than the old Workers.

##### Expected Behavior

- In-flight requests will be properly served (through graceful shutdown or request migration)
- Complete uptime of the model inference deployment is maintained

#### Incompatible KV Cache Transfer Mechanisms during Roll Out

Another scenario a user might encounter is with a disaggregated deployment where the user wants to update their prefill/decode worker configurations that results in an incompatible KV cache transfer mechanism. For instance, the user might want to switch from NIXL to LMCache. It's important to note that KV cache transfer configuration is not tracked as a part of the MDC checksum.

##### Example

An example of a how a user would update the vllm `disagg.yaml` DGD manifest to switch from the default NIXL connector to the LMCache connector for the `VllmDecodeWorker` and `VllmPrefillWorker`.

1. User applies the initial vllm `disagg.yaml` DGD manifest.
2. User updates the DGD manifest by specifying the flag `--connector lmcache` in the `args` array in the `extraPodSpec` for the `VllmDecodeWorker` and `VllmPrefillWorker`.
3. User applies the updated DGD manifest.
4. The new Worker Pods will use the LMCache connector, while the old Worker Pods will continue to use the NIXL connector.

##### Expected Behavior

- In-flight requests will be properly served (through graceful shutdown or request migration)
- Complete uptime of the model inference deployment is maintained

### Roll Out Strategies

#### Deployments

The core Kubernetes `Deployment` resource has [two strategies for rolling out updates:](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy) `RollingUpdate` and `Recreate`.

- `RollingUpdate` is the default strategy where Pods are updated by gradually scaling down the old `ReplicaSet` and scaling up the new `ReplicaSet`.
  - Optional field `maxUnavailable` specifies the maximum number of Pods that can be unavailable during the update.
  - Optional field `maxSurge` specifies the maximum number of Pods that can be created above the desired number of Pods.
- `Recreate` strategy will delete all existing Pods before the new ones are created.

#### StatefulSets

The core [Kubernetes `StatefulSet` resource](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/stateful-set-v1/) has two `updateStrategy` options: `RollingUpdate` and `OnDelete`.

- `RollingUpdate` is the default strategy where Pods are updated in sequence (highest index -> lowest index) one Pod at a time.
  - Optional field `maxUnavailable` specifies the maximum number of Pods that can be unavailable during the update (default is 1).
  - Optional field `partition` indicates the ordinal at which the StatefulSet should be paritioned for updates (default is 0 - all Pods are updated).
- `OnDelete` is the legacy behavior where Pods are only updated when they are manually deleted.

_Note that there is no `maxSurge` for `RollingUpdate` strategy as stable Pod identity (index) is important for exclusive PVCs and deterministic startup/shutdown ordering_

#### LeaderWorkerSet

The `LeaderWorkerSet` resource has a single [`RolloutStrategy` `RollingUpdate`](https://github.com/kubernetes-sigs/lws/blob/main/api/leaderworkerset/v1/leaderworkerset_types.go)

- Optional field `partition` behaves the same as StatefulSet `RollingUpdate` `partition` field.
- Optional field `maxUnavailable` behaves the same as StatefulSet `RollingUpdate` `maxUnavailable` field.
- Optional field `maxSurge` behaves the same as Deployment `RollingUpdate` `maxSurge` field.

_Note that LWS does support `maxSurge`. Example of how it works [here](https://lws.sigs.k8s.io/docs/concepts/rollout-strategy/)_

#### Gateway API Inference Extension

The Gateway API Inference Extension (GAIE) does not control the `Model Server` itself (the piece that is actually running the model inference). Example of a [vllm Model Server Deployment](https://github.com/kubernetes-sigs/gateway-api-inference-extension/blob/main/config/manifests/vllm/gpu-deployment.yaml)

Since GAIE is just a Gateway that is routing between `InferencePool`s (`Model Server` grouping for a single model), they [expect the user to create another `Model Server` Deployment themselves and use an `HTTPRoute` to split traffic between the new and old `InferencePool`](https://gateway-api-inference-extension.sigs.k8s.io/guides/inferencepool-rollout/). This enables canary or blue/green roll out strategies but the way the `Model Server` is rolled out for updates is on the user.

_Note that llm-d uses GAIE and Deployment (single-node) and LeaderWorkerSet (multi-node) for model serving so it inherits both of their roll out strategies._

#### CentML Platform

The CentML Platform leverages [`Argo Rollouts`](https://argo-rollouts.readthedocs.io/en/stable/) to enable an [automated canary roll out strategy](https://github.com/CentML/platform/blob/main/catalog/src/catalog/inference_deployment_v3/templates/rollout.yaml).

Argo Rollouts are designed to be a drop in replacement of `Deployment` where the PodSpec is the same but the `.spec.strategy` has more powerful capabilities.

- An example is a canary style roll out where you can define an [`Analysis`](https://argo-rollouts.readthedocs.io/en/stable/features/analysis/) that can query custom metrics and define success/failure conditions, the step size, traffic weighting and roll out speed

_Future inspiration for how a user can perform a canary roll out that monitors SLOs (TTFT, ITL) of new set of workers during roll out._

#### AIBrix

#### OME

#### Canary Roll Out

#### Blue/Green Roll Out

### Grove Rolling Update Strategy

The Dynamo Kubernetes Operator leverages the [Grove API](https://github.com/ai-dynamo/grove/tree/main) for gang-scheduling and topology awareness. While the Dynamo Operator has support for using traditional Kubernetes Deployments/Services for single-node and LeaderWorkerSet resource for multi-node cases, the Grove API resources are the supported path going forward for single-node and multi-node deployments. Given this is the case, this proposal will only consider the impacts of Groves rolling update strategy on DynamoGraphDeployment (DGD) resources.

#### Single-Node Deployment

[Single-Node Deployment Grove Example](./dynamo-grove-single-node.png)

In Grove, when a rolling update is performed, the `PodCliqueSet` controller will perform the update one `PCS` replica at time. Currently, a DGD only creates a single `PCS` replica. Within the `PCS` replica, there is a `PodClique` (`PC`) for each service in the DGD - in the example above one for the Frontend, one for Decode worker and one for Prefill worker.

For each `PC` concurrently, the `PC` controller will handle one Pod at a time, concurrently deleting an old Pod replica and creating a new Pod replica. The controller will wait for the new Pod replica to be Ready and then proceed to the next replica.

Because the `PC` controller is recreating each Pod index, to ensure up time for a Frotend or set of Model Workers, you need > 1 `PC` replicas for availability.

#### Multi-Node Deployment

[Multi-Node Deployment Grove Example](./dynamo-grove-multi-node.png)

For the Multi-Node case, there will be a `PodCliqueScalingGroup` (`PCSG`) for each set of workers - in the example above one for the Decode workers and one for the Prefill workers. When a `PC` is managed by a `PCSG`, the `PCSG` controller is responsible for performing the rolling update instead of the `PC` controller (different than single-node case).

The `PCSG` controller updates `PCSG` replicas one at a time in ascending ordinal order (0 -> 1 -> 2 -> ...). Within a `PCSG` replica, all `PC`s are deleted and recreated concurrently. Having multiple `PCSG` replicas ensures availability for the set of workers.

**Notes**:

- This is the default rolling update strategy for Grove. There currently is no way to configure the rolling update strategy for a `PCSG` for things like `maxSurge`, `maxUnavailable`. Discussion [here](https://github.com/ai-dynamo/grove/issues/212) on supporting additional rolling update strategies.
- Grove's roll out strategy would be the same as `Deployment` `RollingUpdate` where `maxSurge` is 0 and `maxUnavailable` is 1.

# Issues

## Model Crosstalk

**Diagram here**

When deploying two DGDs for two different models, say **llama3.1-70B** and **Qwen3-0.6B**, if the DGDs are deployed in the same namespace (both have the same `dynamoNamespace` field values), the workers will register under the same `EndpointId`. In the vllm `disagg.yaml` example above, both workers for **llama3.1-70B** and **Qwen3-0.6B** will register under the `EndpointId` `{dynamoNamespace}/backend/generate`. Because the `Client` for both model engines is watching the same prefix `v1/instance/{dynamoNamespace}/backend/generate`, each `Client` will discover instances from both models, resulting in the `PushRouter` routing engine for **llama3.1-70B** for example sending requests to both sets of model workers.

### Shared Frontend

As a current solution, if a user wants to deploy two DGDs for two different models, they can either deploy their DGDs in different namespace or use a shared frontend. An example of a shared frontend is shown [here](https://github.com/ai-dynamo/dynamo/tree/main/examples/basics/kubernetes/shared_frontend). It involves deploying a DGD for the frontend in the global namespace `dynamo` and then deploying two DGDs for the workers in different namespaces.

## Rolling Update Scenarios

### Worker Version Upgrade

_Assumes multiple worker replicas are deployed given Grove's default rolling upgrade strategy._

Has been tested but does not work as desired:

1. User updates the vllm `disagg.yaml` to use a new image for the `VllmPrefillWorker` and `VllmDecodeWorker` services (or single worker service in aggregrated deployment).
2. User patches the DGD using a `kubectl apply -f`
3. The `Prefill` and `Decode` `PodClique`s will terminate an old `Prefill` and `Decode` `Pod` and create a new `Prefill` and `Decode` `Pod` concurrently.
4. The old worker `Pod`s receives a SIGTERM triggering a graceful shutdown by draining in-flight requests then cancelling the primary token which will remove the instance and MDC registration from the KvStore. However, from testing (backend specific), it's found that there is a period where the worker `Pod`s still have a KvStore registration but aren't listening on the `EndpointId` anymore -> results in request errors.
5. The new worker `Pod`s will start up, and once registered in the KvStore -> new requests can now be routed to the new worker `Pod`s.
6. In period during 4. and 5., the extra worker replicas will continue to serve new requests.
7. Grove will wait until new worker `Pod`s are `Ready` before proceeding to the next set of worker `Pod`s.
8. Grove will continue through the rolling update process for the `PodClique` until only new worker `Pod`s are running.

#### Results:

- Some requests during rolling update error (not desired). **This is something that needs to be fixed at the runtime/framework component level.**

### Frontend Version Upgrade

_Assumes multiple frontend replicas are deployed given Grove's default rolling upgrade strategy._

Has been tested and does work as desired:

1. User updates the vllm `disagg.yaml` to use a new image for the `Frontend` service.
2. User patches the DGD using a `kubectl apply -f`
3. The Frontend `PodClique` will terminate an old Frontend `Pod` and create a new Frontend `Pod` concurrently.
4. The old Frontend `Pod` receives a SIGTERM. The old Frontend Pod will no longer be `Ready`, so the Kubernetes `Service` will remove the old Frontend `Pod` from the endpoint list -> new requests will no longer be routed to the old Frontend `Pod`. While in `Terminating` state, the Frontend process will gracefully shutdown by draining in-flight requests before exiting.
5. The new Frontend `Pod` will start up, and once in `Ready` state, the Kubernetes `Service` will add the new Frontend `Pod` to the endpoint list -> new requests can now be routed to the new Frontend `Pod`.
6. In period during 4. and 5., the extra Frontend replicas will continue to serve new requests.
7. Grove will wait until new Frontend `Pod` is `Ready` before proceeding to the next Frontend `Pod`.
8. Grove will continue through the rolling update process for the `PodClique` until only new Frontend Pods are running.

#### Results:

- In-flight requests will be properly served (through graceful shutdown or request migration)
- All requests during rolling update will be served successfully

### Different MDC Checksums

In the scenario where a user would like to update the kv cache block size as an example, currently here is what happens:

1. User patches the DGD CRD using a `kubectl apply -f`
2. The Dynamo Operator will update the Grove `PodCliqueSet` `PodCliqueSetTemplateSpec`.
3. Grove Operator for each `PodClique` with updated template hash will perform a rolling update of the `PodClique`. Per `PodClique`:
4. A Worker `Pod` in the old set will be terminated concurrently with a Worker `Pod` in the new set being started.
5. The old Worker `Pod` receives a SIGTERM. The Worker process itself will gracefully shutdown by draining in-flight requests then cancelling the primary token which will remove the instance and MDC registration from the KvStore. The worker instance will then be removed from the `Client` instance list.
6. The new Worker `Pod` process will start up, in which it creates a KvStore lease and attempts to register the MDC.
7. At this point in time the Frontend will see that for Model A that an engine already exists but that the MDC checksum differs from the MDC of the new Worker Pod. The new worker MDC will be discarded, however the worker instance has the same `EndpointId` so it'll appear on the `Client` instance list.
8. The new Worker instance will start to receive traffic as it's registered on the `Client` instance list.
9. Grove will continue through the rolling update process for the `PodClique` until only new Worker Pods are running.
10. Despite the old Worker Pods being terminated, the `ModelManager` will have the MDC from the old Worker Pods because the `Client` instance list was never empty. This can result in unexpected behavior - for example a new worker can attempt to change the tokenizer config, but since the `ModelManager` engine is using the old MDC, the request preprocessing will still use the old tokenizer config.

#### Results:

- New workers are using old MDC that can result in incorrect pre-processing behavior.

### Incompatible KV Cache Transfer Mechanisms

_Assume initial processed request is first sent to Decode then from Decode to Prefill._

Same as **Different MDC Checksum** from 1-5

6. Now, the KV cache transfer configuration is changed which does not affect the MDC checksum. The new Worker Pod will be added to the `Client` instance list.
7. A decode instance from old Worker set will attempt to communicate with the new Prefill instance.
8. Because the KV cache transfer configuration is incompatible, the decode instance will not be able to communicate with the new Prefill instance, resulting in a communication failure and request error.

#### Results:

- Some requests during rolling update error (not desired)

# Solution

## Model Worker Group Isolation at DGD Level

Building on top of the [Dynamo namespaces based isolation DEP](https://github.com/ai-dynamo/enhancements/pull/28) from @biswapanda, here is what has been implemented so far to date:

- Frontend namespace filtering as outlined in **Model Deployment Card** section above.
- Backend components receive `DYN_NAMESPACE` env var from Operator that is specified as the `dynamoNamespace` in the DGD service spec. However, because the `dynamoNamespace` field can be the same across multiple DGDs, you can still encounter the **Model Crosstalk** issue.

What has been left to be implemented is that the namespace be directly the DGD name instead of the `dynamoNamespace` field value required for each service.

So currently:

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg
spec:
  services:
    Frontend:
      dynamoNamespace: vllm-llama3-70b
      componentType: frontend
      replicas: 1
    VllmDecodeWorker:
      dynamoNamespace: vllm-llama3-70b
      componentType: worker
      subComponentType: decode
      replicas: 1
```

In this example, the `VllmDecodeWorker` `EndpointId` would be `vllm-llama3-70b/backend/{endpoint}`, where `{endpoint}` would be dictated by the endpoint created by the component runtime itself. For example the `VllmDecodeWorker` `generate` endpoint would be `vllm-llama3-70b/backend/generate` and the `VllmDecodeWorker` `clear_kv_blocks` endpoint would be `vllm-llama3-70b/backend/clear_kv_blocks`.

Using the DGD name as the namespace, we would have:

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
    VllmDecodeWorker:
      componentType: worker
      subComponentType: decode
      replicas: 1
```

In this case, the `VllmDecodeWorker` `EndpointId` would be `vllm-agg/backend/{endpoint}`. This eliminates the **Model Crosstalk** issue as each DGD now has a unique namespace and is one less required field for users to configure in DGD manifest.

### Appending Worker Group Hash to Namespace

Using the DGD name as the namespace prevents the **Model Crosstalk** issue. However, it does not solve the **Incompatible KV Cache Transfer Mechanisms** issue where you have a set A and set B of model workers that cannot communicate with each other.

To solve this, we can introduce a hash appended to the namespace to indicate a set A and set B of model workers. For instance, workers for set A would have a namespace like `vllm-llama3-70b-a1b2c3d4` and workers for set B would have a namespace like `vllm-llama3-70b-e5f6g7h8`. This creates a clear isolation for the two sets of workers anad prevents them from communicating with each other.

**Issue**: How do we calculate the hash to append to the namespace for a set of workers?

- Hashing at the DGD spec level, every DGD update would require updating the namespace of the set of workers -> a roll out for all workers (not desirable).
- Hashing at the component level, each DGD component would receive a different hash -> hash needs to be the same for sets of components.

#### Hash Solution 1: Introduce Worker Group to DGD

Solution 1 introduces a new field to the DGD spec called `workerGroupName`. This enables a DGD author to group workers into different sets (namespace isolation). For example, if a user wishes to perform a canary roll out strategy for a disaggegrated deployment where they want to have 9 replicas for set A of prefill/decode workers and 1 replica for set B of prefill/decode workers, they could do so like this:

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1

    VllmDecodeWorker-A:
      workerGroupName: A # ← Groups A decode + prefill
      componentType: worker
      subComponentType: decode
      replicas: 9
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0
          args:
            - --model
            - Qwen/Qwen3-0.6B

    VllmPrefillWorker-A:
      workerGroupName: A # ← Same group = same hash
      componentType: worker
      subComponentType: prefill
      replicas: 9
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --is-prefill-worker

    # Worker Group B (1 replicas)
    VllmDecodeWorker-B:
      workerGroupName: B # ← Groups B decode + prefill
      componentType: worker
      subComponentType: decode
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.7.0
          args:
            - --model
            - Qwen/Qwen3-0.6B

    VllmPrefillWorker-B:
      workerGroupName: B # ← Same group = same hash
      componentType: worker
      subComponentType: prefill
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.7.0
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --is-prefill-worker
```

- The Operator groups services with the same `workerGroupName` into a set and calculates a hash to append to the namespace. If no `workerGroupName` is defined, it's assumed to be a part of the `default` worker group.
- The Frontend would load balance traffic between the two sets of workers based on the number of instances in each set.

##### Pros:

- Easy to visualize the current state of the deployment like in the case of a canary roll out.
- Supports the [Hierachical Planner proposal](https://github.com/ai-dynamo/enhancements/pull/46/files?short_path=09e8f0f#diff-09e8f0f98b689bb0d0f4d26f6a32f85b27c554cb38341f40faab0ef1232442f4) where multiple models can be deployed in the same DGD

##### Cons:

- Manifest becomes verbose when having multiple worker groups in the disaggregated deployment case.

#### Hash Solution 2: Introduce Controller Revision to DGD

Solution 2 leverages the [Kubernetes `ControllerRevision` resource](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/controller-revision-v1/) to store updates to the DGD spec. This is how `LeaderWorkerSet` can support canary roll out strategies where a [`partition` field](https://github.com/kubernetes-sigs/lws/blob/f01674b29576223329e35432c77d2b37a995b669/api/leaderworkerset/v1/leaderworkerset_types.go#L272) is used to determine the number of LWS replicas to update with updated template spec. Similar to the canary roll out scenario outlined in Solution 1, the user would have an initial DGD deployed like so:

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1

    VllmDecodeWorker:
      componentType: worker
      subComponentType: decode
      replicas: 10
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0
          args:
            - --model
            - Qwen/Qwen3-0.6B

    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 10
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --is-prefill-worker
```

The user can then perform a canary roll out by updating their `VllmDecodeWorker` and `VllmPrefillWorker` spec and specifying a `partition` field to determine the number of replicas to update.

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1

    VllmDecodeWorker:
      componentType: worker
      subComponentType: decode
      replicas: 10
      partition: 9 # only replicas with index >= 9 (1 replica) is updated to image 0.7.0
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.7.0 # update image to 0.7.0
          args:
            - --model
            - Qwen/Qwen3-0.6B

    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 10
      partition: 9 # only replicas with index >= 9 (1 replica) is updated to image 0.7.0
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.7.0 # update image to 0.7.0
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --is-prefill-worker
```

Here, the partition indicates that only the 10th replica (0-indexed) is updated to image 0.7.0. The Dynamo Operator will create a `ControllerRevision` resource for every DGD update. So the initial revision `rev1` will be stored and used as the appended hash for the namespace `vllm-disagg-rev1`. The new revision `rev2` will be stored and used as the appended hash for the namespace `vllm-disagg-rev2`.

##### Pros:

- Similar UX as LeaderWorkerSet roll out configuration
- Manifest is more concise compared to Solution 1
- ControllerRevision provides a built in mechanism to rollback to a previous revision.

##### Cons:

- Relies on the `ControllerRevision` resource to store updates to the DGD spec.
- Does not enable deploying multiple models within the same DGD - required for Hierarchical Planner.

#### Suggested Hash Solution

- Solution 1 as it can enable deploying multiple models within the same DGD (Hierachical Planner) and the explicit `workerGroupName` makes the worker grouping clear

### DGD/Operator Updates

- Remove `dynamoNamespace` field from DGD service spec and inject DGD name as `DYN_NAMESPACE` env var.
- If Solution 1 is chosen, add a new field to the DGD service spec called `workerGroupName` with logic in operator to generate hashes per worker group and append to `DYN_NAMESPACE` env var.
- If Solution 2 is chosen, add a new field to the DGD service spec called `partition` with logic in operator to create a `ControllerRevision` resource for every DGD update and inject `DYN_NAMESPACE` env var with the correct revision hash.

### Core Dynamo Lib Updates

- `ModelWatcher.watch` can now filter based on namespace prefix match instead of exact match (e.g. frontend matching for `vllm-agg-*` where `vllm-agg-a1b2c3d4` and `vllm-agg-e5f6g7h8` are the two worker groups).
- `ModelManager` updated to support multiple `Client`s per model engine. Allow `PushRouter` to load balance traffic between multiple sets of workers. Round-robin and random case seem simple (each Client will have an instance count) but KV-Router case is more complex. Do we do weighted selection of worker group and then worker selection based on kv instance radix tree?

### Solves Issues

- **Model Crosstalk** - since namespace is now the DGD name and DGD name is unique, model crosstalk will not occur.
- **Different MDC Checksum** - in a RollingUpdate scenario or with a canary roll out using worker groups, set A and set B will have hashes appended to isolate them. This assumes the Core Dynamo Lib can be updated to support multiple `Client`s per model engine and load balance traffic across.
- **Incompatible KV Cach Transfer** - similar to **Different MDC Checksum**, set A and set B will have different namespaces, so communication between set A and set B will not occur.

### Adds Functionality

- If using `workerGroupName` solution, a DGD can support the Hierarchical Planner proposal where multiple configurations of the same model can be deployed in the same DGD (do note that the SLO routing logic is not specified - would need to be solved).
- Adds support for a canary roll out strategy where a user can deploy X% of new model workers within a DGD and monitor SLOs for the new workers before gradually scaling down old set/scaling up new set.
- Does not **prohibit**:
  - The `Shared Frontend` pattern
  - A `Blue/Green` roll out strategy can still be performed at a layer above the DGD using traditional k8s tooling (Ingress traffic splitting, etc.)

## Additions to Grove's Default Rolling Update Strategy

To give the user more fine-grained control over Grove's default rolling update strategy and to support similar semantics to `Deployment`, `StatefulSet` and `LeaderWorkerSet`, we propose adding:

- `maxUnavailable` field for `PC` and `PCSG` to specify the maximum number of replicas that can be unavailable during the update. This controls the speed of the roll out (you can delete/recreate `maxUnavailable` replicas at a time). The default would be 1 (current behavior). Note that there is already a `minAvailable` field for `PC` and `PCSG` which is used for gang scheduling/termination semantics - 1) min number of replicas that are guaranteed to be scheduled together and 2) if the number of available replicas is less then `minAvailable`, then after `terminationDelay` period, the gang will be terminated.
- `maxSurge` field for `PC` and `PCSG` to specify the maximum number of replicas that can be created above the desired number of replicas. This can enable more availability during the roll out process (can have replica 1 but have maxSurge of 1, maxUnavailable of 0 - ensures that the new replica is ready before the old replica is terminated).
  - However, supporting `maxSurge` becomes a lot more tricky because the surge replica indices would be > desired replicas. It also creates issues with the gang scheduling/termination semantics. Does a surge replica count towards `minAvailable`? This also can consume a lot of resources if done at the PCSG level where you are scheduling multiple PCs at a time.

Support for both `maxUnavailable` and `maxSurge` could look like the following for a `DGD`:

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-disagg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1 # with a single frontend replica, maxSurge 1 and maxUnavailable 0 ensures that the new frontend replica is ready before the old frontend replica is terminated
      rollingUpdate:
        maxSurge: 1
        maxUnavailable: 0
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
    VllmDecodeWorker:
      componentType: worker
      subComponentType: decode
      replicas: 4 # will delete/recreate 2 replicas at a time
      rollingUpdate:
        maxUnavailable: 2
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          workingDir: /workspace/components/backends/vllm
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
    VllmPrefillWorker:
      componentType: worker
      subComponentType: prefill
      replicas: 2 # follows default of maxSurge 0 and maxUnavailable 1 (1 replica deleted/recreated at a time)
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          workingDir: /workspace/components/backends/vllm
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
            - --is-prefill-worker
```

_Note that the `maxUnavailable` and `maxSurge` field is supported at the `DGD` service spec level since the Frontend will be a standalone `PC` and the Prefill/Decode workers will be a `PC` for single-node case or multiple `PC`s managed by a `PCSG` in the multi-node case._

# Guidance on a Single Model vs. Multiple Models per DGD

The proposed solution would allow for both. You can have **llama3.1-70B** and **Qwen3-0.6B** deployed within the same DGD like so:

```yaml
apiVersion: nvidia.com/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: vllm-agg
spec:
  services:
    Frontend:
      componentType: frontend
      replicas: 1
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
    QwenWorker:
      componentType: worker
      workerGroupName: qwen
      replicas: 1
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          workingDir: /workspace/components/backends/vllm
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - Qwen/Qwen3-0.6B
    LlamaWorker:
      componentType: worker
      workerGroupName: llama
      replicas: 1
      resources:
        limits:
          gpu: "1"
      extraPodSpec:
        mainContainer:
          image: nvcr.io/nvidia/ai-dynamo/vllm-runtime:my-tag
          workingDir: /workspace/components/backends/vllm
          command:
            - python3
            - -m
            - dynamo.vllm
          args:
            - --model
            - llama/Llama3.1-70B
```

This would work. You can also have a separate DGD for each model. This also applies to having the same model with different configurations deployed in the same DGD (new roll out or hierarchical planner).

### Single Model per DGD

**Pros**:

- Clearer separation of concerns. DGD == a singular model.
- Lifecycle of the single Model being served is tied to the lifecyle of the DGD.
- Higher level Gateway/Ingress routing is simple - singular DGD Service is tied to a single model. No need for dynamic discovery of what models are being served.

**Cons**:

- Deploying a Frontend per Model. (This can be cirumvented with the shared frontend pattern)

### Multiple Models per DGD

**Pros**:

- More flexibility for advanced use cases - speculative decoding with a draft model, hierarchical planner, etc.

**Cons**:

- Lifecycle of multiple models is tied to the lifecycle of the DGD
- Gang scheduling/termination semantics are at the level of a DGD
- More complex Gateway/Ingress routing - a DGD Service can be serving multiple models.

# Alternatives Considered

## DGD Updates are Not Supported

In this case, the DGD operator would not support updating the DGD spec once deployed. This means that in order for the user to perform any roll out, they would need to deploy another DGD and leverage a higher level Gateway/Ingress routing to load balance traffic between the two DGDs during the roll out process.

**Pros**:

- Keeps the Dynamo Core Lib logic simple and focused on serving a single Model/MDC
- Work to make DGD immutable is simple. No other work would be needed

**Cons**:

- Requires the user to develop/find their own solution for performing roll outs
- Not consistent with roll out strategies supported by alternative solutions
- Two DGDs == two units of gang scheduling/termination and topology awareness

## Only Enable Version Updates for DGD Spec

In this case, we would limit the subset of things a user can update on the DGD spec to only be the image tag of the service. This would enable a `RollingUpdate` strategy in the scenario of **Worker Version Update** and **Frontend Version Update**.

**Pros**:

- Enables in-place roll out strategy for **Worker Version Update** and **Frontend Version Update**
- **Frontend Version Update** is currently supported, **Worker Version Update** would require work to ensure requests during roll out do not error.
- Still keeps the Dynamo Core Lib logic simple and focused on serving a single Model/MDC

**Cons**:

- In the case of **Incompatible KV Cache Transfer Mechanisms** or **Different MDC Checksums**, the user would need to perform a roll out at a higher level than Dynamo DGD (blue/green with Ingress traffic splitting)
- Two DGDs == two units of gang scheduling/termination and topology awareness

## Enable Multiple MDCs per Model (No Support for Worker Group Isolation)

In this case, a user can make any update to the DGD spec in order to perform a roll out. The Dynamo Core Lib would be updated to support multiple MDCs per model engine. This would solve for **Worker Version Update**, **Frontend Version Update** and **Different MDC Checksums**.

However, this would not solve for **Incompatible KV Cache Transfer Mechanisms** as their is no worker group isolation (all workers new/old in the same namespace). Also, it is not clear to the user what would make KV cache transfer incompatible.

**Pros**:

- Enables in-place roll out strategy for **Worker Version Update**, **Frontend Version Update** and **Different MDC Checksums**

**Cons**:

- Makes the Dynamo Core Lib more complex as it needs to support multiple MDCs per model engine.
- Does not solve for **Incompatible KV Cache Transfer Mechanisms** case

**TODOS**:

- if the user updates the served model name, this results in a different model key. Should we support this? What would happen in the current solution?
- LORA support?

- Issues should be clear and clear how proposed solution(s) addresses them
- ## Follow DEP outline structure:
