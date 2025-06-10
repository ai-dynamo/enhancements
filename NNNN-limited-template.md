# Consolidating dynamo-serve and dynamo-run component API

**Status**: Under Review

**Authors**: Ishan Dhanani, Alec Flowers

**Category**: Architecture

**Required Reviewers**: Itay Neeman, Kyle Kranen, Mohammed Abdulwahhab, Maksim Khadkevich, Biswa Panda Rajan

**Review Date**: 06/09/2025

**Implementation PR / Tracking Issue**: WIP

# Background: Dynamo-serve vs Dynamo-run

Right now we have 2 different ways of running components that go through different code paths:

## dynamo-run

- **Purpose**: CLI tool for running OAI frontend/processor and python engines
- **Usage**: `dynamo run in=http out=vllm ~/models/Llama-3.2-3B`
- **Architecture**: Direct component execution - each engine runs as a subprocess
- **Control**: Explicit configuration via CLI flags

## dynamo-serve

- **Purpose**: Orchestrates inference graphs composed of multiple interdependent services
- **Usage**: `dynamo serve graphs.agg:Frontend -f ./configs/agg.yaml`
- **Architecture**: Service graph execution using circus process manager + BentoML decorators
- **Control**: High-level via YAML configs, dependency/env var injection and `depends()`, `.link()`

## Current Hybrid Approach

The frontend actually uses `dynamo-run` internally to run the OpenAI-compatible HTTP server and Rust-based processor, while `dynamo-serve` manages the overall service orchestration.

## The Problem

Dynamo is meant to be a distributed, modular framework for building inference graphs with modern strategies like disaggregated serving and KV-aware routing. While `dynamo serve` provides a clean UX via `circusd` process management, we've created a problematic split:

**Historical Context**: Pre-GTC, our components demonstrated rust↔python interoperability and were runnable with `python3 component.py <flags>` - they were pythonic and easy to hack on.

**Current Issues**:

1. Duplicated dynamo-run examples vs dynamo-serve examples with no code sharing
2. All rust bindings logic is buried in the SDK under `serve_dynamo.py`
3. Current components in `examples/{llm, vllm_v1, vllm_v0, sglang, tensorrt_llm}` cannot be run outside the `dynamo serve` code path
4. We no longer demonstrate the power of our rust/python bindings in an accessible way

# Motivation

- Users must use `dynamo-serve` for all components - they can no longer run any example code with `python3 component.py <flags>`. This breaks the pythonic, hackable experience that made Dynamo accessible.

- Decorators hide critical logic, making debugging nearly impossible. Runtime injection of `CUDA_VISIBLE_DEVICES`, namespace overrides in K8s, and other "magic" configurations surprise users with no clear error messages.

- Large model deployment requires unintuitive flags and `.link` files.

- We've hit significant issues with `mpi` and `circus` when running tensorrt-llm examples on SLURM clusters for benchmarking.

- We maintain two separate sets of examples - [dynamo serve](https://github.com/ai-dynamo/dynamo/blob/main/examples/sglang/components/worker.py) and [dynamo run](https://github.com/ai-dynamo/dynamo/blob/main/launch/dynamo-run/src/subprocess/sglang_inc.py) - with duplicated logic and no code sharing.

- We've hidden the rust↔python interoperability that differentiates Dynamo. Users can't see or interact with our core bindings (component, namespace, endpoint).

As we ramp up to production, fixing this UX split is critical. This proposal provides a path to maintain `dynamo serve`'s developer experience while staying true to our rust core and making components standalone + runnable.

## Goals

1. Each Dynamo component **MUST** be runnable using `python3 component.py <flags>`

2. `dynamo serve` **MUST** launch components using `python3 -m component.py <flags>` internally

3. We **MUST** expose dynamo core bindings (component, namespace, endpoint) directly in each component - no more hiding behind decorators

4. We **MUST** unify argument parsing across standalone and serve modes (similar to current `ServiceConfig` but shared)

5. We **MUST** maintain single-source components compatible with both `dynamo-run` and `dynamo serve`

6. We **MUST** provide transparent configuration - no runtime injection or environment variable magic

7. We **MUST** deliver clear error messages with actionable fix instructions instead of silent overrides

8. We **SHOULD** enable seamless local → K8s deployment patterns matching industry standards (AIBrix, VLLM production stack, llmd)

### Non Goals

1. We **SHOULD NOT** deprecate `dynamo serve` - the orchestration UX remains valuable

2. This proposal will not address SDK layer abstractions like `depends()` and `link` that are currently present in the SDK

## Proposal

### `main` per component

- Replace `serve_dynamo` with each component's main function. An example is shown below for the SGL Decode Worker.
- Remove `@service` decorator (BentoML construct) that held `resources` and dynamo info like `namespace`
- Add `BaseDeployment` class to store:
  - Resources configuration
  - Namespace info
  - K8s deployment helper functions
  - Default values that can be overridden
- Eliminate confusing code:
  - Remove `async_on_start`
  - Remove `dynamo_endpoint`
  - Remove runtime injection via `dynamo_context`
- We can start using abstract classes for our request handlers to address the issue to repeated code for things like `Router` and `Frontend`

```python
from dynamo.deploy import BaseDeployment # adding this class turns this into a valid deployment
from dynamo.sglang.utils import parse_sglang_args

class Deployment(BaseDeployment):
    namespace = "dynamo"
    name = "sgldecode"
    resources = {"gpu": 1}

class DecodeHandler(BaseDecodeClass):
    def __init__(self, engine_args: ServerArgs):
        self.engine_args = engine_args
        self.engine = sgl.Engine(server_args=self.engine_args)
        logger.info("Decode worker initialized")

    async def generate(self, req: DisaggPreprocessedRequest):
        g = await self.engine.async_generate(
            input_ids=req.request.token_ids,
            sampling_params=req.sampling_params,
            stream=True,
            bootstrap_host=req.bootstrap_host,
            bootstrap_port=req.bootstrap_port,
            bootstrap_room=req.bootstrap_room,
        )

        async for result in g:
            yield result

if __name__ == "__main__":
    import asyncio
    import uvloop
    from dynamo.runtime import DistributedRuntime, dynamo_worker

    engine_args = parse_sglang_args()

    @dynamo_worker()
    async def worker(runtime: DistributedRuntime, engine_args):
        worker = SGLangDecodeWorker(engine_args)
        deploy = Deployment()

        component = runtime.namespace(deploy.namespace).component(deploy.name)
        await component.create_service()

        # serve endpoint
        endpoint = component.endpoint("generate")
        await endpoint.serve_endpoint(worker.generate)

    uvloop.install()
    asyncio.run(worker(engine_args))
```

### FlexibleArgumentParser for Dynamo

Currently - we pass in arguments for each worker via a YAML file. This YAML file is combined with any CLI overrides, saved in an environment variable, and then exported in each workers process. An end user has no idea how this works unless they dive into the `ServiceConfig` class. Instead, we propose a `DynamoFlexibleArgumentParser`. This works similarly to the current `ServiceConfig` but is expanded to also be used if a user is running `python3 component.py <flags>`.

# Appendix

## Unified README/examples

Work in progress. Would love feedback!

```bash
examples/
├── README.md
├── common/
│ ├── frontend.py
│ └── base_classes.py
├── sglang/
│ ├── sglang_engine.py
│ └── utils.py
├── vllm/
│ ├── vllm_engine.py
│ └── utils.py
└── trtllm/
│ ├── trtllm_engine.py
│ └── utils.py
```

Each engine's `utils.py` would contain things like argument parsing/validation and any other helper functions. The `common/base_classes.py` could contain abstract classes for the `BaseDecodeWorker` or the `BaseWorker` class.
