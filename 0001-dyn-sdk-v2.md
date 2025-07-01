# Dynamo SDK v2

**Status**: Draft

**Authors**: [Name/Team] 

**Category**: Architecture

**Replaces**: [Link of previous proposal if applicable] 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary


```yaml
version: 1.0
name: my-graph
namespace: ns1
components:
  - name: frontend
    py_class: a.b.c:Frontend
    dependency:
       backend1: dynamo://v1/ns1/backend_1
       backend2: dynamo://v1/ns1/backend_2
    parameters:
       a: b 
    resources:
       cpu: 200m
       gpu: 1
    replicas: 4
    environment:
      CUDA_VISIBLE_DEVICES: "4,5"
      SAMPLE_CONFIG: A1
      DB_URI: "${{ secrets.DB_URI }}"
    secrets:
      - DB_URI
  - name: backend_1
    cmd: ["dynamo", "serve"]
    arg: ["a.b.c:Backend"]
    instances: 1
    dependency:
       backend: dynamo://v1/ns1/backend_2
  - name: backend_2
    # alternative syntax - dynamo serve
    cmd: ["dynamo", "serve", "..."]    # python component
    cmd: ["dynamo", "run", "..."]      # rust component
    replicas: 2
  - name: backend3
    cmd: ["/my/rust_backend3"]
    dependency:
       backend: dynamo://v1/ns1/backend3
  # New: dynamo run components
  - name: http_ingress
    cmd: ["dynamo", "serve"] # current dynamo-run
    run_config:
      input: http
      output: dyn
      port: 8080
      model_name: "llama3-8b"
    replicas: 1
    resources:
      cpu: 500m
      memory: 2Gi
  - name: vllm_worker
    cmd: ["dynamo", "serve"] # current dynamo-run 
    run_config:
      input: "dyn://llama3-8b.backend.generate"
      output: vllm
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
  - name: sglang_worker
    cmd: ["dynamo", "serve"] # current dynamo-run
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
  - name: llamacpp_worker
    cmd: ["dynamo", "serve"] # current dynamo-run
    run_config:
      input: text
      output: llamacpp
      model_path: "~/llms/Llama-3.2-3B-Instruct-Q4_K_M.gguf"
      model_config: "meta-llama/Llama-3.2-3B-Instruct"
      context_length: 4096
    replicas: 1
    resources:
      cpu: 2000m
      memory: 8Gi
  - name: batch_processor
    cmd: ["dynamo", "serve"] # current dynamo-run
    run_config:
      input: "batch:/data/prompts.jsonl"
      output: mistralrs
      model_path: "Qwen/Qwen3-4B"
      verbosity: 2  # -vv flag
    replicas: 1
    resources:
      gpu: 1
      memory: 16Gi
  # Multi-node distributed example
  - name: trtllm_leader
    cmd: ["dynamo", "serve"] # current dynamo-run
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


# Motivation

**\[Required\]**

Describe the problem that needs to be addressed with enough detail for
someone familiar with the project to understand. Generally one to two
short paragraphs. Additional details can be placed in the background
section as needed. Cover **what** the issue is and **why** it needs to
be addressed. Link to github issues if relevant.

## Goals

**\[Optional \- if not applicable omit\]**

List out any additional goals in bullet points. Goals may be aspirational / difficult to measure but guide the proposal. 

* Goal

* Goal

* Goal

### Non Goals

**\[Optional \- if not applicable omit\]**

List out any items which are out of scope / specifically not required in bullet points. Indicates the scope of the proposal and issue being resolved.

## Requirements

**\[Optional \- if not applicable omit\]**

List out any additional requirements in numbered subheadings.

**\<numbered subheadings\>**

### REQ \<\#\> \<Title\>

Describe the requirement in as much detail as necessary for others to understand it and how it applies to the DEP. Keep in mind that requirements should be measurable and will be used to determine if a DEP has been successfully implemented or not.

Requirement names should be prefixed using a monotonically increasing number such as “REQ 1 \<Title\>” followed by “REQ 2 \<Title\>” and so on. Use title casing when naming requirements. Requirement names should be as descriptive as possible while remaining as terse as possible.

Use all-caps, bolded terms like **MUST** and **SHOULD** when describing each requirement. See [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119) for additional information.


# Proposal

**\[Required\]**

Describe the high level design / proposal. Use sub sections as needed, but start with an overview and then dig into the details. Try to provide images and diagrams to facilitate understanding.

# Alternate Solutions

**\[Required, if not applicable write N/A\]**

List out solutions that were considered but ultimately rejected. Consider free form \- but a possible format shown below.

## Alt \<\#\> \<Title\>

**Pros:**

\<bulleted list or pros describing the positive aspects of this solution\>

**Cons:**

\<bulleted list or pros describing the negative aspects of this solution\>

**Reason Rejected:**

\<bulleted list or pros describing why this option was not used\>

**Notes:**

\<optional: additional comments about this solution\>

