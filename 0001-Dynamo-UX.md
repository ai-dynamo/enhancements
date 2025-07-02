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
# serve 
dynamo serve <model-name>
dynamo serve --mode disagg --engine=vllm <model-name>
dynamo serve --mode disagg --engine=vllm -f ./config.yaml <model-name>

# deploy golden path
dynamo deploy <model-name>
dynamo deploy --mode disagg --engine=vllm <model-name>
dynamo deploy --mode disagg --engine=vllm -f ./my_custom_config.yaml <model-name>
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
### REQ 2: Dynamo users MUST be able to explicitly specify exact configuration
### REQ 3: Dynamo users MUST be able to deploy a dynamo graph using a simplified config

## Scenarios

Persona: Entrprise customer (K8s  Savvy)
Persona: Compoent Developer

# Proposal

## Launching a component

`dyanmo serve` command will launch individual component (single process) 

Example:

Launch Frontend (+Processor+Router)
```bash
dynamo serve in=http out=dyn -f config.yaml
```
Launch vllm worker
```bash
dynamo serve in=dyn out=vllm -f config.yam
```

Current UX
```bash
dynamo serve --system-app-port 5000 --enable-system-app --use-default-health-checks \
  --service-name VllmWorker graphs.agg:Frontend \
  --VllmWorker.ServiceArgs.dynamo.namespace=dynamo
```

## Launching a graph


### Alternative 1: Separate deployment and component configs

```bash
# creates deploymenet manifests for the target (default =k8s) 
dynmao deploy -c ./config.yaml -f ./deployment.yaml --out_dir=k8s_deployment

dynmao deploy --target slurm -c ./config.yaml -f ./deployment.yaml --out_dir=slum_deployment
```

1. config.yaml

This will map to [current config yaml](https://github.com/ai-dynamo/dynamo/blob/main/examples/vllm_v1/configs/disagg.yaml)
`dynamo serve -c ./config.yaml` to run a service


### Alternative 2: Single config file with embedded component configs

`deployment-config.yaml`
```yaml
version: 1.0
name: dynamo-graph
components:
  - name: http_ingress
    image: "<pre-built-image>"
    cmd: ["dynamo", "serve"]    # default cmd, current dynamo-run
    run_config:
      input: http
      output: dyn
      parameters:               # these parameters are passed to component
        port: 8080
    replicas: 5
    resources:
      cpu: 500m
      memory: 2Gi
  - name: vllm_worker
    cmd: ["dynamo", "serve"]
    image: ...
    run_config:
      input: "dyn://llama3-8b.backend.generate"
      output: vllm
      parameters:
        model_path: "meta-llama/Meta-Llama-3-8B-Instruct"
        tensor_parallel_size: 2
        context_length: 8192
        base_gpu_id: 0
        extra_engine_args: "vllm_config.json"
    replicas: 2
    resources:
      gpu: 2
      memory: 24Gi
    environment:
      CUDA_VISIBLE_DEVICES: "0,1"
      HF_TOKEN: "${{ my_secret_name.HF_TOKEN }}"
    secrets:
      - my_secret_name
```
