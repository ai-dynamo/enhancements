# Dynamo SDK v2 and IR design

**Status**: Draft

**Authors**: [biswa](https://github.com/biswapanda)

**Category**: Architecture

**Replaces**: 

**Replaced By**: 

**Sponsor**: 

**Required Reviewers**: Neelay, Ishan, Alec, Mohammed, Maksim

**Review Date**: [TBD]

**Related Docs**:
-  [Dynamo SDK Abstractions design and Multi-Target Deployment](https://docs.google.com/document/d/1UNSD_MUOYa1cbGwHp0Wn53wdO0Ir55KR7rfDUZYvNto/edit?tab=t.0)

# Summary

1. current `dynamo-run` will converge into `dynamo serve`

2. separate responsibilities
- `dynamo serve` will launch a single component only
- `dynamo deploy` will launch multiple components (graph)


# Motivation

Issues
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

## 2: Too many levels of configuration 
Configurations are managead in SDK decorators, CLI args, env variables and config files. 
- Dynamo components authors are confused how to config and launch components
- K8s savvy end-users are confused where and how to configure a dynamo graph in k8s

## 3: Implicit resource allocation
Users are unable to specify gpu resources explicitly

## Design Principles

### SOC: Separation of concerns
1. Decouple component author API from k8s deployment related concerns
2. Separate component and graph launch verbs (dynamo serve and dynamo deploy)


### Dev-Ex: Simple is better than complex.
1. Enable dynamo developers to completely control how to spin up a component   

### Explicit is better than implicit
Allow users to fully and explicitly specify all configurations (gpu resources, parameters etc.)


## Requirements

### REQ 1: Dynamo serve SHOULD not interleave deployment logic
### REQ 2: Dynamo users MUST be able to explicitly specify exact configuration
### REQ 3: Dynamo users MUST be able to deploy a dynamo graph using a simplified config


# Proposal

## Graph Deployment IR

Dynamo deployment IR is where user can specify deployment spec in a deployment target agnostic 


```bash
# creates deploymenet manifests for the target (default=k8s) 
dynmao deploy --target k8s -f ./my-graph-config.yaml --out_dir=k8s_deployment
dynmao deploy --target slurm -f ./my-graph-config.yaml --out_dir=slum_deployment
```

`my-graph-config.yaml`
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
    parameters:
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
      HF_TOKEN: "${{ secrets.HF_TOKEN }}"
    secrets:
      - my_secret_name
  - name: sglang_worker
    image: ...
    cmd: ["dynamo", "serve"]
    run_config:
      input: "dyn://qwen3-32b.backend.generate" 
      output: sglang
      model_path: "/data/models/Qwen/Qwen3-32B"
      tensor_parallel_size: 4
      router_mode: "kv"
      num_nodes: 2
      node_rank: 0
      leader_addr: "127.0.0.1:9876"
    replicas: 1
    resources:
      gpu: 4
      memory: 64Gi
  - name: batch_processor
    cmd: ["dynamo", "serve"]
    run_config:
      input: "batch:/data/prompts.jsonl"
      output: mistralrs
      model_path: "Qwen/Qwen3-4B"
      verbosity: 2  # -vv flag
    replicas: 1
    resources:
      memory: 16Gi
  # Multi-node distributed example
  - name: trtllm_leader
    cmd: ["dynamo", "serve"]
    run_config:
      input: "dyn://deepseek-70b.backend.generate"
      output: trtllm
      model_path: "deepseek-ai/DeepSeek-R1-Distill-Llama-70B"
      tensor_parallel_size: 16
      num_nodes: 2
      node_rank: 0
      leader_addr: "10.217.98.122:5000"
      extra_engine_args: "trtllm_config.yaml"
    replicas: 1
    resources:
      gpu: 8
      memory: 80Gi
    node_selector:
      role: leader
```
