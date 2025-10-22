# Background

In Dynamo, each model is represented by a **ModelDeploymentCard (MDC)**. The MDC comprises of the model name (display name), tokenizer config, context length, model input/type specification, among other fields that defines the model inference behavior.

When a model worker starts up, it publishes its MDC to the KvStore with the key `v1/mdc/{namespace}.{component}.{endpoint}/{instance_id}`.

When deploying a **DynamoGraphDeployment (DGD)**, the CRD spec has a list of services which define the components that make up an inference graph. For instance, in the vllm `disagg.yaml` file, the services are: `Frontend`, `VllmDecodeWorker`, and `VllmPrefillWorker`. Each component has a field `dynamoNamespace` which defines the `namespace` part of the `Endpoint` that the worker will listen on. The `component` and `endpoint` parts of the worker `Endpoint` are not derived from the DGD but rather statically defined within the backend component itself. For instance, the `VllmDecodeWorker` component `Endpoint` has `component` part `backend` and `endpoint` part `generate`. `VllmPrefillWorker` component `Endpoint` has `component` part `prefill` and `endpoint` part `generate`.

The Frontend has a `ModelWatcher` that watches the KvStore for keys with the prefix `v1/mdc`. If the Frontend is deployed in the global namespace (`dynamo`), then all MDCs will be processed. If the frontend is deployed into a non-global namespace, than only MDCs that match the namespace will be processed (`v1/mdc/{namespace}.*`)

On a KvStore PUT event, the `ModelWatcher` will grab the model name (display_name) from the MDC and check if the `ModelManager` already contains an engine for that model. If there is an engine that exists but the MDC checksum (generated from select MDC fields) does not match, the `ModelWatcher` will discard the instance and log an error. Currently, the worker with discarded MDC will continue to run and consume resources without ever receiving traffic. If there is no engine for that model, an engine is created and stored on the `ModelManager`.

The model engine contains a `Client` constructed for the MDC `Endpoint`. This `Client` will watch the KvStore PUT events for keys with the prefix `v1/instance/{namespace}/{component}/{endpoint}`. When a PUT event is received, the `Client` will update the instance list.

# Issues

## Model Crosstalk

When deploying two DGDs for the different models, say llama3.1-70B and Qwen3-0.6B, if the DGDs are deployed in the same namespace (both have the same `dynamoNamespace` field value), the workers will register under the same `Endpoint`. In the vllm `disagg.yaml` example above, both workers for llama3.1-70B and Qwen3-0.6B will register under the `Endpoint` `{dynamoNamespace}/backend/generate`. Because the `Client` for both model engines is watching the same prefix `v1/instance/{dynamoNamespace}/backend/generate`, each `Client` will discover instances from both models, resulting in the `PushRouter` routing engine for llama3.1-70B for example sending requests to both sets of model workers.

TODO: temp solution - shared frontend

## Rolling Update Scenarios

### Update Dynamo Worker Version

### Update Frontend Worker Version

### Different MDC Checksums

### Incompatible KV Cache Transfer Mechanisms

https://github.com/ai-dynamo/enhancements/pull/28/files

- DGD name becomes namespace
- component name derived from componentType
- can have canary/rolling upgrade support using hash of component - append to namespace for supporting two worker pools for a model. solves differing MDCs and kv cache transfer compatibility
