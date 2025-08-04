# Multi-tenancy support for DynamoGraphDeployment

## Problem
1. Currently we dont have strong isolation between dynamo graph deployments.
Users expect a Dynamo namespace scoped frontend to serve models from same dynamo namespace. 
For example,two distinct DynamoGraphDeployment frontend pods should not serve models from the different namespaces.

2. Dynamo namespace is logic grouping of components but its not fully enforced across the entire system.
a. k8s CR `dynamoNamespace` is not used to isolate the models.

## What is a Dynamo namespace? Why do we need this?

Dynamo namespace is a way to logically partition control, data and event plane. This is a hybrid sharing model where we share some resources (operator/etcd/nats deployments, resources, data - pvc) within a k8s namespace and not others (logical component deployments) across multiple dynamo namespaces.

1. It helps multi-tenenacy use cases where each unit (model/user) has a dynamo namespace.
2. A/B test models in same namespace with model weights in RWX PVC volume.
3. Deploy 2 models in same k8s namespace and use Inference gateway to serve them. 
   Allow configuring granular model routing, Flow control and scheduling policies.

Although K8s namespace is the strongest isolation boundary for users and dont need the additional isolation boundary.


## Requirements

1. Users **SHOULD** be able to create `multiple independent` `DynamoGraphDeployment` (serving same or different models) within single k8s namespace by specifying different `dynamoNamespace` in k8s CR.
For example, I can create two dynamo graph deployments in same k8s namespace with same models with different parameters/backends and benchmark results.

2. Single Dynamo cloud deployment (etcd/nats/operator) **SHOULD** be able to serve models from multiple dynamo namespaces.

3. User **SHOULD** be able to deploy  in same k8s namespace using different dynamo namespaces.

## Design principles

- Dynamo namespace is a way to logically partition control, data and event plane.

- Reduce complexity and cognitive load. Reuse existing dynamo namespace as the isolation boundary.

- Minimize cross-contention in FE (http, router, processor) across different namespaces.

- Absolute share nothing architecture should use K8s namespace as the isolation boundary.


## Proposal

### Use K8s namespace as stronger isolation boundary:
Dynamo graph deployment in different k8s namespaces are fully isolated from each other.

They dont share any -
1. deployment (etcd, nats, operator, etc) 
2. resources (cpu, memory, etc)
3. data (PVC for models, etc)
4. components (http, router, processor or llm backends)

### Dynamo namespace as secondary isolation boundary within a k8s namespace:
#### Isolation:
This approach is a hybrid sharing model.

Shared:
1. deployment (etcd, nats, operator, etc) and its resources (cpu, memory, etc)
2. data (PVC for models, etc)

Not shared:
1. components (http, router, processor or llm backends)

#### High level design:
1. `DYNAMO_NAMESPACE` environment variable is used by components to scope their functionality.

2. when `DYNAMO_NAMESPACE` is not specified, `default` is the default namespace.

3. Frontend components (http, router, processor) are scoped to the dynamo namespace.
Advantages:
- Provides sharding ability to scale Routers independently.
- Provides dynamo namespace scoped sharding ability to scale all-in-one frontend components independently.

4. Dynamo namespace itself is hierarchial allowing heierchial isolation of Frontend components.
for example,
- Frontend components in `default` namespace can be used to serve models for all users.
- Frontend components in `llama-8b/version-A` dynamo namespace can be used to serve llama-8b model version-1 for all users.
- Frontend components in `llama-8b/version-B` dynamo namespace can be used to serve llama-8b model version-2 for all users.

Use cases:
a. A frontend launched without any `DYNAMO_NAMESPACE` will be scoped to `default` namespace and it will be able to serve models from all namespaces.

b. A frontend launched with specific `DYNAMO_NAMESPACE` will be scoped to it's namespace and all children namespaces.


### Implementation

#### Dynamo Operator changes:

Top level `dynamoNamespace` in DynamoGraphDeployment automatically sets `DYNAMO_NAMESPACE` environment variable in all components.

```yaml
apiVersion: dynamo.ai/v1alpha1
kind: DynamoGraphDeployment
metadata:
  name: dynamo-graph-deployment
spec:
  dynamoNamespace: default/model1
```

Similar changes are required in helm chart approach as well.

#### Dynamo Frontend components (http, router, processor):
They use `DYNAMO_NAMESPACE` environment variable to read from etcd and nats.
Phase 1:
- Ignore any etcd data/watch events for namespaces other than the specific namespace.
- Ignore any nats messages for namespaces other than the specific namespace.

Phase 2:
This phase will handle childrent namespaces as well.
same frontend can handle `llama/version-A` and `llama/version-B` namespaces.
- Subscribe to etcd data/watch events for specific namespace as prefix.
- Subscribe to nats messages for namespaces specified namespace as prefix.
- Ignore any event which doesn't match the prefix.

#### Dynamo Backend components (vllm,sglang, trtllm):
  - uses `DYNAMO_NAMESPACE` environment variable to scope their functionality.

Remove `--endpoint` argument in this format(`dyn://namespace.component.endpoint`) from all backend components.
1. namespace is read from `DYNAMO_NAMESPACE` environment variable. This makes it possible to use same backend components for multiple namespaces. it makes a python command line to be `relocatable` across dynamo namespaces by just changing the `DYNAMO_NAMESPACE` environment variable and not the command line arguments.

2. A component hosts multiple endpoints so `dyn://namespace.component.endpoint` is too specific. we dont use it in actual code.

3. components name is unique in a dynamo namespace. we can delegate this until actually we have a use case for multiple components with same name in a namespace.

Remove this argument from all backend components.
```python
  parser.add_argument(
        "--endpoint",
        type=str,
        default=DEFAULT_ENDPOINT,
        help=f"Dynamo endpoint string in 'dyn://namespace.component.endpoint' format. Default: {DEFAULT_ENDPOINT}",
    )
```