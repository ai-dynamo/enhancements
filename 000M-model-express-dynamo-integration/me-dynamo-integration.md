# Model Express ~ Dynamo Integration Proposal
**Status**: Draft
**Authors**: Biswa Panda
**Category**: Architecture
**Sponsor**: Neelay, Ganesh
**Required Reviewers**: Nick, Ganesh, Kavin,  Alec, Graham
**Review Date**: 2025-08-20

# Summary
# Motivation
## Goals
### Non Goals
## Requirements

### REQ 1: Model Express can be deployed as a per namespace service as part of Dynamo Operator
### REQ 2:  

# Proposal

## Operator Configuration:
Following model express configuration will be added to the Dynamo Operator configuration.
This will enable Model Express to be deployed as a standalone service as part of Dynamo Operator.
Also it will automatically inject an environment variable (`MODEL_EXPRESS_URL` or better name) to the component.

```yaml
# top level attribute in DynamoOperator CRD
spec:
    services:
        componentType: ModelExpress
        dynamoNamespace: <DYN_NAMESPACE> / "global"
        ModelExpress:
            pvc:
                name: model-express-pvc
                size: 10Gi
                storageClass: standard
                create: true

```

## Model express sharing between dynamo namespaces

Model express can be deployed as a shared service across all dynamo namespaces  by specifiying it in top level attribute in DynamoGraphDeployment CRD.

```yaml
# top level attribute in DynamoOperator CRD
spec:
    services:
        componentType: ModelExpress
        dynamoNamespace: "global"     # shared service across all dynamo namespaces
        ModelExpress:
            pvc:
                name: model-express-pvc
                size: 10Gi
                storageClass: standard
                create: true
```

For multi-tenant deployment in a single k8s namespace, model expresses can be deployed as a per `DYN_NAMESPACE` basis.

```yaml
# top level attribute in DynamoOperator CRD
spec:
    services:
        componentType: ModelExpress
        dynamoNamespace: "MY_DYN_NAMESPACE"  # per namespace model express service  
        ModelExpress1:
            pvc:
                name: model-express-pvc
                size: 10Gi
                storageClass: standard
                create: true
```

## Alternate Solutions

### top level attribute in DynamoOperator CRD
Shared model express service across all dynamo namespaces:
```yaml
# top level attribute in DynamoOperator CRD
ModelExpress:
    dynamoNamespace: <DYN_NAMESPACE> / "global"
    pvc:
        name: model-express-pvc
        size: 10Gi
        storageClass: standard
        create: true
    image: ...
```

### Create a separate helm chart for model express and compose with Helmfile


## Model Express Integration with dynamo components:

When model express is enabled, env variable `MODEL_EXPRESS_URL` will be injected to each component.

This also acts as a feature flag to enable/disable model express for dynamo components.

A common model initialization flow can be used across backend components (to facilitate sharing of model express between components). This flow will use model express lib to fetch models from model express.

- `MODEL_EXPRESS_PATH` env variable will be set if we are using a common PVC path for model express in earlier phases.

### Model express disabled:
When `MODEL_EXPRESS_URL` is not set, model express will be disabled for the component.
1. model initialization flow will skip fetching the model from model express.
2. Backend components will use huggingface hub libs directly. (This is the current behavior)

### Model express enabled:
When `MODEL_EXPRESS_URL` is set, model express will be enabled for the component.

1. model initialization flow will use model express lib to fetch the model from model express.
2. Backend specific initialization flow [for example: vllm]() will start after model is fetched from model express.

Backend UX and command line will remain unchanged in the deployment spec. we'll internally use model express lib to fetch the model by changing the command line arguments for the backend. 
For example: 

deployment spec:
```
python3 -m dynamo.vllm --model Qwen/Qwen3-0.6B 
```

Internally, we'll change the vllm args to use model express lib to fetch the model.

1. 1st phase: using shared pvc: 
`MODEL_EXPRESS_URL` env variable will be set to model express service url.
`MODEL_EXPRESS_PATH` env variable will be set to the pvc path.

a. call model express lib to fetch the model to `MODEL_EXPRESS_PATH`
b. we'll set appropriate HF env vars - [HF_HUB_OFFLINE](https://huggingface.co/docs/huggingface_hub/en/package_reference/environment_variables#hfhuboffline), [HF_HOME] mapping to `MODEL_EXPRESS_PATH`