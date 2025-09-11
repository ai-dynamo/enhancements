# Benchmarking Harness

Author: @biswapanda

# Problem statement:
In current state, benchmarking has a few problems:

1. UX: Experimentation, debugging, and iteration is hard. 
use case: As a user, I want to easily experiment with different configs, and get results quickly and compare them.

2. Reproducibility is hard: we don't store the input configs and results.
use case: As a user, I want to be able to reproduce my experiments and share them with others.

3. benchmarking steps are tightly coupled. If a sinlge step/benchmark config fails the entire process is aborted/retried.   

4. port-forwarding and benchmarking has non-deterministic latency characteristics. 

## Proposed plan:

1. decouple all steps and then compose them together: prep a model, deploy k8s cr, benchmark, collect data 

2. capture configs for the experiment: deploy (config or a reference to deployment), benchmark, model etc

3. we'd run benchmarks inside k8s cluster in k8s native approach.


## Steps:
Following steps are executed by the harness:

Note: These steps are reusable across different tests (LLM benchmarking, Accuracy testing, Functional testing etc)

Since these steps are reusable across different tests, we can swap the container used for each step.

1. Initialize experiment

    a. (Optional) deploy model

    b. wait for the model to be ready

2. Run Benchmarking test using configs and benchmark container (genai-perf, ai perf or 3rd party tool)

    a. Prepare configs (matrix of params: isl/osl, concurrency, etc) 
        pass as a config file to the harness container

    b. Run test for each config

3. Teardown

    a. (Optional) Collect artifacts - push files to upstream storage (s3/minio)

    b. Collect output results: 
        Push benchmark metrics to a data storage layer (s3/minio/database) using a cli tool

4. Analytics:
    a. Generate charts, graphs, and tables from the benchmark metrics


LLM benchmarking container can use the `python3 -m benchmarks.utils.*` utils to generate the config and run the benchmark.

## Config

Benchmarking config file:
```yaml
name: "blueprint-name"
model: 
    name: "RedHat/Llama-3.3-70B-Instruct"
    path: "/path/to/model"
concurrency: [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]
endpoint: "/v1/chat/completions"
endpoint_type: "chat"
benchmark:
    isl_osl
    - [8192, 1024]
    - [1024, 1024]
    - [1024, 8192]
```

## Alternatives:

### Alternative 1: Benchmarking as a first class citizen in dynamo

```
kind: DynamoBenchmark
metadata:
  name: vllm-agg-benchmark
spec:
  model:
     modelRef: llama-3-70b-instruct-v1
  config: 
      model: "RedHat/Llama-3.3-70B-Instruct"
      path: "/path/to/model"
      concurrency: [1, 2, 4, 8, 16, 32, 64, 128, 256, 512, 1024]
      endpoint: "/v1/chat/completions"
      endpoint_type: "chat"
      benchmark:
          isl_osl
          - [8192, 1024]
          - [1024, 1024]
          - [1024, 8192]
```

### Alternative 2: Benchmarking helm chart + workflow manager

Simpler to manage and deploy.
Reuse Argo workflows for the workflow manager to orchestrate deps and workflow.