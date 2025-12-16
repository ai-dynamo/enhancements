# A Python-first frontend for Dynamo

**Status**: Draft

**Authors**: Graham King

**Category**: Architecture

**Replaces** (partially): https://github.com/ai-dynamo/enhancements/pull/50

# Summary

The frontend (`python -m dynamo.frontend`) is currently implemented in Rust, with a very thin Python layer to invoke it. This proposes changing that so that frontend request processing is entirely orchestrated in Python.

Dynamo's Rust code will only handle the HTTP part. Everything else happens in the Python handler: Pre-processing, routing, and post-processing. That Python can use our Rust pieces, such as the router or the pre/post processor.

We will provide a version that uses the engine's input and output processors. It is enabled via a command line flag or environment variable. The user can also copy the frontend Python code and provide their own Python implementation.

# Motivation

1. **Mistral** ships their template and tokenizer as a Python library (`mistral-common`). They do not have an official Jinja file. To do pre-processing the official way we must call their Python library.

2. `vllm` and `sglang` sometimes get **day-0 access** to new models, which Dynamo does not. If we could optionally switch to using vllm and sglang for pre/post processing we would support more models on day of release.

3. Dynamo's goal is to provide **high-performance Rust components usable from Python**. Moving control of frontend request processing into Python enables that.

# Proposal, as pseudo-code

A Python engine for each model, on the frontend. That has full control over request handling.

The engine has to be per-model, so that is uses the correct pre/post processor and so on. The frontend doesn't know which models are available until the instances register. Hence I propose a Python engine factory.

Example is for vllm. We would provide one for each engine.

## Setup

Set a Python handler for the entire request. This is in `components/src/dynamo/frontend/main.py` on startup.

```
    ### THE EXCITING NEW PART

    # kwargs contains all the command line flags, such as router mode. The engine factory will use those.
    if args.use_native_engine:
        py_engine = PyEngine.new(kwargs)
        kwargs['engine_factory'] = py_engine.factory

    ### END EXCITING NEW PART

    e = EntrypointArgs(EngineType.Dynamic, **kwargs)
    engine = await make_engine(runtime, e)

    await run_input(runtime, "http", engine)
```

## On new model

The normal HTTP server and Discovery service run like before. When a new backend appears, we call the engine factory. Probably it has saved `EntrypointArgs` / `kwargs` from Setup step:

```
from vllm.v1.engine.async_llm import AsyncLLM

class PyEngine:

    def factory(self, mdc: ModelDeploymentCard) -> PythonAsyncEngine:

        # TODO: We need input_processor and output_processor, but don't load the weights
        self.engine = vllm.AsyncLLM.from_vllm_config(...)

        # Download the model if necessary
        fetch_llm(mdc.model_name)

        self.endpoint_client = await mdc.endpoint.client()

        # Implements our standard `AsyncEngine` trait. Already exists.
        return PythonAsyncEngine.new(self.generate, event_loop)
```


## Request handling

The Rust HTTP handler will call our `PythonAsyncEngine`, which delegates to the registered generator, here `self.generate`.

```
class PyEngine:

    def generate(self, request: dynamo.NVCreateChatCompletionRequest) -> dynamo.NvCreateChatCompletionStreamResponse:

        # Let vllm handle all pre-processing
        vllm_preproc: vllm.EngineCoreRequest = self.engine.input_processor.process_inputs(..request fields..)
        self.engine.output_processor.add_request(vllm_preproc)

        # Convert to our type
        our_preproc: dynamo.PreprocessedRequest = convert(vllm_preproc)

        # Dynamo Router. This goes to the backend, waits, gets the streaming response, returns it
        # stream is AsyncResponseStream
        dynamo_stream: dynamo.AsyncResponseStream = await self.endpoint_client.round_robin(our_preproc)

        async for dynamo_response in dynamo_stream:
            vllm_response: [EngineCoreOutput] = convert(dynamo_response)

            # Let vllm handle all post-processing
            vlm_out: vllm.RequestOutput = self.engine.output_processor.process_output(vllm_response)

            dynamo_out: dynamo.NvCreateChatCompletionResponse = convert(vllm_out)

            # Rust now handles Server Sent Events back to user
            yield dynamo_out
```

This approach preserves most of the discovery handling. Useful things discovery watcher does:
- Discover the existence of a new instance, which endpoint it will respond to, and which model it serves
- Create the routing Client
- Notify http service to open correct endpoints (e.g. `/v1/chat/completions` for a chat completions model)
- Notify metrics

## Teardown

When the last instance serving a model shuts down, the frontend will unregister our model. Do we need to tell the Python engine that it is stopping? vllm/sglang have many sub-processes, so likely yes.

## What we already have

### A Python engine

We have an `HttpAsyncEngine` binding, that wraps a `PythonAsyncEngine` that wraps a `PythonServerStreamingEngine`. This can form the basis / proof-of-concept to make a Python engine. I prefer us to not use the existing `HttpAsyncEngine` because it is quite old, it doesn't do discovery, and various other things. We only want one place where we start an HTTP server.

### Python router bindings

These two lines should work today:
```
    endpoint_client = await generate_endpoint.client()
```
and
```
    dynamo_stream: dynamo.AsyncResponseStream = await endpoint_client.round_robin(our_preproc)
```

## What we need

### Python engine factory called from Rust

The Rust type for the engine is `Arc<dyn AsyncEngine<Context<NvCreateChatCompletionRequest>, Pin<Box<dyn AsyncEngineStream<Annotated<CreateChatCompletionStreamResponse>>>>, Error>>`, but when talking to Python it might be `serde::Json` instead of `NvCreateChatCompletion...`.

Pass `ModelWatcher` a function to make the engine, instead of calling `build_routed_pipeline`. Pass it in through `LocalModel` (created in Python bindings `run_input`, so that's our entry point), `entrypoint/http.rs::run_watcher`, `discovery/watcher.rs::ModelWatcher::new`.

This function will be called from `discovery/watcher.rs::handle_put` where it now calls `build_routed_pipeline`.

### Type conversion

To call the vllm input and output processors we need to convert the types, the imaginary `convert` functions in the pseudo-code.

# Open questions

1. The example handler takes `NvCreateChatCompletionRequest`. What about Completions, Requests, Embeddings, and Tensors? Do we have a separate handler for each? Should a single handler be able to cope with all of them, and we indicate in the call which type it is? Or it could take a union and figure it out.

2. Can we get a vllm InputProcessor / OutputProcessor without loading the weights into memory? The frontend can't load the weights for mulitple models concurrently. Maybe we don't need to create the AsyncLLM, can directly call InputProcessor.new ?

# Misc

## Using it with KVRouter

KVRouter has good [Python bindings - docs -](https://github.com/ai-dynamo/dynamo/blob/main/docs/router/kv_cache_routing.md#using-kvpushrouter-python-api):

In factory:
```
    kv_router_config = KvRouterConfig()
    kv_router = KvPushRouter(
        endpoint=endpoint,
        block_size=16,
        kv_router_config=kv_router_config
    )
```

In generate:

```
    stream = kv_router.generate(token_ids=..., ...)
```

## Downside

This will most likely run slower than the current setup, because there is more Python and less Rust. Tokenization and de-tokenization are heavy CPU users, so moving them to Python (e.g. `mistral-common`) is expected to be slower.

The current frontend will remain, allowing users to choose.

## Out of scope

- Bindings for the current Rust pre and post processors.
- Multi-process frontends, to handle GIL contention.
- Experimenting with free-threaded python, to handle GIL contention.

# Related Proposals

https://github.com/ai-dynamo/enhancements/pull/50

