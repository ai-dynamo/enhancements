# Dynamo Deploy v2

Given that we have a working version of our platform that has accumulated some early feedback, it might be a good idea to revisit the design and architecture of the platform

## Dynamo Deploy v1 Feedback (Inspired by bentoml)

It's become apparent that when customers get to the deployment stage, they are extremely k8s savvy and are looking for all the levers that they can adjust.
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
      - going from python code to a k8s deployment in a single step
      - (these happen to be the things that trace their genealogy to bentoml which was for a separate audience (researchers))
  - Other peers (AIBRIx, llm-d) are all CRD first in their docs and UX. We are SDK first at the moment in our docs and UX.
  - New changes introduced to the CRD must then be propagated to the SDK which is clunky and non-intuitive.

We should make the CRD the first thing that users are exposed to. Let's remove artifact management and image builds from it and focus on re-architecting the CRD from the ground up.

## Re-architecting dynamo deploy focused on Kubernetes experience

### 1. Dynamo deploy should NOT be involved in artifact management and image builds.

Right now, `dynamo build` builds an archive of code and persists it on the platform to later build an image out of it. I propose we remove the idea of a dynamo artifact from our platform and simplify to user-vendored images.

Current pitfalls with this approach:
  - Concept of archiving python code -> pushing to remote api store -> building image remote -> pushing to registry is confusing. There are too many entities in our system (dynamo artifact, images, dynamo manifest, etc)
  - Customers are already asking to build their own images outside of the image build path (hannah and julien driving this). This should be the native path, not an alternative path.
  - Requires registry configuration in onboarding process
  - CRs are linked to artifacts in the api-store rather than images (which is the standard)

How does this change impact our platform?
- we remove the notion of a dynamo artifact (archive containing python code and dynamo.yaml manifest)
- dynamo build/deploy no longer push an archive of packaged python code.
- disable image build apparatus

### 2. DynamoDeployment CRD should be the FOCUS of the platform.

We've built a cool operator that can deploy dynamo components (frontend, router, etc) to k8s and should make it front and center in our docs and UX.

**The CRD should be the Source of Truth for the capabilities of the platform.**

Advantages to approach:
- Focuses on customer feedback
- Brings parity with other peers (AIBRIx, llm-d) who use CRDs for their platform

Proposed changes:
- New capabilities (fluid cache, lws, sla planner) should present as self-documenting fields in the CRD. Users should be able to check the CRD definition page to see what new capabilities are available in deployment.
- DynamoGraphDeployment operates on images, all notion of dynamo artifacts are removed.

### 3. Dynamo deploy phases out involvement in the SDK

Inputs to dynamo deploy is a yaml file where each component is represented by an `image`/`args`
