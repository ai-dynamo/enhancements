# Dynamo integration with Inference Gateway

**Status**: Draft

**Authors**: [Biswa Panda](https://github.com/biswapanda) 

**Category**: Architecture

**Replaces**: [Link of previous proposal if applicable] 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: Itay, Maksim, Neelay

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

This proposal outlines the integration of Dynamo components with the Gateway API Inference Extension. 

## Acronyms

**EPP:** Endpoint Picker Protocol
**IGW:** Inference Gateway

## Goals

* Support Inference gataway concepts in Dynamo 
* Maintain backward compatibility with existing EPP functionality
* Reuse dynamo components
* Minimize network hops

### Non Goals

* Replace existing EPP internal scheduling
* Modify core Gateway API specifications
* Change existing Dynamo worker interfaces significantly

## Requirements

### REQ 1 External Processing Integration

Dynamo EPP (Endpoint picker) **MUST** support calling Frontend and processor for request preprocessing and scheduling while maintaining the existing ext-proc interface.

### REQ 3 Unified Dynamo deployment

Dynamo EPP and components (Frontend, Processor, Router, Workers) **MUST** be deployable within Kubernetes through a unified helm chart to maintain version compatibility.

### REQ 4 Maintain compatibility with Inference Gateway protocols

Dynamo EPP **MUST** be compatible with Inference Gateway API and concepts (InferencePool, InferenceModel)

# Proposal

## Architecture Overview

This architecture unifies Inference Gateway with Dynamo Graph deployment. See diagram below for detailed component interactions.

![Architecture Diagram](./arch.png)

### Data flow

1. The client sends an HTTP inference request to the Gateway.
2. Gateway receives the request and extracts the model name and relevant metadata.  
   Gateway consults the InferenceModel configuration to determine the inference pool (dynamo graph) to route the request.
3. Gateway calls EPP over grpc for worker scehduling based on envoy ext_proc protocol.
4. EPP forwards the request to Frontend sidecar
```yaml
Request: 
    - req header: set x-routing-request: true
    - req body: original request body (For example, Chat completion request)

Respose:
    worker_id: this is dynamo specific worker_id
    token_ids: (Optional) tokens generated from processor step
```
4. Dynamo Frontend accepts OAI request and forwards request through dynamo request plane (nats)
5. Dynamo Processor performs necessary pre-processing and generates tokens. It calls routers to decide worker_id.
6. Dynamo Router takes tokens as input and decides worker_id based on scheduling policies and KV metrics
7. EPP sets headers (x-gateway-destination-endpoint and x-gateway-worker-id)
Optional optimization: We can inject the tokens in request body to avoid recomputing tokens in service path.
Note: `tokens` key in request body is not OpenAI compatible.
```
Set Req Header: 
- `x-gateway-destination-endpoint`: worker address of the Dynamo frontend pod
- `x-gateway-worker-id`: Dynamo worker id of Backend LLM worker instance 

Add to Req Body (Optional):
- `tokens`
```
8. IGW forwards the request to appropriate Dynamo frontend based on request header `x-gateway-destination-endpoint`
Note: This could be ideally routed to Frontend service because Frontend/Processor deployment is decoupled from LLM workers.

9. Processor skips pre-processing 
- `tokens` in request body and skips pre-processing step
- `x-gateway-worker-id` in the request and skips call to router

10. Request is sent to LLM Backend and response is streamed back through 
- processsor: Postprocessing steps
- Frontend: Change response shape from Dynamo native to OpenAI compatible response

**Notes:**
- All inter-component communication within Dynamo (Processor, Router, Workers) uses NATS with two-part JSON messages.
- Deployment is unified via a single Helm chart for version compatibility.

### Decision Points

#### 1. EPP integration with Dynamo: plugin vs sidecar vs external callout service

![EPP integration with Dynamo](./alt-epp-dyn.png)
##### sidecar container (Preffered)
Needs support in EPP to deploy a sidecar container and specify the port to request at.

Pro
- Reduced network hops: Direct communication between EPP and Dynamo components within the same pod
- Lower latency: No network overhead for inter-component communication
- Simpler deployment and management: Deployed as a single unit, easier to manage lifecycle

Con
- Tightly coupled scaling: Scaling decisions for EPP and Frontend are coupled
- Deployment of EPP is coupled with Dynamo sidecar image. Version upgrades should be done in-sync. 

##### external callout service
Pro
- completely isolated deployments 
- Each component can be deployed and scaled independently

Con
- Additional network hops: More latency due to network communication between services
- Service discovery complexity: Need to manage service endpoints and load balancing
- Additional network failure points in the request path

##### plugin
Pro
- Minimum number of network hops
- Simpler architecture without additional layer
- Lower latency for request processing

Con
- Dynamo runtime/component don't have native integration with golang
- Hard to scale across models
- Tight coupling with golang based implementation

#### 2. Dynamo component co-location
Issue: Higher latency due to several network hops
This is a dynamo specific problem/question. It's orthogonal to IGW but correlated to some extent. (may be it'd be a separte DEP)

![Dynamo component co-location](./dyn_comp_deployment.png)

##### Alt.1: Single binary/pod
 component is deployed as independently scalable deployment.
+ lower latency
+ Reduced network hops
+ tight coupling
- Scaled as a unit

##### Alt.2: Separate deployment
Each component is deployed as independently scalable deployment.  
This is current state.

+ simpler to scale
- More hops

##### Alt.3: Frontend as entrypoint and Processor/Router as sidecars
+ easier to deploy and manage
- Scaled as a unit

## Problems
1. Currently EPP schedluling has tightly coupling with in-porcess preprocessing.
  It's hard to scale/maintain it accross different models.

2. double tokenization during scheduling and service path

## Guiding Principles

1. Composibiltiy: EPP should externalize scheduling decision to dynamo router
2. DRY: Aim to reduce duplications in preprocessing steps (tokenization, prompt template application)
3. Compatibility: Maintain full compatibility with inference gateway api
4. Reduce network hops to minimize tail latency

## Design constraints
- Dynamo componetns (processor, router) use dynamo native transport (two part json messsages over nats)
- Dynamo does not support co-scheduling in disaggregated mode. Currently request flow goes from decode to prefill.

## Current state of IGW and Dynamo

### Dynamo Graph deployment
A `Dynamo Graph` contains one or more `Dynamo Component`s and this one-to-many relation is reflected in corresponding Kubernetes deployment Kubernetes CRs DynamoGraphDeployment and DynamoComponentDeployments respectively.

Each dynamo component deployment creates a Kuberenetes deployment which manages component's pods.

![Dynamo Graph Deployment](./graph_deployment.png)

| Module | Dynamo | IGW
| :---- | :---- |
| **Event Plane** | Push based KV/capacity related metric events using Nats | Scrapers populate Datastore with metrics for a pod (pull based)
| **Service/Data Plane** | Custom nats/tcp based protocol, uses json serialization | Standard HTTP based protocol
| **Control Plane** | Planner is responsible for scaling decisions, Orchestration happens via operator | TODO


### Inference Gateway Request Flow:
```
HTTP Request
     │
     ▼
┌─────────────┐    Extract model name   ┌──────────────────┐
│   Gateway   │ ──────────────────────► │ InferenceModel   │
│ (HTTPRoute) │                         │ (Model Config)   │
└─────────────┘                         └──────────────────┘
     │                                           │
     │ Route to backend                          │ References
     ▼                                           ▼
┌─────────────┐    Smart routing via     ┌──────────────────┐
│InferencePool│ ◄─────────────────────── │ Endpoint Picker  │
│ (Compute)   │      EPP extension       │ Extension (EPP)  │
└─────────────┘                          └──────────────────┘
     │
     ▼
┌─────────────┐
│ Model Server│
│    Pods     │
└─────────────┘
```

## Deferred to Implementation
- Fallback mechanisms for failures
- Metrics and observability integration

# Alternate Solutions

## Alt 1 Entire Dynamo Graph as a blackbox

![Dynamo Graph as a blackbox](./alt_dyn_black_box.png)

**Pros:**
+ Simple to deploy
+ Gateway+EPP deployment is orthogonal to Dynamo cloud/graphs deployment

**Cons:**
- Unable to reuse Dynamo KV router component
- Metrics Service Protocol: Currently dynamo components are not MSP compatible

# Alt 2: Tokenization Extension Chain
Instead of embedding tokenization in EPP, create a dedicated tokenization extension that runs before EPP in the processing chain:

```
Client -> Tokenization Extension -> EPP ->  Model Server
```

Pro:
- Follows Gateway API's extensible processing chain philosophy
- Separates concerns
- Can be reused across different routing strategies

Con:
- complicated deployment
- Additional network hop
- More complex chain management

# Related Proposals
* [Gateway API Inference Extension Documentation](https://gateway-api-inference-extension.sigs.k8s.io/)
* [Envoy External Processing Filter](https://www.envoyproxy.io/docs/envoy/latest/configuration/http/http_filters/ext_proc_filter)
* [Gateway API Specification](https://gateway-api.sigs.k8s.io/)