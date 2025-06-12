# Summary

Introduce a strongly-typed, class-based component architecture for Dynamo SDK v2 that leverages Pydantic models. This proposal consolidates the current fragmented component so we can capture deployment metadata at build-time, eliminates boilerplate code.


# Guiding Principles

Definiing a `component`
- it hosts strongly typed endpoints at a discoverable dynamo address 
- it can be launched with `python3 my_component.py --args`
- has minimum resource requirements to run it locally or in kuberenetes
- can contain multiple instances (replicas)
- a scalable stateful (chained single-in/many-out async generators) unit of distributed compute


# Motivation
- address issues with accessing low level runtime/operational aspects
- eliminate dynamo_context
- dynamo serve command launches dynamo graph based on a user provided resource config


# Why class based components/endpoints ?
"Programming to an interface, not an implementation" is core principle in software design that promotes flexibility, maintainability, and testability.

- decoupling components at interface level
- each component has a strongly typed interface (defined by the async generator endpoints)
- we can swap a component implementation without breaking integration points.
- CPU based mocks for CI
- rollout new version of a single component without regression
- we are seeing lot of code/component duplication across examples (llm, vllm_v0, trtllm, sglang). If we use common interface & concrete implementation

from current files use same interface 
`async def generate(self, request: PreprocessedRequest):`

- `examples/vllm_v0/components/worker.py`
- `examples/sglang/components/worker.py`
- `examples/vllm_v1/components/simple_load_balancer.py`
```python
from utils.protocol import PreprocessedRequest

@service(
    dynamo={"namespace": "dynamo"},
    resources={"gpu": 1, "cpu": "10", "memory": "20Gi"},
    workers=1,
)
class VllmWorker:

    @async_on_start
    async def async_init(self):

    @endpoint()
    async def generate(self, request: PreprocessedRequest):
```

# Class based component

Example:

```python
import asyncio
import uvloop

from dynamo.core import Resource, Client, Instance, component

from my_graph import AnotherComponent

# Note: this can be default namespace
namespace = os.environ.get("DYNAMO_NAMESPACE", "dynamo")

@service(namespace=namespace, resource=Resource(gpu=1, cpu="0.5", memory="3Gib"))
class MyComponent(BaseComponent):

    def __init__(self, instance: Instance):
        self.instance = instance

    @async_init
    def init_clients():
        self.another_component = AnotherComponent.client()

    @endpoint
    async def foo(self, req: RequestTypeA) -> ResponseTypeA:
        # call endpoint in another component
        res = await self.another_component.bar(req.some_attribute)

    @endpoint
    async def baz(self, req: RequestTypeB) -> ResponseTypeB:
        pass

if __name__ == "__main__":
    uvloop.install()
    uvloop.run(MyComponent.run())
```



# Resolved Issues

## 1.  Dynamo Context is a free-form dict
Use unified Dynamo Identifier based on [component descriptor proposal](https://github.com/ai-dynamo/enhancements/blob/89d87e9b962a953cd9d5e66b205eda53a6810baa/enhancements/0000-component-descriptor-model.md#descriptor-types)

```python

class Identifier(BaseModel):
    """Base descriptor for component identification."""
    namespace: str
    component: Optional[str] = None
    endpoint: Optional[str] = None


class Instance(BaseModel):
    """Identifier with instance ID (immutable once created)."""
    identifier: Identifier
    instance_id: int

```

## 2. hard to get runtime inner details 

Sample Issue:
```
VLLM_WORKER_ID = dynamo_context["endpoints"][0].lease_id()

To get endpoints you have to know that there is a list of endpoints
Have to guess which one is the endpoint you actually want
```

Resolution: 

Instance provides direct access to operational entities and runtime
```python
VLLM_WORKER_ID = self.instance.instance_id
```
## 3 Inability to launch with python3

with proposed footer user can launch individual component or graph (using `dynamo serve`)

```python
if __name__ == "__main__":
    uvloop.install()
    uvloop.run(MyComponent.run())
```

# References:
- [component + discovery](https://github.com/ai-dynamo/enhancements/pull/11)
- [Consolidating dynamo-serve and dynamo-run component API](https://github.com/ai-dynamo/enhancements/pull/10)
