# Simplify model deployment and auxiliary utilities: benchmarking

## Problems: 
1. Missing UX around model deployment and auxiliary utilities.

- Use case 1: I want a simple quickstart reference to deploy a model and optionally run auxiliary utilities like benchmarking, inference gateway, model express, etc.

- Use case 2: I want to specify many configs for trtllm backend and dont want to list all arguments in k8s CR.

- Use case 3: I want to deploy and reproduce `perf benchmarks` for a specific model. 
    This is hard to do now due to tight coupling between dynamo namespace, SLA profiler code, k8s CR and backend config

2. I want to pass configs to the container instead of listing all arguments in k8s CR. This is not manageable for trtllm backend with many configs.


## Objective:
- decouple config from framework image: this will simplify model deployment and benchmarking
- easy quickstart for users: a reference quickstart to deploy a model with benchmarking in minimal steps
- well-tested recipes: deploy and tune fewer models to generate best configs for benchmarking
- composable helm charts: use helmfile to deploy all the components in a composable way

## Principles:
- Use k8s CRD DynamoGraphDeployment as the base
- Decouple image, helm chart and recipes
- Provide quick start helm chart/enhance dynamo operator to deploy and benchmark

# Design

## High-level data model for deployment:

For example, the YAML below can be used to deploy a Qwen model with vLLM backend, disagg mode and KV routing in the `qwen-test` dynamo namespace. 

Additionally, vLLM parameters are passed as configmap ref `my-model-config` in the container.

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
    decode: # this is the name of the component
        extraConfigs:
        -  name: model-config      # name of the config 
           # this path will be passed to the backend component as 
           # an environment variable (default: `DYNAMO_EXTRA_CONFIG`) in the container
           env: DYNAMO_EXTRA_CONFIG
           # this is where the model config is available in the container
           # (default path: /opt/dynamo/model/config.yaml)
           path: /opt/dynamo/model/config.yaml 
           # this is the configmap that contains config
           configMapRef:
               name: my-model-decode-config
    prefill: # this is the name of the component
        extraConfigs:
          - name: prefill-config
            configMapRef:
                name: my-model-prefill-config
    ### Deployment mode ###
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

This high-level data model can be used as:
1. a values.yaml for a parent helm chart to deploy a model and auxiliary utilities like benchmark, inference gateway, etc.
2. in the next phase, this can be absorbed by k8s DynamoGraphDeployment CR to be used by the operator

A high-level (parent helm chart) can take the above data model as input and render the k8s DynamoGraphDeployment CR and auxiliary utilities like benchmark, inference gateway, etc.

### Passing configs to the container

Each component can be configured with a k8s configmap ref which is mounted at a path specified in `DYNAMO_EXTRA_CONFIG` environment variable in the container.
 
```yaml
extraConfig:
    # this is where the extra config is available in the container
    # this path will be passed to the backend component as 
    # an environment variable `DYNAMO_EXTRA_CONFIG` in the container
    # default path is /opt/dynamo/model/config.yaml
    path: /opt/dynamo/model/config.yaml
    # this is the configmap that contains the model config
    # this will be mounted to the container as a volume
    # at the path specified in the path field above
    configMapRef:
        name: my-model-prefill-config
```

configmap example:

```yaml
apiVersion: v1  
kind: ConfigMap
metadata:
  name: my-model-prefill-config
data:
    # example vllm config
    is-prefill-worker: true
    data-parallel-size: 2
    enable-kv-routing: true
    max-model-length: 10240
    gpu-memory-utilization: 0.8
    enforce-eager: true
```

This loosely defines how to pass configs to the components.

Backend component will 
- read the env variable `DYNAMO_EXTRA_CONFIG` 
- read the config file during initialization
- update args based on the config file


## Alternative 1: Composable helm charts
We can leverage [Helmfile](https://github.com/helmfile/helmfile?tab=readme-ov-file#getting-started) to compose helm charts for different dynamo functionalities.

A single helmfile can be used to deploy the components below in a composable way. A reference would be the [wide-ep-lws example in llm-d](https://github.com/llm-d-incubation/llm-d-infra/tree/main/quickstart/examples/wide-ep-lws).

Helm charts:
- [Dynamo cloud platform](https://github.com/ai-dynamo/dynamo/tree/main/deploy/cloud/helm) is the base helm chart and deploys the operator for managing the life cycle of the graph deployment, grove integration, etc.
  - current state: independent [Dynamo cloud platform](https://github.com/ai-dynamo/dynamo/tree/main/deploy/cloud/helm) helm chart
- Dynamo Inference Gateway helm chart
  - current state: independent [Dynamo Inference Gateway helm chart](https://github.com/ai-dynamo/dynamo/blob/f7e468c7e8ff0d1426db987564e60572167e8464/deploy/inference-gateway/helm/dynamo-gaie/values.yaml#L27)
- Metrics
  - current state: we don't have a helm chart but we have a few YAML files with env variables [Metrics](https://github.com/ai-dynamo/dynamo/tree/main/deploy/metrics/k8s)
- Benchmark
  - current state: a few hard-coded jobs in the `benchmark` [folder](https://github.com/ai-dynamo/dynamo/tree/main/benchmarks/profiler/deploy)
- Model Express
  - current state: hard-coded YAML for single config in agg mode [model-express](https://github.com/ai-dynamo/modelexpress/pull/31/files)
- Fault injection/Test
  - Doesn't exist
- Troubleshooting
  - Doesn't exist


### Other alternative considerations
1. Use environment variables (current approach)
- We are reinventing template rendering with `envsubst` in a non-sustainable way
- This is not an ideal extensible quickstart for users

2. Use `kustomize` to render the template
- This is a sane way for customization and better than env vars + `envsubst`
- This can be derived from helm charts (use `helm template` to get the k8s YAML)
- Helmfile supports `kustomize` to render the template. [Reference](https://helmfile.readthedocs.io/en/latest/advanced-features/#deploy-kustomizations-with-helmfile)


## Phase 1: Quickly iterate on helm chart and publish for public usage/feedback

Based on the inputs in values.yaml, the helm chart renders appropriate k8s DynamoGraphDeployment CR/Services fragments.

A similar reference structure is already used in the [Dynamo Inference Gateway helm chart](https://github.com/ai-dynamo/dynamo/blob/f7e468c7e8ff0d1426db987564e60572167e8464/deploy/inference-gateway/helm/dynamo-gaie/values.yaml#L27).
```yaml


# This is the Dynamo namespace where the dynamo model is deployed
dynamoNamespace: "vllm-agg"

# This is the port on which the model is exposed
model:
  # This is the model name that will be used to route traffic to the dynamo model
  # for example, if the model name is Qwen/Qwen3-0.6B, then the modelShortName should be qwen
  identifier: "Qwen/Qwen3-0.6B"
  # This is the short name of the model that will be used to generate the resource names
  shortName: "qwen"
  # Criticality level for the inference model
  criticality: "Critical"

inferencePool:
  ...

# HTTPRoute configuration
httpRoute:
  enabled: true
  ...

extension:
  # the GAIE extension
  image: ...
```


## Phase 2: Stabilize and update operator

Incorporate the logic in the operator after the UX above is validated/finalized through quick iteration.
