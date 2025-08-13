# Simplify model deployment and auxiliary utilities: benchmarking

Problems: 
1. `Model deployment` is hard to configure and run. We need a standardized way to configure and quickly launch hand picked models with given backend, (disagg/router) mode and config.

2. Auxiliary utilities like `benchmarking` are hard to configure and run.
Tight coupling between dynamo namespace, sla profiler code, k8s cr and backend config makes it hard to -
1. tune the config for different models or parameters (for example vllm parameters)
2. framework image 
3. backend config

Objective:
- decouple config from framework image: this will simplify the model deployment and benchmarking
- easy quickstart for users: a reference quickstart to deploy a model and with benchmarking with minimal steps
- well teseted recipies: deploy and tune fewer models to generate best configs for benchmarking
- composible helm charts: use helmfile to deploy all the components in composible way

Principles:
- Use k8s CRD DynamoGraphDeployment as the base
- Decouple image, helm chart and recipies 
- Provie quick start helm chart/enhance dynamo operator to deploy and benchmark

# Design

## High level data model for deployment:

For example, below yaml can be used to deploy a Qwen model with vllm backend, disagg mode and kv routing in `qwen-test` dynamo namespace. 

Additionally, vllm parameters are passed as configmap ref `my-model-config` in the container.

```yaml
dynamoNamespace:  qwen-test

model:
    # below parameters are used to generate the k8s DynamoGraphDeployment CR
    name: Qwen/Qwen-0.6b
    backend:
        # name of the backend: vllm, trtllm, sglang
        type: vllm
        image: dynamo-vllm:0.1.0
    deployment:
        # create the deployment if not exists
        create: true
    decode:
        extraConfig:
            # this is where the model config is available in the container
            # this path will be passed to the backend component as 
            # an environment variable `DYNAMO_EXTRA_CONFIG` in the container
            # default path is /opt/dynamo/model/config.yaml
            path: /opt/dynamo/model/config.yaml
            # this is the configmap that contains the model config
            # this will be mounted to the container as a volume
            # at the path specified in the path field above
            configMapRef:
                name: my-model-prefill-config
    prefill:
        extraConfig:
            path: /opt/dynamo/model/config.yaml
            configMapRef:
                name: my-model-decode-config
    ### Deployment mode ###
    # in helm chart or can be first class attributes in operator CR
    # enables disaggregation
    disaggregation: true
    # routing policy: none, kv, random, round-robin
    routingPolicy: kv

# enables running the benchmark job
benchmark:
    enabled: true
    config:
        # benchmark configs

observability:
    enabled: true
```

This is a high level data model can be used as -
1. a values.yaml for a parent helm chart to deploy a model and auxiliary utilities like benchmark, inference gateway etc.
2. in next phase, this can be absorbed by k8s DynamoGraphDeployment CR to be used by operator

A high level (parent helm chart) can take above data model as input and render the k8s DynamoGraphDeployment CR and auxiliary utilities like benchmark, inference gateway etc.

## Alternative 1: Composible helm charts
We can leverage [Helmfile](https://github.com/helmfile/helmfile?tab=readme-ov-file#getting-started) to compose helm charts for different dynamo functionalities.

Single helmfile can be used to deploy below components in composible way. a reference would be [wide-ep-lws example in llm-d].(https://github.com/llm-d-incubation/llm-d-infra/tree/main/quickstart/examples/wide-ep-lws)

Helm charts:
- [Dynamo cloud platform](https://github.com/ai-dynamo/dynamo/tree/main/deploy/cloud/helm) is the base helm chart and deploys operator for managing life cycle of the graph deployment, grove integration, etc.
  - current state: independent [Dynamo cloud platform](https://github.com/ai-dynamo/dynamo/tree/main/deploy/cloud/helm) helm chart
- Dynamo Inference Gateway helm chart
  - current state: independent [Dynamo Inference Gateway helm chart](https://github.com/ai-dynamo/dynamo/blob/f7e468c7e8ff0d1426db987564e60572167e8464/deploy/inference-gateway/helm/dynamo-gaie/values.yaml#L27)
- Metrics
  - current state: we dont have helm chart but we have few yaml files with env variables [Metrics](https://github.com/ai-dynamo/dynamo/tree/main/deploy/metrics/k8s)
- Benchmark
  - current state: few hard-coded jobs in `benchmark` [folder](https://github.com/ai-dynamo/dynamo/tree/main/benchmarks/profiler/deploy)
- Model Express
  - current state: hard-coded yaml for single config in agg mode [model-express](https://github.com/ai-dynamo/modelexpress/pull/31/files)
- Fault injection/Test
  - Doesn't exist
- Troubleshooting
  - Doesn't exist


### Other Alternative considerations
1. use environment variables (current approach)
- we are reinventing template rendering with `envsubst` in non-sustainable way 
- this is not ideal extensible quickstart for users

2. use `kustomize` to render the template
- this is a sane way for customization and better than env vars + `envsubst`
- this can be derieved from helm charts (use `helm template` to get the k8s yaml)
- helmfile supports `kustomize` to render the template. [reference](https://helmfile.readthedocs.io/en/latest/advanced-features/#deploy-kustomizations-with-helmfile)


## Phase 1: Quickly iterate on helm chart and publish for public usage/feedback

Similar reference structure is already used in  [Dynamo Inference Gateway helm chart](https://github.com/ai-dynamo/dynamo/blob/f7e468c7e8ff0d1426db987564e60572167e8464/deploy/inference-gateway/helm/dynamo-gaie/values.yaml#L27)

Based on the inputs in values.yaml, helm chart renders approrpriate k8s DynamoGraphDeployment CR/Services fragments.

## Phase 2: Stabilize and Update operator 

Incorporate the logic in operator after the UX above is validated/finalized after quick iteration.
