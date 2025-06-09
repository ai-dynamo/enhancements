# An alternate approach to Dynamo Serve and a Dynamo Component

**Status**: Under Review

**Authors**: Ishan Dhanani, Alec Flowers

**Category**: Architecture

**Required Reviewers**: Mohammed Abdulwahhab, Biswa Panda Rajan, Neelay Shah, Maksim Khadkevich

**Review Date**: 06/09/2025

**Implementation PR / Tracking Issue**: WIP

# Summary

The current set of dynamo components that live under `examples/{llm, vllm_v1, vllm_v0, sglang, tensorrt_llm}` are unable to be used outside of the `dynamo serve` code path. This defeats the purpose of modular components and does not allow our users to run each component separately using just python. Additionally - all logic to run each component is hidden in the `sdk` under `serve_dynamo.py`. This single file contains all of the python bindings required to run our components. We need to provide a UX that demonstrates the power of our rust/python interoperability/bindings while maintaining the ease of use provided by `dynamo serve`.

# Motivation

Dynamo is meant to be a distributed, modular framework that allows users to build inference graphs for modern day inference strategies like disaggregated serving and KV aware routing. The CLI tool `dynamo serve` (which stems from the BentoML CLI) is used to launch multiple components as subprocesses controlled by a process watcher named `circusd`. Dynamo serve provides a very clean user experience allowing first time users to run a single commmand and interact with each inference strategy. However, due to the limitations of the BentoML component syntax - our components are now only usable via the `dynamo serve` tool with all of dynamo logic that makes our platform special hidden behind a single `serve_dynamo.py` file about 6 folder down.

1. Our framework has lost its modularity. A user cannot run each component using just python
2. The core of our framework has been lost to the BentoML-style path of running all logic for a component using a single python module
3. It is very difficult to write multinode instructions. We have to provide some `.link` file leverage a non-intuitive `--service-name` flag just to run a single component
4. We've found trouble with `mpi` and `circus` which causes pain when trying to run tensorrt-llm examples on our SLURM clusters for benchmarking
5. We are starting to have duplicated code under `/launch`. As someone who writes these components for our different inference frameworks, it's a lot more intuitive to use the bindings than it is to use the BentoML component syntax

As we start ramping up to a production version of dynamo, it is paramount that we get our UX nailed down. In this proposal we provide an alternative approach that allows us to maintain the developer experience of `dynamo serve` while allowing us to demonstrate the power of our framework

## Goals

- Each Dynamo component **MUST** be runnable using `python3 component.py <flags>`

- `dynamo serve` **MUST** be able to launch multiple component using `python3 -m component.py <flags>`

- We **SHOULD** have a common way of argument parsing for each component (similar to the `ServiceConfig`) that is used during a standalone component run and `dynamo serve`

- We **SHOULD NOT** hide our dynamo core bindings code (component, namespace, endpoint, etc) that is used in each component

- We **SHOULD** have a single source of worker (sglang, vllm, tensorrtllm) code in our repository that is compatible with `dynamo-run` and `dynamo serve`

- We **SHOULD** have a local -> K8s experience that mirrors the top OSS projects in the community today

### Non Goals

- We **SHOULD NOT** deprecate dynamo serve. The UX is very helpful

- This proposal will not address other abstractions like `depends()` and `link` that are currently present in the SDK

## Requirements

1. Each component (Frontend, Processor, PrefillWorker, DecodeWorker, etc) should have it's own `main` method in the file itself. This main method should leverage the python bindings (component creation, endpoint serving).
2. `dynamo serve` should run each component as a module instead of running `serve_dynamo` and pointing toward each class

List out any additional requirements in numbered subheadings.

### REQ 1 - `main` per component

Replace `serve_dynamo` with each component's main function. An example is shown below for the SGL Decode Worker. ote:
Note that we remove the `@service` decorator (a BentoML construct). The purpose of this decorator was to hold information on `resources` and top level dynamo info like `namespace`. Instead - we propose a `BaseDeployment` class that can hold this information. A user can extend this class to overwrite or use default values. This class can contain information/helpers in order to faciliate the K8s deployment. This also allows us to eliminate a couple different confusing aspects of the current code including `async_on_start` and `dynamo_endpoint`. This also allows us to move away from injecting the runtime into each worker via the `dynamo_context`. This concept is very confusing to an end user and there is no reason to have it.

Additionally - there has been discussion on repeated code between components. We can start using `BaseDecodeClass` and other variants to handle this logic.

```python
from dynamo.deploy import Deployment
from dynamo.runtime import {...}

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

### REQ 2 - FlexibleArgumentParser for Dynamo

Currently - we pass in arguments for each worker via a YAML file. This YAML file is combined with any CLI overrides, saved in an environment variable, and then exported in each workers process. An end user has no idea how this works unless they dive into the `ServiceConfig` class. Instead, we propose a `DynFlexibleArgumentParser`. This works similarly to the current `ServiceConfig` but is expanded to also be used if a user is running `python3 component.py <flags>`.

# SDK and Runtime relationship

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

Note that this includes new endpoints that abstract away the SDK (and require `dynamo serve` to run). It is an open question as to whether we should be using this approach or not.
