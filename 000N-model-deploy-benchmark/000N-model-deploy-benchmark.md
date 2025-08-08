# Simplify model benchmarking and deployment 

Problem: `Model deployment` and `benchmarking` are very complex to setup.

tldr; tight coupling between dynamo namespace, sla profiler code, k8s cr and backend config makes it hard to - 
1. tune the config for different models or parameters (for example vllm parameters)
2. change framework image 


Objective:
- Make it easy to quickstart: One command to deploy and benchmark a model name, mode and config
- Make it repeatable: users can reproduce benchmark results and diagnose issues
- Make it Simple: Simplify the setup and decouple image and config from the benchmark code
- ProvideRecipies: Lay down golden path to deploy and benchmark few models with best known config based on tuning

Principles:
- Use k8s CRD DynamoGraphDeployment as the base
- Decouple image, helm chart and recipies 
- Provie quick start helm chart/enhance dynamo operator to deploy and benchmark

# Design

## High level data model for deployment:
```yaml
dynamoNamespace:  xyz

model:
    name: Qwen/Qwen-0.6b
    shortName: qwen_v1
    extraConfig:
        # this is where the model config is available in the container
        # this will be passed to the backend component as 
        # an environment variable `DYNAMO_EXTRA_CONFIG` in the container
        path: /path/to/config.yaml
        # this is the configmap that contains the model config
        # this will be mounted to the container as a volume
        # at the path specified in the path field above
        configMapRef:
            name: my-model-config

deployment:
    # create the deployment if not exists
    create: true
    ### Deployment mode ###
    # below parameters are used to generate the k8s DynamoGraphDeployment CR
    # in helm chart or can be first class attributes in operator CR
    # enables disaggregation
    disagg: true
    # enables routing
    routing:
        enabled: true
        # routing policy: kv, random, round-robin
        policy: kv 

# enables running the benchmark job
benchmark
    enabled: true
    config:
        # benchmark configs

observability:
    enabled: true
```

## Alternative: Composible helm charts
We can leverage [Helmfile](https://github.com/helmfile/helmfile?tab=readme-ov-file#getting-started) to compose helm charts for different dynamo functionalities.
- Dynamo Operator (dynamo cloud platform)
- [Dynamo Inference Gateway helm chart](https://github.com/ai-dynamo/dynamo/blob/f7e468c7e8ff0d1426db987564e60572167e8464/deploy/inference-gateway/helm/dynamo-gaie/values.yaml#L27)
- Benchmark (TBD)
- Model Express (TBD)
- Troubleshooting (TBD)

## Phase 1: Quickly iterate with helm chart and publish for public usage

Similar reference structure is already used in  [Dynamo Inference Gateway helm chart](https://github.com/ai-dynamo/dynamo/blob/f7e468c7e8ff0d1426db987564e60572167e8464/deploy/inference-gateway/helm/dynamo-gaie/values.yaml#L27)

Based on the inputs in values.yaml, helm chart renders approrpriate k8s DynamoGraphDeployment CR/Services fragments.

## Phase 2: Update operator 

Incorporate the logic in operator after the UX above is validated/finalized after quick iteration.

##  todo
Image from brainstorming with Anis