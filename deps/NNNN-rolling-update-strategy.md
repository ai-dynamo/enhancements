# Background

This enhancement proposal is meant to address two things:

1. Building on @biswapanda [Dynamo namespaces based isolation DEP](https://github.com/ai-dynamo/enhancements/pull/28) to propose DynamoGraphDeployment (DGD) CRD strict coupling of model <> (frontend/set of workers) and how the DGD specification creates addressable services.
2. Proposing solutions for outlined roll out scenarios given 1.

## Model Deployment Cards (MDC)

In Dynamo, each model is represented by a **ModelDeploymentCard (MDC)**. The MDC comprises of the model name (display name), tokenizer config, context length, model input/type specification, among other fields that defines the model inference behavior.

When a model worker starts up, it publishes its MDC to the KvStore with the key `v1/mdc/{namespace}.{component}.{endpoint}/{instance_id}`.

The Frontend has a `ModelWatcher` that watches the KvStore PUT events for keys with the prefix `v1/mdc`.

The `ModelWatcher.watch` method first determines if the [Frontend should process the MDC or discard it](https://github.com/ai-dynamo/dynamo/blob/6deeecb1d6a9f4eb1770b4272bfa85a4b6226e0a/lib/llm/src/discovery/watcher.rs#L138). If the frontend is deployed in the global namespace (`dynamo`), then all MDCs observed will be processed. If the Frontend is deployed in a non-global namespace, than only MDCs with keys that match the namespace will be processed (`v1/mdc/{namespace}.*`)

If the MDC should be processed, `ModelWatcher` will extrace the model name (display name) from the MDC and check if the `ModelManager` already contains an engine for that model. If there is an engine that exists but the MDC checksum (generated from select MDC fields) does not match, [the `ModelWatcher` will discard the instance and log an error](https://github.com/ai-dynamo/dynamo/blob/6deeecb1d6a9f4eb1770b4272bfa85a4b6226e0a/lib/llm/src/discovery/watcher.rs#L156). Currently, the worker with discarded MDC will continue to run and consume resources without ever receiving traffic. If there is no engine for that model, an engine is created and stored on the `ModelManager`.

The model engine contains a `Client` constructed for the MDC `EndpointId`. The MDC `EndpointId` is extracted from the MDC key (`{namespace}.{component}.{endpoint}`), which is the address of the model worker. This `Client` will watch the KvStore PUT events for keys with the prefix `v1/instance/{namespace}/{component}/{endpoint}`. When a PUT event is received, the `Client` will update the instance list.

### DynamoGraphDeployment (DGD) interaction with MDC

A **DynamoGraphDeployment (DGD)**, is the Dynamo Kubernetes CRD for deploying a distributed inference graph using the Dynamo distributed runtime and pre-built components.

The CRD spec contains a list of services which define the components that make up an inference graph. For instance, in the [vllm `disagg.yaml` file](https://github.com/ai-dynamo/dynamo/blob/main/components/backends/vllm/deploy/disagg.yaml), the services are: `Frontend`, `VllmDecodeWorker`, and `VllmPrefillWorker`. Each component has a field `dynamoNamespace` which defines the `namespace` part of the `EndpointId` that the worker will listen at. The `component` and `endpoint` parts of the worker `EndpointId` are not derived from the DGD but rather statically defined within the backend component itself. For instance, the `VllmDecodeWorker` component `EndpointId` has `component` part `backend` and `endpoint` part `generate`. `VllmPrefillWorker` component `EndpointId` has `component` part `prefill` and `endpoint` part `generate`.
**TODO: point to the Model crosstalk section to explain why the static `component` and `endpoint` cause issues**

## Common Roll Out Scenarios

A roll out scenario is when a user wants to update their model inference deployment. This could be for a variety of reasons such as updating the runtime version, updating the model version or configuration, etc. For the purposes of this proposal, we will only focus on how the impact of these roll out scenarios are affected by updates to a DGD in a Kubernetes environment.

### Update Dynamo Worker Version

#### Use Cases

- Bug/vulnerability fix in newer version of Dynamo worker runtime
- New feature in the Dynamo worker runtime that the user wants to leverage
- Performance improvement in newer version of Dynamo worker runtime

#### Example

An example of a how a user would update the Dynamo worker runtime version from `0.6.0` to `0.7.0`.

1. User applies the initial vllm `disagg.yaml` DGD manifest with the image for each service set to `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.6.0`.
2. User updates the DGD manifest to use the image `nvcr.io/nvidia/ai-dynamo/vllm-runtime:0.7.0` for `VllmDecodeWorker` and `VllmPrefillWorker`.
3. User applies the updated DGD manifest.

#### Expected Behavior

- In-flight requests will be properly served (through graceful shutdown or request migration)
- Complete uptime of the model inference deployment is maintained

### Update Frontend Worker Version

Same use cases, example and expected behavior as **Update Dynamo Worker Version**. Instead of updating the image for the workers, the user would update the image for the `Frontend` service.

### Different MDC Checksums during Roll Out

As explained in the **Model Deployment Cards (MDC)** section above, the MDC is published from the model worker on startup and specifies the model weights being used, the context length, the tokenizer config, prompt formatter, etc. For any of the following use cases listed below, a user would want to update their model deployment via their DGD manifest.

#### Use Cases

- Update model revision
- Update model weight configuration such as quantization
- Update model context length
- Update tokenizer config
- Update chat template

#### Example

An example of a how a user would update the vllm `disagg.yaml` DGD manifest to deploy an updated model revision of Qwen3-0.6B.

1. User applies the initial vllm `disagg.yaml` DGD manifest.
2. User updates the DGD manifest by specifying env var `MODEL_REVISION` as `ABCDDE...` for the `VllmDecodeWorker` and `VllmPrefillWorker`.
3. User applies the updated DGD manifest.
4. The new Worker Pods will fetch the HF model revision `ABCDDE...`, which will result in a different MDC checksum than the old Workers.

#### Expected Behavior

- In-flight requests will be properly served (through graceful shutdown or request migration)
- Complete uptime of the model inference deployment is maintained

### Incompatible KV Cache Transfer Mechanisms during Roll Out

Another scenario a user might encounter is with a disaggregated deployment where the user wants to update how their KV cache transfer mechanism is configured. For instance, the user might want to switch from UCX to NIXL. It's important to note that KV cache transfer configuration is not tracked as a part of the MDC checksum.

#### Example

An example of a how a user would update the vllm `disagg.yaml` DGD manifest to switch from the default NIXL connector to the LMCache connector for the `VllmDecodeWorker` and `VllmPrefillWorker`.

1. User applies the initial vllm `disagg.yaml` DGD manifest.
2. User updates the DGD manifest by specifying the flag `--connector lmcache` in the `args` array in the `extraPodSpec` for the `VllmDecodeWorker` and `VllmPrefillWorker`.
3. User applies the updated DGD manifest.
4. The new Worker Pods will use the LMCache connector, while the old Worker Pods will continue to use the NIXL connector.

#### Expected Behavior

- In-flight requests will be properly served (through graceful shutdown or request migration)
- Complete uptime of the model inference deployment is maintained

# Issues

## Model Crosstalk

When deploying two DGDs for two different models, say **llama3.1-70B** and **Qwen3-0.6B**, if the DGDs are deployed in the same namespace (both have the same `dynamoNamespace` field values), the workers will register under the same `EndpointId`. In the vllm `disagg.yaml` example above, both workers for **llama3.1-70B** and **Qwen3-0.6B** will register under the `EndpointId` `{dynamoNamespace}/backend/generate`. Because the `Client` for both model engines is watching the same prefix `v1/instance/{dynamoNamespace}/backend/generate`, each `Client` will discover instances from both models, resulting in the `PushRouter` routing engine for **llama3.1-70B** for example sending requests to both sets of model workers.

### Shared Frontend

As a current solution, if a user wants to deploy two DGDs for two different models, they can either deploy their DGDs in different namespace or use a shared frontend. An example of a shared frontend is shown [here](https://github.com/ai-dynamo/dynamo/tree/main/examples/basics/kubernetes/shared_frontend). It involves deploying a DGD for the frontend in the global namespace `dynamo` and then deploying two DGDs for the workers in different namespaces.

## Rolling Update Scenarios

### Different MDC Checksums

In the scenario where a user would like to update the model revision as an example, currently here is what happens:

1. User patches the DGD CRD using a `kubectl apply -f`
2. The Dynamo Operator will update the Grove `PodCliqueSet` `PodCliqueSetTemplateSpec`. View Grove example of single-node disaggregated deployment [here](https://github.com/ai-dynamo/grove/blob/main/docs/assets/singlenode-disaggregated.excalidraw.png).
3. Grove Operator for each `PodClique` with updated template hash has been changed will perform a rolling update of the `PodClique`. Per `PodClique`:
4. A Worker `Pod` in the old set will be terminated concurrently with a Worker `Pod` in the new set being started.
5. The old Worker `Pod` receives a SIGTERM. The Worker process itself will gracefully shutdown by draining in-flight requests then cancelling the primary token which will remove the instance and MDC registration from the KvStore. The worker instance will then be removed from the `Client` instance list.
6. The new Worker `Pod` process will start up, in which it creates a KvStore lease and attempts to register the MDC.
7. At this point in time the Frontend will see that for Model A that an engine already exists but that the MDC checksum differs from the MDC of the new Worker Pod. The new worker MDC will be discarded, however the worker instance has the same `EndpointId` so it'll appear on the `Client` instance list.
8. The new Worker instance will start to receive traffic as it's registered on the `Client` instance list.
9. Grove will continue through the rolling update process for the `PodClique` until only new Worker Pods are running.
10. Despite the old Worker Pods being terminated, the `ModelManager` will have the MDC from the old Worker Pods because the `Client` instance list was never empty. This can result in unexpected behavior - for example a new worker can attempt to change the tokenizer config, but since the `ModelManager` engine is using the old MDC, the request preprocessing will still use the old tokenizer config.

### Incompatible KV Cache Transfer Mechanisms

Same as **Different MDC Checksum** from 1-4
**TODO: fill in for the rest of the steps in how a prefill/decode worker of the new set would attempt to communicate with the olde set, resulting in communication failures and request errors.**

**TODO: confirm what is said is correct by testing this scenario.**

# Solution

## Updates to Addressing Scheme

Building on top of the [Dynamo namespaces based isolation DEP](https://github.com/ai-dynamo/enhancements/pull/28) from @biswapanda, here is what has been implemented so far to date:

- Frontend namespace filtering as outlined in **Model Deployment Card** section above.
- Backend components receive `DYN_NAMESPACE` env var from Operator that is specified as the `dynamoNamespace` in the DGD service spec. However, note that the `component` and `endpoint` parts of the `EndpointId` address are still static which can result in the **Model Crosstalk** issue.

What has been left to be implemented is that the namespace be directly the DGD name instead of the `dynamoNamespace` field value required for each service.

So instead of:

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

Where the `VllmDecodeWorker` `EndpointId` would be `vllm-llama3-70b/backend/{endpoint}`, where `{endpoint}` would be dictated by the endpoint created by the component runtime itself. For example the `VllmDecodeWorker` `generate` endpoint would be `vllm-llama3-70b/backend/generate` and the `VllmDecodeWorker` `clear_kv_blocks` endpoint would be `vllm-llama3-70b/backend/clear_kv_blocks`.

We would have:

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

**TODO: hash of component deployment spec would be unique to the component so the namespace would differ**
Where the `VllmDecodeWorker` `EndpointId` would be `vllm-llama3-70b-abcdef/worker-decode/{endpoint}`, where `{endpoint}` would be dictated by the endpoint created by the component runtime itself. Note that the `EndpointId` naming schema is now `{dgdName}-{hashOfComponentDeploymentSpec}/{componentType}-{subComponentType}/{endpoint}`. `subComponentType` is optional so if not provided, then the endpoint would be `{dgdName}-{hashOfComponentDeploymentSpec}/{componentType}/{endpoint}`.

**Pros**:

- This eliminates the **Model Crosstalk** issue as each DGD now has a unique namespace.
- The hash of the `DynamoComponentDeploymentSpec` gives us a nice property where in the case of for the update of a `DynamoComponentDeploymentSpec` that the old and new set of workers are in separate namespaces. **TODO reference section where this is discusses**

**Cons**:

### DGD/Operator Updates

- Remove `dynamoNamespace` field from DGD service spec
- Have `componentType` become a validated enum of `frontend`, `planner`, or `worker`

### Core Dynamo Lib Updates

- `ModelWatcher.watch` can now filter

### Component Updates

- component name derived from componentType
- can have canary/rolling upgrade support using hash of component - append to namespace for supporting two worker pools for a model. solves differing MDCs and kv cache transfer compatibility

- remove shared frontend concept - DGD now becomes a tightly couple frontend to workers for a single model.
- maybe include what would it take for a frontend to be shared across models. downsides of not supporting this?

- how does Grove play into things?

- if the user updates the served model name, this results in a different model key. Should we support this? What would happen in the current solution?
- alternatives considered?

- using a hash of the `DynamoComponentDeploymentSpec` would essentially mean for any update whether it's the image, env vars, etc. - that we are essentially having an isolated set of worker pools. Are there any scenarios in which this would not be desired - i.e. we'd want the new/old set of workers to be in the same pool

- will need to update components where they are creating a client to a hardcoded component/endpoint - example sglang prefill_router_client

- still supporting shared frontend?

- how do we support LORAs?

- hierarchical planner

- could have a top level field for worker componentType `modelName` that becomes the served model name by way of injecting env var. Can then support multiple models within a single DGD.

- spec dec requiring target and draft model

- node maintenance - is this in scope?

### Understanding of componentType and subComponentType

- **componentType**: can be `frontend`, `planner`, `worker` or `*` (anything else or not provided). Should be enforced as enum at kube API level
- **subComponentType**: can be `*` (any value or not provided). Has no affect on operator but used by Planner to determine if what services are prefill and decode.
