# Background

This enhancement proposal is meant to address two things:

1. Proposal on how the DynamoGraphDeployment (DGD) CRD specification creates addressable components. (TODO: fix wording)
2. Proposing solutions for outlined rolling update scenarios.

## Model Deployment Cards (MDC)

In Dynamo, each model is represented by a **ModelDeploymentCard (MDC)**. The MDC comprises of the model name (display name), tokenizer config, context length, model input/type specification, among other fields that defines the model inference behavior.

When a model worker starts up, it publishes its MDC to the KvStore with the key `v1/mdc/{namespace}.{component}.{endpoint}/{instance_id}`.

The Frontend has a `ModelWatcher` that watches the KvStore PUT events for keys with the prefix `v1/mdc`.

**TODO: link to the code where this is done**
The `ModelWatcher.watch` method first determines if the Frontend should process the MDC or discard it. If the frontend is deployed in the global namespace (`dynamo`), then all MDCs observed will be processed. If the Frontend is deployed in a non-global namespace, than only MDCs with keys that match the namespace will be processed (`v1/mdc/{namespace}.*`)

**TODO: link to the code where this is done**
If the MDC should be processed, `ModelWatcher` will grab the model name (display name) from the MDC and check if the `ModelManager` already contains an engine for that model. If there is an engine that exists but the MDC checksum (generated from select MDC fields) does not match, the `ModelWatcher` will discard the instance and log an error. Currently, the worker with discarded MDC will continue to run and consume resources without ever receiving traffic. If there is no engine for that model, an engine is created and stored on the `ModelManager`.

The model engine contains a `Client` constructed for the MDC `Endpoint`. This `Client` will watch the KvStore PUT events for keys with the prefix `v1/instance/{namespace}/{component}/{endpoint}`. When a PUT event is received, the `Client` will update the instance list.

### DynamoGraphDeployment (DGD) interaction with MDC

A **DynamoGraphDeployment (DGD)**, is the Dynamo Kubernetes CRD for deploying a distributed inference graph using the Dynamo runtime.

**TODO: link to the yaml**
The CRD spec contains a list of services which define the components that make up an inference graph. For instance, in the vllm `disagg.yaml` file, the services are: `Frontend`, `VllmDecodeWorker`, and `VllmPrefillWorker`. Each component has a field `dynamoNamespace` which defines the `namespace` part of the `Endpoint` that the worker will listen on. The `component` and `endpoint` parts of the worker `Endpoint` are not derived from the DGD but rather statically defined within the backend component itself. For instance, the `VllmDecodeWorker` component `Endpoint` has `component` part `backend` and `endpoint` part `generate`. `VllmPrefillWorker` component `Endpoint` has `component` part `prefill` and `endpoint` part `generate`.
**TODO: point to the Model crosstalk section to explain why the static `component` and `endpoint` cause issues**

## Common Rolling Update Strategies

## Rolling Update Scenarios

### Update Dynamo Worker Version

### Update Frontend Worker Version

### Different MDC Checksums

### Incompatible KV Cache Transfer Mechanisms

# Issues

## Model Crosstalk

When deploying two DGDs for two different models, say **llama3.1-70B** and **Qwen3-0.6B**, if the DGDs are deployed in the same namespace (both have the same `dynamoNamespace` field values), the workers will register under the same `Endpoint`. In the vllm `disagg.yaml` example above, both workers for **llama3.1-70B** and **Qwen3-0.6B** will register under the `Endpoint` `{dynamoNamespace}/backend/generate`. Because the `Client` for both model engines is watching the same prefix `v1/instance/{dynamoNamespace}/backend/generate`, each `Client` will discover instances from both models, resulting in the `PushRouter` routing engine for **llama3.1-70B** for example sending requests to both sets of model workers.

TODO: temp solution - shared frontend

## Rolling Update Scenarios

### Different MDC Checksums

In the scenario where a user would like to update the model's context length as an example, currently here is what happens:

1. User patches the DGD CRD using a `kubectl apply -f`
2. The Dynamo Operator will update Grove `PodCliqueSet` `PodClique`(s) `PodTemplate` (TODO: confirm this is correct wording)
3. Grove Operator for each `PodClique` whose template hash has been changed will perform a rolling update of the `PodClique`
4. A Worker `Pod` in the old set will be terminated concurrently with a Worker `Pod` in the new set being started.
5. The old Worker `Pod` receives a SIGTERM. The Worker process itself will gracefully shutdown by draining in-flight requests then cancelling the lease which will remove the instance and MDC registration from the KvStore and will stop it from listening on the `Endpoint` address (TODO: confirm correct wording here)
6. The new Worker `Pod` process will start up, in which it creates a KvStore lease and attempts to register the MDC.
7. At this point in time the Frontend will see that for Model A that an engine already exists but that the MDC checksum differs from the MDC of the new Worker Pod.
8. Grove will continue through the rolling update process for the `PodClique` until only new Worker Pods are running.
9. However, the new Worker Pods will never attempt to re-register the MDC and so the Frontend drops the Model A. All subsequent requests will fail with Model Not Found despite the new Worker Pods being able to serve the model.
   **TODO: confirm what is said in 9 is correct by testing this scenario.**

### Incompatible KV Cache Transfer Mechanisms

Same as **Different MDC Checksum** from 1-4
**TODO: fill in for the rest of the steps in how a prefill/decode worker of the new set would attempt to communicate with the olde set, resulting in communication failures and request errors.**

**TODO: confirm what is said is correct by testing this scenario.**

# Solution

## Updates to Addressing Scheme

Building on top of the [Dynamo namespaces based isolation DEP](https://github.com/ai-dynamo/enhancements/pull/28) from @biswapanda, here is what has been implemented so far to date:

- Frontend namespace filtering as outlined in **Model Deployment Card** section above.
- Backend components receive `DYN_NAMESPACE` env var from Operator that is specified as the `dynamoNamespace` in the DGD service spec. However, note that the `component` and `endpoint` parts of the `Endpoint` address are still static which can result in the **Model Crosstalk** issue.

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

Where the `VllmDecodeWorker` `Endpoint` would be `vllm-llama3-70b/backend/{endpoint}`, where `{endpoint}` would be dictated by the endpoint created by the component runtime itself. For example the `VllmDecodeWorker` `generate` endpoint would be `vllm-llama3-70b/backend/generate` and the `VllmDecodeWorker` `clear_kv_blocks` endpoint would be `vllm-llama3-70b/backend/clear_kv_blocks`.

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

Where the `VllmDecodeWorker` `Endpoint` would be `vllm-llama3-70b-abcdef/worker-decode/{endpoint}`, where `{endpoint}` would be dictated by the endpoint created by the component runtime itself. Note that the endpoint naming schema is now `{dgdName}-{hashOfComponentDeploymentSpec}/{componentType}-{subComponentType}/{endpoint}`. `subComponentType` is optional so if not provided, then the endpoint would be `{dgdName}-{hashOfComponentDeploymentSpec}/{componentType}/{endpoint}`.

**Pros**:

- This eliminates the **Model Crosstalk** issue as each DGD now has a unique namespace.
- The hash of the `DynamoComponentDeploymentSpec` gives us a nice property where in the case of for the update of a `DynamoComponentDeploymentSpec` that the old and new set of workers are in separate namespaces. **TODO reference section where this is discusses**

**Cons**:

### DGD/Operator Updates

- Remove `dynamoNamespace` field from DGD service spec
- Have `componentType` become a validated enum of `frontend`, `planner`, or `worker`

### Core Rust Updates

TODO: discuss

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

### Understanding of componentType and subComponentType

- **componentType**: can be `frontend`, `planner`, `worker` or `*` (anything else or not provided). Should be enforced as enum at kube API level
- **subComponentType**: can be `*` (any value or not provided). Has no affect on operator but used by Planner to determine if what services are prefill and decode.
