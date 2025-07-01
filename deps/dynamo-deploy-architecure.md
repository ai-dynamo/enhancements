# Dynamo Deploy Re-architecture

## Context
Some notes/decisions from "The Two Verbs of Dynamo" meeting on 6/30
- `dynamo serve` will simply be used to launch exactly one component. It's role as a launcher will be replaced by `dynamo deploy`.
- `dynamo deploy` should be the launcher/orchestrator layer that is responsible for launching multiple components across platforms (k8s primarily, potentially local and others)

## Goals and Motivation
- [P0] Remove Kubernetes configs from @service decorator
  - The hierarchy of overrides across the service decorator, yaml file, and cli args is not intuitive to end users
  - Philosophically, there should not be tight coupling between the serve path and the deploy path. There is agreement that deploy should be a strict consumer of serve.
  - Code changes shouldn't be required to change deployment configuration.

- [P0] The DynamoDeployment CRD should be the PRIMARY focus of dynamo deploy team and the first thing that users are exposed to.
    - It's become apparent that when customers get to the deployment stage, they are extremely k8s savvy and are looking for all the levers that they can adjust.
    - The CRD is the meat and potatoes of our product.
    - Customers are MORE focused on:
        - CRD definition (together ai)
        - LWS/Grove (tencent)
        - Fluid cache (tencent)
        - GitOps management (together ai)
        - prebuilt images
        - SLA planner
        - Prometheus / NATS/ ETCD management
    - They seem to be LESS interested in:
        - archiving of python code into artifacts/artifact registry
        - automatic image builds in cluster
        - sdk language to define and connect abstract components
        - (these happen to be the things that trace their genealogy to bentoml which was for a separate audience (researchers))
    - Other peers (AIBRIx, llm-d) are all CRD first in their docs and UX. We are SDK first at the moment.
    - New changes introduced to the CRD must then be propagated to the SDK which is clunky and non-intuitive.

    We should make the CRD the first thing that users are exposed to. Let's remove artifact management and image builds from it and focus on re-architecting the CRD from the ground up.

- [P1] `dynamo deploy` should take as input a yaml file that is a thin shim to the CRD. We should get out of the business of managing artifacts and image builds. We should also get out of the business of managing the SDK and ship a superb operator.
  - Simplifies our focus, allows us all to converge on making cloud installation and deployment as seamless as possible.


## Dynamo IR

We still want to support a deploy.yaml file that is a thin shim to the CRD that can be used to deploy examples to k8s via the CLI. This is a convenience method to go from dynamo example to k8s deployment.

Draft from @biswa
```yaml
version: 1.0
name: my-graph
namespace: ns1
components:
  - name: frontend
    image: "ghcr.io/together-ai/dynamo-examples/frontend:latest"
    cmd: ["dynamo run..."]
    parameters:
       &vllmArgs
    resources:
       cpu: 200m
       gpu: 1
    replicas: 4
    environment:
      SAMPLE_CONFIG: A1
      DB_URI: "${{ secrets.DB_URI }}"
    secrets:
      - DB_URI
  - name: backend_1
    image: "ghcr.io/together-ai/dynamo-examples/backend1:latest"
    cmd: ["dynamo run..."]
    replicas: 2
    resources:
       cpu: 200m
       gpu: 1
```
