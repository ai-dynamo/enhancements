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

- Reduce complexity and cognitive load. Reuse existing dynamo namespace as the isolation boundary.

- Minimize cross-contention in FE (http, router, processor) across different namespaces.

- Absolute share nothing architecture should use K8s namespace as the isolation boundary.

## Proposal
1. `DYNAMO_NAMESPACE` environment variable is used by components to scope their functionality.
    dont use it to isolate the models.

2. when it is not specified, `default` is the default namespace.

3. Frontend components (http, router, processor) are scoped to the dynamo namespace.
Provides sharding ability to scale Routers independently.
