# Dynamo UX v2

**Status**: Draft

**Authors**: [biswa](https://github.com/biswapanda)

**Category**: Architecture

**Required Reviewers**: Itay, Neelay, Ishan, Alec, Mohammed, Maksim

**Review Date**: 07/02/2025

**Related Docs**:
-  [Dynamo SDK Abstractions design and Multi-Target Deployment](https://docs.google.com/document/d/1UNSD_MUOYa1cbGwHp0Wn53wdO0Ir55KR7rfDUZYvNto/edit?tab=t.0)
- [merge dynamo serve and run](https://github.com/ai-dynamo/enhancements/blob/grahamk/serve-run-merge/deps/NNNN-serve-run-merge.md)

# Summary

1. current `dynamo-run` converges into `dynamo serve` [related DEP](https://github.com/ai-dynamo/enhancements/blob/grahamk/serve-run-merge/deps/NNNN-serve-run-merge.md)

2. separate responsibilities
- `dynamo serve` will launch a single component only
- `dynamo deploy` will launch multiple components (graph)

3. Consistent UX

```bash
# serve frontend
dynamo serve in=http out=dyn
# serve a backend
dynamo serve --engine vllm <model-name>
dynamo serve --task prefill --engine=trtllm <model-name>
dynamo serve --engine=vllm -f ./config.yaml <model-name>

# deploy golden path
dynamo deploy <model-name>

# deploy a model in aggregated mode
dynamo deploy --mode agg --engine vllm <model-name>

# explicit config file
dynamo deploy --mode disagg --engine trtllm -f ./my_custom_config.yaml <model-name>
```

# Motivation

## Tight coupling between component's implementation and deployment
Dynamo user persona range from expert k8s to power dynamo component developers.
Both dont't need handholding and full control.

```python
@service(
    dynamo={
        "namespace": "dynamo-demo",
    },
    resources={"gpu": 1, "cpu": "10", "memory": "20Gi"},
    workers=1,
)
class PrefillWorker:
```

## Too many levels of configuration 
Configurations are managead in SDK decorators, CLI args, env variables and config files. 
- Dynamo components authors are confused how to config and launch components
- K8s savvy end-users are confused where and how to configure a dynamo graph in k8s

## Implicit resource allocation
Users are unable to specify gpu resources explicitly

## Design Principles

### SOC: Separation of concerns
1. Decouple component author API from k8s deployment
2. Separate component and graph launch verbs (dynamo serve and dynamo deploy)


### UX: Simple is better than complex.
- Consistent UX across `dynamo serve` and `dynamo deploy` commands
- Enable dynamo developers to completely control how to spin up a component 

### Explicit is better than implicit
- No handholding needed, customers/users are domain experts
- Allow users to fully and explicitly specify all configurations (gpu resources, parameters etc.)


## Requirements

### REQ 1: Dynamo serve SHOULD not interleave deployment logic
### REQ 2: Dynamo users MUST be able to deploy a dynamo graph using a simplified explicitly specified config

## Scenarios

Persona: Entrprise customer
- experts in managing K8s 
- need full control over customizing a graph deployment

Persona: Compoent Developer
- need full control over launching a component
- need consistent deployment in k8s through ci/cd  

# Proposal

## Launching a component

`dyanmo serve` command will launch individual component (single process) 

Dedidcated DEP [merge dynamo serve and run](https://github.com/ai-dynamo/enhancements/blob/grahamk/serve-run-merge/deps/NNNN-serve-run-merge.md) to address the same.

### Alt 1: Preserve current dynamo run experience

Launch Frontend (+Processor+Router)
```bash
dynamo serve in=http out=dyn
```
Launch vllm worker
```bash
dynamo serve in=dyn out=vllm -f config.yaml
```

### Alt 2: similar experience across serve and deploy

```bash
dynamo serve --engine vllm <model-name>
dynamo serve --task prefill --engine=trtllm <model-name>
dynamo serve --engine=vllm -f ./config.yaml <model-name>
```


### Note
Current UX
```bash
dynamo serve --system-app-port 5000 --enable-system-app --use-default-health-checks \
  --service-name VllmWorker graphs.agg:Frontend \
  --VllmWorker.ServiceArgs.dynamo.namespace=dynamo
```

## Launching a graph

Dynamo deploy command will generate target specific manifests. 
- input: config.yaml (component run config), deployment.yaml (deployment spec)
- output: target specific manifests

### Alternative 1: Separate deployment and component configs

```bash
# creates deploymenet manifests for the target (default =k8s) 
dynmao deploy -c ./config.yaml -f ./deployment.yaml --out_dir=k8s_deployment

dynmao deploy --target slurm -c ./config.yaml -f ./deployment.yaml --out_dir=slum_deployment
```

### `config.yaml` 
maps to [current config yaml](https://github.com/ai-dynamo/dynamo/blob/main/examples/vllm_v1/configs/disagg.yaml) and used with `dynamo serve ...  -c ./config.yaml` to run a service.

`deployment.yaml`
```yaml
version: 0.1
name: dynamo-graph
components:
  - name: http_ingress        # component name matches with name in config.yaml
    image: "<pre-built-image>"
    cmd: ["dynamo", "serve"]    
    # default command is `dynamo serve`
    # Alternatively, user can specify any command -
    # cmd: ["python3", "-m", "a.b.MyComponent"]
    # cmd: ["rust-binary"]
    run_config:
      # raw argv style positional args
      args:                     # command arguments 
        - input=http
        - output=dyn
    replicas: 5
    resources:
      cpu: 500m
      memory: 2Gi
  - name: vllm_worker
    cmd: ["dynamo", "serve"]
    image: "<pre-built-image>"
    run_config:
      args:
        input: "dyn://llama3-8b.backend.generate"
        output: vllm
    replicas: 2
    resources:
      gpu: 2
      cpu: 10
      memory: 24Gi
    environment:
      DISABLE_FOO: 1
    # these secrets will be injected as env variables
    # in k8s, these are secret refs
    secret_env:
      - my_k8s_secret_name
```

### Alternative 2: Single config file

```bash
dynamo deploy -f ./config.yaml --out_dir=k8s_deployment
```


`config.yaml`
```yaml
version: 0.1
name: dynamo-graph

components:
  - name: http_ingress
    options:
      port: http
  - name: vllm_worker
    options:
      model_path: "meta-llama/Meta-Llama-3-8B-Instruct"
      tensor_parallel_size: 2
      context_length: 8192
      base_gpu_id: 0

# deployment section
deployments:
  - name: http_ingress
    image: "<pre-built-image>"
    cmd: ["dynamo", "serve"]    
    # default command is `dynamo serve`
    # Alternatively, user can specify any command -
    # cmd: ["python3", "-m", "a.b.MyComponent"]
    # cmd: ["rust-binary"]
    run_config:
      # raw argv style positional args
      args:                     # command arguments 
        - input=http
        - output=dyn
      options:   # options are rendered in the format --key value
        port: 8000
    replicas: 5
    resources:
      cpu: 500m
      memory: 2Gi
  - name: vllm_worker
    cmd: ["dynamo", "serve"]
    image: "<pre-built-image>"
    run_config:
      args:
        input: "dyn://llama3-8b.backend.generate"
        output: vllm

    replicas: 2
    resources:
      gpu: 2
      cpu: 10
      memory: 24Gi
    environment:
      DISABLE_FOO: 1
    # these secrets will be injected as env variables
    # in k8s, these are secret refs
    secret_env:
      - my_k8s_secret_name
```


### Alternative 3: Single config file with embedded component configs

`deployment.yaml`
```yaml
version: 0.1
name: dynamo-graph
components:
  - name: http_ingress
    image: "<pre-built-image>"
    cmd: ["dynamo", "serve"]    
    # default command is `dynamo serve`
    # Alternatively, user can specify any command -
    # cmd: ["python3", "-m", "a.b.MyComponent"]
    # cmd: ["rust-binary"]
    run_config:
      # raw argv style positional args
      args:                     # command arguments 
        - input=http
        - output=dyn
      options:   # options are rendered in the format --key value
        port: 8000
    replicas: 5
    resources:
      cpu: 500m
      memory: 2Gi
  - name: vllm_worker
    cmd: ["dynamo", "serve"]
    image: "<pre-built-image>"
    run_config:
      args:
        input: "dyn://llama3-8b.backend.generate"
        output: vllm
      options:
        model_path: "meta-llama/Meta-Llama-3-8B-Instruct"
        tensor_parallel_size: 2
        context_length: 8192
        base_gpu_id: 0
    replicas: 2
    resources:
      gpu: 2
      cpu: 10
      memory: 24Gi
    environment:
      DISABLE_FOO: 1
    # these secrets will be injected as env variables
    # in k8s, these are secret refs
    secret_env:
      - my_k8s_secret_name
```


### Building base image

Publish engine specific image with pre-built components for example current form of `examples/vllm/*` is available for python import. 