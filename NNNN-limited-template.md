# Consolidating dynamo-serve and dynamo-run component API

**Status**: Under Review

**Authors**: Ishan Dhanani, Alec Flowers

**Category**: Architecture

**Required Reviewers**: Mohammed Abdulwahhab, Biswa Panda Rajan, Neelay Shah, Maksim Khadkevich

**Review Date**: 06/09/2025

**Implementation PR / Tracking Issue**: WIP

# Summary

Dynamo is meant to be a distributed, modular framework that allows users to build inference graphs for modern day inference strategies like disaggregated serving and KV aware routing. The CLI tool `dynamo serve` is used to launch multiple components as subprocesses controlled by a process watcher named `circusd`. Dynamo serve provides a very clean user experience allowing first time users to run a single commmand and interact with each inference strategy.

Pre-GTC we had a set of components that were defined by the dynamo core bindings. An example can be found [here](https://github.com/dynamo-ai/dynamo/blob/main/examples/sglang/decode_worker.py) and [here](https://github.com/ai-dynamo/dynamo/blob/main/launch/dynamo-run/src/subprocess/sglang_inc.py). These components were runnable using python3 and demonstrated our the rust<>python interoperability. They were pythonic and easy to hack on.

The current set of dynamo components that live under `examples/{llm, vllm_v1, vllm_v0, sglang, tensorrt_llm}` are unable to be used outside of the `dynamo serve` code path (i.e cannot be run using `python3 component.py <flags>`). Additionally - all logic to run each component is hidden in the `sdk` under `serve_dynamo.py`. We need to provide a UX that demonstrates the power of our rust/python interoperability/bindings while maintaining the ease of use provided by `dynamo serve`.

# Motivation

1. A user must use dynamo-serve to run any/all components. They cannot use `python3` to run any of our example code anymore.
2. Difficult to run large models in a multinode setting due to unintiutive flags and `.link` files
3. We've found trouble with `mpi` and `circus` which causes pain when trying to run tensorrt-llm examples on our SLURM clusters for benchmarking
4. We have 2 sets of examples ([dynamo serve](https://github.com/ai-dynamo/dynamo/blob/main/examples/sglang/components/worker.py) and [dynamo run](https://github.com/ai-dynamo/dynamo/blob/main/launch/dynamo-run/src/subprocess/sglang_inc.py)). Difficult to maintain both
5. Decorators hide much of the logic which makes it difficult to debug
6. Things are injected at runtime that are not intuitive to an end user. An example is `CUDA_VISIBLE_DEVICES` override and namespace override in K8s

As we start ramping up to a production version of dynamo, it is paramount that we get our UX nailed down. In this proposal we provide an alternative approach that allows us to maintain the developer experience of `dynamo serve` while allowing us to stay true to our rust core.

## Goals

1. Each Dynamo component **MUST** be runnable using `python3 component.py <flags>`

2. `dynamo serve` **MUST** launch multiple component using `python3 -m component.py <flags>`

3. We **SHOULD** have a common way of argument parsing for each component (similar to the `ServiceConfig`) that is used during a standalone component run and `dynamo serve`

4. We **SHOULD NOT** hide our dynamo core bindings code (component, namespace, endpoint, etc) that is used in each component

5. We **SHOULD** have a single source of worker (sglang, vllm, tensorrtllm) code in our repository that is compatible with `dynamo-run` and `dynamo serve`

6. We **SHOULD** have a local -> K8s experience that mirrors the top OSS projects in the community today

7. We **SHOULD** not override anything a user sets. Instead we should have a clear error API and instructions on how to fix

### Non Goals

1. We **SHOULD NOT** deprecate dynamo serve. The UX is very helpful

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

## SDK abstractions

Over the last few months - there have been efforts on the SDK side to create a set of abstract classes and abstract endpoints that can be used to create a more flexible inference graph. An example of this graph is below

```python
class WorkerInterface(AbstractService):
    """Interface for LLM workers."""

    @abstract_endpoint  # enforces that the service implements the method, but also that it is properly decorated
    async def generate(self, request: ChatRequest):
        pass


class RouterInterface(AbstractService):
    """Interface for request routers."""

    @abstract_endpoint
    async def generate(self, request: ChatRequest):
        pass


@service(
    dynamo={"namespace": "llm-hello-world"},
    image=DYNAMO_IMAGE,
)
class VllmWorker(WorkerInterface):
    @endpoint()
    async def generate(self, request: ChatRequest):
        # Convert to Spongebob case (randomly capitalize letters)
        for token in request.text.split():
            spongebob_token = "".join(
                c.upper() if random.random() < 0.5 else c.lower() for c in token
            )
            yield spongebob_token
```

We believe this can be met by the proposal above using a set of abstract classes for request handlers (to avoid repeated code) and a `BaseDeployment` class to hold the resources and namespace info to faciliate the K8s deployment.
