# A Python-first frontend for Dynamo

**Status**: Draft

**Authors**: Graham King

**Category**: Architecture

**Replaces** (partially): https://github.com/ai-dynamo/enhancements/pull/50

# Summary

The frontend (`python -m dynamo.frontend`) is currently implemented in Rust, with a very thin Python layer to invoke it. This proposes changing that so that frontend request processing happens entirely in Python.

Dynamo's Rust code will only handle the HTTP part. Everything else happens in the Python handler: Pre-processing, routing, and post-processing. That Python can use our Rust pieces, such as the router or the pre/post processor.

We provide a handler that uses the engine's input and output processors. It is enabled use via a command line flag or environment variable. The user can also copy the frontend Python code and provide their own Python handler.

# Motivation

1. **Mistral** ships their template and tokenizer as a Python library (`mistral-common`). They do not have an official Jinja file. To do pre-processing the official way we must call their Python library.

2. `vllm` and `sglang` sometimes get **day-0 access** to new models, which Dynamo does not. If we could optionally switch to using vllm and sglang for pre/post processing we would support more models on day of release.

3. Dynamo's goal is to provide **high-performance Rust components usable from Python**. Moving control of frontend request processing into Python enables that.

# Proposal, as pseudo-code

Optionally attach a callback which will handle the entire request.

## Setup

Set a Python handler for the entire request:

```
    if args.use_native_handler:
        kwargs['handler'] = py_handler ## <--- THE EXCITING NEW PART

    e = EntrypointArgs(EngineType.Dynamic, **kwargs)
    engine = await make_engine(runtime, e)

    await run_input(runtime, "http", engine)
```

## Request handling

Example is for vllm. We would provide one for each engine.

```
from vllm.v1.engine.async_llm import AsyncLLM
engine = vllm.AsyncLLM.from_vllm_config(...)

component = runtime.namespace(config.namespace).component(config.component)
generate_endpoint = component.endpoint(config.endpoint)
endpoint_client = await generate_endpoint.client()

def py_handler(request: dynamo.NVCreateChatCompletionRequest) -> dynamo.NvCreateChatCompletionResponse:

    # Let vllm handle all pre-processing
    vllm_preproc: vllm.EngineCoreRequest = engine.input_processor.process_inputs(..request fields..)
    engine.output_processor.add_request(vllm_preproc)

    # Convert to our type
    our_preproc: dynamo.PreprocessedRequest = convert(vllm_preproc)

    # Dynamo Router. This goes to the backend, waits, gets the streaming response, returns it
    # stream is AsyncResponseStream
    dynamo_stream: dynamo.AsyncResponseStream = await endpoint_client.round_robin(our_preproc)
    vllm_stream: [EngineCoreOutput] = map_it_somehow(dynamo_stream)

    # Let vllm handle all post-processing
    vlm_out: vllm.RequestOutput = engine.output_processor.process_output(vllm_stream)

    dynamo_out: dynamo.NvCreateChatCompletionResponse = convert(vllm_out)
    return dynamo_out
```

## What we already have

### A Python engine

We have an `HttpAsyncEngine` binding, that wraps a `PythonAsyncEngine` that wraps a `PythonAsyncEngine`. This can form the basis / proof-of-concept to make a Python engine. I prefer us to not use the existing `HttpAsyncEngine` because it is quite old, it doesn't do discovery, and various other things. We only want one place where we start an HTTP server.

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

### Python handler support in Rust

Using the engine we already have "A Python engine", we need to wire up calling the `kwargs['handler']` Python callback.

### Type conversion

To call the vllm input and output processors we need to convert the types, the imaginary `convert` functions in the pseudo-code.

# Misc

## Downside

This will most likely run slower than the current setup, because there is more Python and less Rust. Tokenization and de-tokenization are heavy CPU users, so moving them to Python (e.g. `mistral-common`) is expected to be slower.

The current frontend will remain, allowing users to choose.

## Out of scope

Bindings for the current Rust pre and post processors is out of scope for this proposal.

# Related Proposals

https://github.com/ai-dynamo/enhancements/pull/50

