# ETCD-less Dynamo in Kubernetes: Endpoint Instance Discovery

## Background

This document is part of a series of docs attempting to replace the ETCD dependency in Dynamo using Kubernetes

Source: https://github.com/ai-dynamo/enhancements/blob/neelays/runtime/XXXX-runtime-infrastructure.md

ETCD usage in Dynamo can be categorized into the following:
- Endpoint Instance Discovery (Current focus)
- Model / Worker Discovery (Current focus)
- Worker Port Conflict Resolution
- Router State Sharing Synchronization

## ServiceDiscovery Interface

To de-couple Dynamo from ETCD, we define a minimal `ServiceDiscovery` interface that can be satisfied by different backends (etcd, kubernetes, etc). In Kubernetes environments, we will use the Kubernetes APIs to implement this interface.

### Methods
- `create_instance`(namespace: str, component: str, metadata: dict) -> Instance
    - Creates an instance and persists associated immutable metadata. Does not mark the instance as ready.
- `list_instances`(namespace: str, component: str) -> List[Instance]
    - Lists all instances that match the given namespace and component.
- `get_metadata`(namespace: str, component: str, instance_id: str) -> Instance
    - Returns the metadata for an instance.
- `subscribe`(namespace: str, component: str) -> EventStream
    - Subscribes to events for the set of instances that match (namespace, component). Returns a stream of events giving signals for when instances that match the subscription are created, deleted, or readiness status changes.
- `set_instance_status`(instance: Instance, status: str)
    - Marks the instance as ready or not ready for traffic.
- `get_instance_status`(instance: Instance) -> str
    - Returns the status of the instance.
<!-- Useful for MDC entry. Note tied to the instance lifecycle -->
- `write_kv`(key: str, value: str)
    - Writes data at a given key. Data not associated with any instance.
- `read_kv`(key: str)
    - Reads data at a given key. Data not associated with any instance.

### Overall Flow

At a high level, these are the service discovery requirements in a given disaggregated inference setup:

Credits: @itay

1. The frontend needs to know how to reach the decode workers
2. The frontend needs to know what model key to route to the above workers
3. The frontend needs to know some model specifics (like tokenizer) for the specified model
4. The decode worker needs to know how to reach the prefill workers (for the disagg transfer)

#### Frontend Discovers Decode Workers

- Decode calls `create_instance("dynamo", "decode", metadata)` and `set_instance_status(decode_worker, "ready")` to register itself with the service discovery backend. Its metadata includes the model it serves.
- Frontend calls `list_instances("dynamo", "decode")` and `get_metadata` to get all decode workers and the associated model information to bootstrap its cache.
- Frontend sets up a watch on the decode workers using `subscribe("dynamo", "decode")` to keep its cache up to date.

```python
# Frontend start:
decode_workers = service_discovery.list_instances("dynamo", "decode")
for decode_worker in decode_workers:
    # Fetch the associated metadata for the instance to register the model in-memory model registry
    metadata = service_discovery.get_metadata("dynamo", "decode", decode_worker.instance_id)
    model = metadata["Model"]
    # map the instance to the model in the in-memory model registry ...
```

#### Frontend Needs to Know what Model Key -> Instance Mapping

Addressed above.

#### Frontend Needs to Know Some Model Specifics (like Tokenizer)

- Decode writes a Model Card using the `write_kv` method. Frontend can simply read this entry.

#### Decode Worker Needs to Know how to Reach Prefill Workers

- Decode worker does the exact same thing as the frontend, instead listing and subscribing to the "prefill" component.


<!-- ### Simplifications and Assumptions -->

## API Reference

This table relates when each method is used in the context of endpoint instance discovery and model / worker management. We will also compare reference impls for each method in etcd and kubernetes.

### create_instance

This method is used to create an instance and persist associated metadata tied to the life of the instance. Notably, this metadata could capture immutable information associated with the instance such as model information (e.g what is currently stored in ETCD as `models/{uuid}`).

```python
# Example: Registration of a decode worker with associated model information
# Currently, this is being stored as part of the register_llm function in ETCD as `models/{uuid}`
metadata = {
    "Model": {
        "name": "Qwen3-32B",
        "type": "Completions",
        "runtime_config": {
            "total_kv_blocks": 24064,
            "max_num_seqs": 256,
            "max_num_batched_tokens": 2048
        }
    }
}
decode_worker = service_discovery.create_instance(namespace, component, metadata)
```

#### Kubernetes Reference Impl
- Asserts the pod has namespace and component labels that match up with method args. Fails otherwise.
- Asserts there is a Kubernetes Service that will select on namespace and component labels. (More on this later)
- Writes a ConfigMap with the attached metadata. ConfigMap name is a function of the pod name. Returns an Instance object.

Note: There are many ways to persist and access this immutable metadata. We can use ConfigMaps, a PVC with ReadWriteMany access, etc.

### get_metadata

This method is to fetch the metadata associated with an instance. For instance, the frontend can use this method to fetch the model information associated with an instance when it receives an event that a new worker is ready.

```python
# Context: frontend has watch setup on decode workers
stream = service_discovery.subscribe("dynamo", "decode")
for event in stream:
    switch event:
        case NewReadyInstanceEvent:
            # Fetch the associated metadata for the instance to register the model in-memory model registry
            metadata = service_discovery.get_metadata("dynamo", "decode", instance_id)
            model = metadata["Model"]
```

#### Kubernetes Reference Impl
- Fetches the associated ConfigMap with the attached metadata.

Note: There are many ways to persist and access this immutable metadata. We can use ConfigMaps, a PVC with ReadWriteMany access, etc.

### subscribe

This method is to subscribe to events for the set of instances that match (namespace, component). Returns a stream of events giving signals for when instances that match the subscription are created, deleted, or readiness status changes. This can be used by the frontend to maintain its inventory of ready decode workers. It can be used by decode workers to maintain its inventory of ready prefill workers.

```python
# Context: frontend has watch setup on decode workers
stream = service_discovery.subscribe("dynamo", "decode")
for event in stream:
    switch event:
        case NewReadyInstanceEvent:
            metadata = service_discovery.get_metadata("dynamo", "decode", instance_id)
            model = metadata["Model"]
            # register the model in-memory model registry
            # map the instance to the model
```

#### Kubernetes Reference Impl
- Sets up a kubectl watch for EndpointSlices corresponding to the Service that selects on the namespace and component labels.
- Returns a stream of events that inform us when a READY pod matching the namespace and component is up.

### set_instance_status / get_instance_status

This method is to mark the instance as ready or not ready for traffic.

```python
# Context: decode worker has finished loading the model
metadata = {
    "Model": {
        "name": "Qwen3-32B",
        "type": "Completions",
        "runtime_config": {
            "total_kv_blocks": 24064,
            "max_num_seqs": 256,
            "max_num_batched_tokens": 2048
        }
    }
}
decode_worker = service_discovery.create_instance(namespace, component, metadata)
# Set up endpoints/transport. This can be http/grpc/nats. When the underlying transport is ready to serve, mark the instance as ready for discovery.
await start_http_server()
# When the server is ready to serve, mark the instance as ready for discovery.
service_discovery.set_instance_status(decode_worker, "ready")
```

#### Kubernetes Reference Impl
- The readiness probe of the pod will proxy the result of `get_instance_status`. When the instance will be ready for traffic, the readiness probe will return 200 and EndpointSlices will be updated to include the instance.
- Components that have called `subscribe` on this component will be notified when the instance is ready for traffic. They can use this instance to route traffic to using their transport.

### write_kv / read_kv

This method is to write and read data at a given key. This is a useful place to store the model card (e.g `mdc/{model-slug}`) and other model specific information.

```python
# Context: decode worker is registering the MDC before it marks itself as ready
service_discovery.write_kv("mdc/Qwen3-32B", mdc)
```

#### Kubernetes Reference Impl
- Writes a ConfigMap with the attached data. ConfigMap name is a function of the key.

## Kubernetes EndpointSlice Discovery Mechanism

This section provides a detailed explanation of how the ServiceDiscovery interface is implemented using Kubernetes EndpointSlices for service discovery.

### Core Components

The Kubernetes implementation relies on three main Kubernetes resources:

1. **Pods**: Each worker instance runs as a pod with specific labels
2. **Service**: Selects pods based on namespace and component labels
3. **EndpointSlices**: Automatically managed by Kubernetes, tracks ready pod endpoints

### Resource Structure

#### Pod Labels
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: dynamo-decode-pod-abc123
  labels:
    nvidia.com/dynamo-namespace: dynamo
    nvidia.com/dynamo-component: decode
spec:
  readinessProbe:
    httpGet:
      path: /health
      port: 8080
    initialDelaySeconds: 5
    periodSeconds: 10
```

#### Service Selector
```yaml
apiVersion: v1
kind: Service
metadata:
  name: dynamo-decode-service
spec:
  selector:
    nvidia.com/dynamo-namespace: dynamo
    nvidia.com/dynamo-component: decode
  ports:
  - port: 8080 # The port itself is a dummy port. It is not used for traffic (although it could be).
    targetPort: 8080
    name: http
```

#### EndpointSlice (Auto-generated)
```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: dynamo-decode-service-abc123
  labels:
    kubernetes.io/service-name: dynamo-decode-service
    nvidia.com/dynamo-namespace: dynamo
    nvidia.com/dynamo-component: decode
addressType: IPv4
endpoints:
- addresses: ["10.1.2.3"]
  conditions:
    ready: true  # Based on pod readiness probe
  targetRef:
    kind: Pod
    name: dynamo-decode-pod-abc123
    namespace: default
ports:
- name: http
  port: 8080
  protocol: TCP
```

### Discovery Flow

1. **Instance Registration**: When `create_instance()` is called, the pod creates a ConfigMap with instance metadata
2. **Readiness Signaling**: When `set_instance_status(instance, "ready")` is called, the pod's readiness probe starts returning 200
3. **EndpointSlice Update**: Kubernetes automatically updates the EndpointSlice to mark the endpoint as ready
4. **Event Propagation**: Clients watching EndpointSlices receive events about the ready instance

### Event Mapping

The `subscribe()` method maps EndpointSlice events to ServiceDiscovery events:

| EndpointSlice Event | ServiceDiscovery Event | Description |
|---------------------|------------------------|-------------|
| Endpoint added with `ready: true` | `NewReadyInstanceEvent` | New instance is ready for traffic |
| Endpoint condition changes to `ready: true` | `InstanceReadyEvent` | Existing instance becomes ready |
| Endpoint condition changes to `ready: false` | `InstanceNotReadyEvent` | Instance becomes unavailable |
| Endpoint removed from slice | `InstanceDeletedEvent` | Instance is terminated |

### Implementation Details

#### Instance ID Resolution
- **Instance ID**: Pod name (e.g., `dynamo-decode-pod-abc123`)
- **EndpointSlice Reference**: `targetRef.name` points to the pod name
- **Metadata Lookup**: ConfigMap name derived from pod name for `get_metadata()`

#### Watch Implementation
```python
# Kubernetes watch setup for subscribe()
watch_filter = {
    "labelSelector": f"kubernetes.io/service-name=dynamo-{component}-service"
}
endpoint_slice_watch = k8s_client.watch_endpoint_slices(filter=watch_filter)

for event in endpoint_slice_watch:
    if event.type == "MODIFIED":
        for endpoint in event.object.endpoints:
            if endpoint.conditions.ready:
                emit_event(NewReadyInstanceEvent(
                    instance_id=endpoint.targetRef.name,
                    address=endpoint.addresses[0]
                ))
```

### Benefits of EndpointSlice Approach

1. **Native Kubernetes Integration**: Leverages built-in service discovery
2. **Automatic Cleanup**: Pods and EndpointSlices are cleaned up when instances terminate
3. **Scalability**: EndpointSlices handle large numbers of endpoints efficiently
4. **Consistency**: Kubernetes ensures eventual consistency across the cluster
5. **Health Integration**: Readiness probes directly control traffic eligibility
