# Multi-tenancy support for DynamoGraphDeployment

## Problem
1. Currently we dont have strong isolation between dynamo graph deployments.
Users expect a Dynamo namespace scoped frontend to serve models from same dynamo namespace. 
For example,two distinct DynamoGraphDeployment frontend pods should not serve models from the different namespaces.

2. Dynamo namespace is logic grouping of components but its not fully enforced across the entire system.
a. k8s CR `dynamoNamespace` is not used to isolate the models.


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
1. `DYNAMO_NAMESPACE` environment variable is used by components to scope their functionality.

2. when `DYNAMO_NAMESPACE` is not specified, `default` is the default namespace.

3. Frontend components (http, router, processor) are scoped to the dynamo namespace.
Advantages:
- Provides sharding ability to scale Routers independently.
- Provides dynamo namespace scoped sharding ability to scale all-in-one frontend components independently.

4. namespace itself can be hierarchial allowing heierchial isolation of Frontend components.
for example,
- Frontend components in `default` namespace can be used to serve models for all users.
- Frontend components in `default/model1` namespace can be used to serve model1 for all users.
- Frontend components in `default/model2` namespace can be used to serve model2 for all users.

Use cases:
1. A frontend launched without any `DYNAMO_NAMESPACE` will be scoped to `default` namespace and it will be able to serve models from all namespaces.

2. A frontend launched with specific `DYNAMO_NAMESPACE` will be scoped to it's namespace and all children namespaces.