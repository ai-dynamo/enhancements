# Dynamo Probes: Liveness and Readiness

This is an exploratory document the current state of health checks in Dynamo and whether we can make them align more with the Kubernetes model.

## Existing Dynamo Health Check Workflow

In the context of health checks, we are interested in solving the following problems:

1. How do new, healthy instances onboard to receive traffic?
2. How are unhealthy/unresponsive instances recognized?
3. How do we stop sending traffic to a bad instance?

Dynamo Runtime solves most of these problems using NATS and ETCD.

### How do new instances register to receive traffic?

Here is the rough flow of how a new endpoint instance comes to process requests:

1. An instance of an endpoint (1 replica) acquires a new lease in ETCD. ETCD publishes an event on its behalf informing the other clients of the DRT of the new instance.
2. The new instance subscribes to a subject (topic) in NATS. The instance will dequeue from the NATS subject as requests come in.
3. (Enqueueing) Now aware of the newly joined instance, other clients of the DRT may enqueue requests to the NATS subject corresponding to the instance.
4. (Dequeueing) The instance will dequeue requests from the NATS subject and process them.

### How are unhealthy/unresponsive instances recognized?

1. Instances will periodically send a heartbeat to ETCD to renew their lease.
2. If an instance fails to renew its lease, ETCD will expire the entry and publish an event to the other members of the DRT.

### How do we stop sending traffic to a bad instance?

1. Other clients of DRT will receive an event from ETCD when a given instance is no longer available. This will cause them to not write requests to the NATS subject corresponding to the defunct instance.

## Differences between Kubernetes and Dynamo

### Heartbeat vs Probe

- Kubernetes: K8s controller actively probes container for readiness/liveness. (Probe model)
- Dynamo: Instance is responsible for sending a heartbeat to ETCD to renew its lease. (Heartbeat model)

### Pull based vs push based transport

- Kubernetes: Kubernetes pushes traffic to replicas behind a Kubernetes service directly.
- Dynamo: Clients push requests to NATS. Instances then pull from their corresponding NATS queue. (Requests are brokered)

### Individually addressable instances

- Kubernetes: Replicas/instances are abstracted behind a Kubernetes service DNS. It is not possible to address a specific replica/instance directly.
- Dynamo: Instances are individually addressable. Clients can push requests to queue corresponding to a given instance.

## Brainstorm: Implementing DRT using Kubernetes

Given that Kubernetes has a mature solution for service discovery, routing, and health checks, one option could be to offer an alternative implementation of the DRT for Kubernetes. 
In this alternative impl, we would not need to use NATS/ETCD and instead rely on battle-tested Kubernetes primitives to solve the same set of problems.

Key differences:
- Transport between clients and instances is non-brokered and could take place via HTTP/GRPC eliminating the need for NATS.
- Management of instances could be delegated to Kubernetes using built in system of health checks/selectors.

Considerations (items that need to still be hashed out):
- client.direct(instance) # this is not possible in K8s as replicas are abstracted behind a service
- we might need l7 routing to route requests to the correct instance (istio)

