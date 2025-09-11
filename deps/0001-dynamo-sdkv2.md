# Goals

p0s:
- Components should be authored as classes
- Components should be runnable with python3
- component.start() should be without bloat

p1:
- components should expose core bindings for serving in some cases

## Proposal

In the following section I'll iterate through variations that show how different personas with different needs can use the SDK. default user -> mixed user -> total control user.

### Persona 1: Default User

```python
@service(
    name="worker",
    namespace="dynamo",
)
class Worker():
    def __init__(self):
        ...

    @endpoint()
    async def generate(self, request: ChatRequest):
        yield "token1"
        yield "token2"
        yield "token3"


if __name__ == "__main__":
    import asyncio
    import uvloop

    asyncio.run(Worker.start())
```

Solves:
- can now run a component with python3

What is happening under the hood:
- start() is serving my endpoints for me

What do we need to add next?
- Ability to give access to the runtime, endpoints, component back to the user

### Persona 2: Mixed User (wants access to dist runtime)

Example: I want to use the runtime to create clients directly to other services on the runtime.

```python
from dynamo.sdk import DynamoContext

@service(
    name="worker",
    namespace="dynamo",
)
class Router():
    # dynamo context injected by start()
    def __init__(self, dynamo_context: DynamoContext):
        self.runtime = dynamo_context.runtime

    @async_init
    async def async_init(self):
        # e.g generate a direct client to the worker from the router
        self.worker_client = self.runtime.namespace(self.namespace).component("worker").endpoint("generate").client()

    @endpoint()
    async def generate(self, request: ChatRequest):
        # call the client to worker
        for token in self.worker_client.generate(request):
            yield token

if __name__ == "__main__":
    import asyncio
    import uvloop

    # start now injects the DynamoContext since it's a named argument in the constructor
    asyncio.run(Router.start())

# ... Defined in the sdk else where
# Typed 
class DynamoContext(BaseModel):
    runtime: DistributedRuntime
    component: Component
    endpoints: List[Endpoint]
    name: str
    namespace: str
``` 

Solves:
- giving user access to the runtime, component, endpoints without a global dynamo_context dict
    - user can introspect start() function to see how this injection is happening
    - extremely clear when dynamo_context is populated, since it is being injected

What is happening under the hood:
- start() is still serving my endpoints for me
- start() creates a DynamoContext instance and injects it into the constructor

What do we need to add next?
- what if I want total control over how the endpoints are served and don't want anything to happen automatically? What if I want visibility into the bindings that are facilitating the serving of the component and not have that hidden behind a start() function?

### Persona 3: Power user: Total control over how the endpoints are served and don't want anything to happen automatically. 

Wants top level code to use raw bindings to serve component.

```python
from dynamo.sdk import DynamoContext

@service(
    name="worker",
    namespace="dynamo",
)
class Router():
    def __init__(self, dynamo_context: DynamoContext):
        self.ctx = dynamo_context

    # @entrypoint decorator allows a user to totally override how the component will be served leaving everything to the user
    # specifying it means that components/endpoints will not automatically be served
    @entrypoint
    async def entrypoint(self):
        # CORE BINDINGS: Visibility into the bindings that are facilitating the serving of the component
        # Delegates full control back to the user
        # Allows the user to do everything manually
        await self.ctx.component.create_service()
        await self.ctx.component.endpoint("generate").serve_endpoint(self.generate)
    
    # no endpoint decorator since entrypoint is directly registering this function
    async def generate(self, request: ChatRequest):
        # call the client to worker
        for token in self.worker_client.generate(request):
            yield token

if __name__ == "__main__":
    import asyncio
    import uvloop

    asyncio.run(Router.start())
``` 

Solves:
- core bindings are exposed, just not in the footer, but cleanly encapsulated in the component
- user can now see how the component is actually being served

## Definition of start

To meet our goal of keeping start super simple, it should simply look like the following

```python
from dynamo.runtime import DistributedRuntime, dynamo_worker
class ServiceInterface(BaseModel):
    ...

    async def start(self, runtime: DistributedRuntime = None):
        # 1. Create a runtime/dyn worker if one is not passed
        # 2. Init the context
        # 3. Init the inner class injecting the context
        # 4. Run async init to do any async setup
        # 5. Use default entrypoint or user passed entrypoint if one is decorated
```
