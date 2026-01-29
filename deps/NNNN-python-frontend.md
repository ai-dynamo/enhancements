# A Python-first frontend for Dynamo

**Status**: Active

**Authors**: Graham King

**Category**: Architecture

**Replaces** (partially): https://github.com/ai-dynamo/enhancements/pull/50

**Implements Pull Requests**:
    - https://github.com/ai-dynamo/dynamo/pull/4999
    - https://github.com/ai-dynamo/dynamo/pull/5544

# Done (move from TODO)

- Copy over all the possible sampling params between Dynamo and Vllm.
- Accept all vllm flags
- Try mistral
- Add KV routing (see section in this document)

# TODO (updated)

- Handle all vllm cmd line flags, pass to ModelConfig and/or VllmConfig. Do it similar to the vllm wrapper component.

- Tell vllm when the model is unloaded so it can stop (do we still need that with given that we're not using the engine?)

- Handle other types of request, is Chat Completions only right now:
    - Completions
    - Embeddings
    - Prompt embeddings

- Verify:
    - Can you run multiple requests at once?
    - Does it load prompt templates from external files, or always from tokenizer_config.json?
    - Does tool calling / parsing / etc work?

- Compare perf vs regular Dynamo


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

```python
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

```python
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

```python
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
```python
    endpoint_client = await generate_endpoint.client()
```
and
```python
    dynamo_stream: dynamo.AsyncResponseStream = await endpoint_client.round_robin(our_preproc)
```

# Open questions

1. The example handler takes `NvCreateChatCompletionRequest`. What about Completions, Requests, Embeddings, and Tensors? Do we have a separate handler for each? Should a single handler be able to cope with all of them, and we indicate in the call which type it is? Or it could take a union and figure it out.

# Misc

## Using it with KVRouter

KVRouter has good [Python bindings - docs -](https://github.com/ai-dynamo/dynamo/blob/main/docs/router/kv_cache_routing.md#using-kvpushrouter-python-api):

In factory:
```python
    kv_router_config = KvRouterConfig()
    kv_router = KvPushRouter(
        endpoint=endpoint,
        block_size=16,
        kv_router_config=kv_router_config
    )
```

In generate:

```python
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

----------

# Appendix

## Appendix A.

spec.md for injecting the Python engine factory callback
Opus 4.5 Thinking implemented this to create https://github.com/ai-dynamo/dynamo/pull/4999

# Spike: Python Engine Factory Callback

## Objective

Add the ability to pass a Python async function (`engine_factory`) as a callback into the Rust code. This spike verifies that we can successfully call an async Python function from Rust code when a model is discovered.

## Background

The callback signature is:
```python
async def engine_factory(mdc: ModelDeploymentCard) -> PythonAsyncEngine:
    ...
```

- `ModelDeploymentCard` - Rust object with Python bindings in `lib/bindings/python/rust/llm/model_card.rs`
- `PythonAsyncEngine` - Rust object with Python bindings in `lib/bindings/python/rust/engine.rs`

When an async Python function is passed to Rust, we need to:
1. Capture Python's event loop context (TaskLocals) at registration time
2. Use those locals when converting the Python coroutine to a Rust Future

## Reference: Existing Pattern

The `register_engine_route` function in `lib/bindings/python/rust/lib.rs` (lines 627-703) demonstrates how to properly handle async Python callbacks:

```rust
fn register_engine_route(&self, py: Python<'_>, route_name: String, callback: PyObject) -> PyResult<()> {
    // Capture TaskLocals at registration time
    let locals = Arc::new(pyo3_async_runtimes::tokio::get_current_locals(py).map_err(to_pyerr)?);
    let callback = Arc::new(callback);

    let rust_callback: EngineRouteCallback = Arc::new(move |body: serde_json::Value| {
        let callback = callback.clone();
        let locals = locals.clone();

        Box::pin(async move {
            let py_future = Python::with_gil(|py| {
                let py_body = pythonize::pythonize(py, &body)?;
                let coroutine = callback.call1(py, (py_body,))?;
                pyo3_async_runtimes::into_future_with_locals(&locals, coroutine.into_bound(py))
            })?;

            let py_result = py_future.await?;

            Python::with_gil(|py| {
                pythonize::depythonize::<serde_json::Value>(py_result.bind(py))
            })
        })
    });
    // ...
}
```

## Implementation Steps

### Step 1: Create the Python dummy callback

**File:** `components/src/dynamo/frontend/main.py`

Add a dummy `engine_factory` function that returns a minimal `PythonAsyncEngine`. This will be used to test the callback mechanism.

1. Import the necessary types at the top of the file:
   ```python
   from dynamo.llm import ModelDeploymentCard, PythonAsyncEngine
   ```

2. Create a dummy async generator function (required for `PythonAsyncEngine`):
   ```python
   async def dummy_generator(request):
       """Minimal generator that yields nothing - just for spike testing."""
       return
       yield  # Makes this an async generator
   ```

3. Create the `engine_factory` callback:
   ```python
   async def engine_factory(mdc: ModelDeploymentCard) -> PythonAsyncEngine:
       """
       Spike: Dummy engine factory callback.
       Called by Rust when a model is discovered.
       """
       import asyncio
       loop = asyncio.get_event_loop()
       print(f"[SPIKE] engine_factory called with MDC: {mdc.to_json_str()[:100]}...")
       return PythonAsyncEngine(dummy_generator, loop)
   ```

4. In the `async_main()` function, add to `kwargs` before creating `EntrypointArgs`:
   ```python
   # Get TaskLocals for the callback
   kwargs['engine_factory'] = engine_factory
   ```

### Step 2: Define the Rust callback type

**File:** `lib/llm/src/entrypoint.rs` (or create a new file `lib/llm/src/engine_factory.rs`)

Define a type alias for the engine factory callback that mirrors `EngineRouteCallback`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::Arc;
use crate::model_card::ModelDeploymentCard;

/// Callback type for engine factory (async)
/// Takes a ModelDeploymentCard, returns unit (for now - spike only logs success)
pub type EngineFactoryCallback = Arc<
    dyn Fn(
        ModelDeploymentCard,
    ) -> Pin<Box<dyn Future<Output = anyhow::Result<()>> + Send>>
        + Send
        + Sync,
>;
```

### Step 3: Add `engine_factory` parameter to `EntrypointArgs`

**File:** `lib/bindings/python/rust/llm/entrypoint.rs`

1. Add new imports at the top:
   ```rust
   use pyo3_async_runtimes::TaskLocals;
   ```

2. Add a new field to the `EntrypointArgs` struct:
   ```rust
   #[pyclass]
   #[derive(Clone, Debug)]
   pub(crate) struct EntrypointArgs {
       // ... existing fields ...
       engine_factory: Option<EngineFactoryCallback>,
   }
   ```

   **Note:** Since `EngineFactoryCallback` contains `dyn Fn`, the struct cannot derive `Clone` automatically. You'll need to wrap it in `Arc` or change the approach. Consider:
   ```rust
   engine_factory: Option<Arc<EngineFactoryCallback>>,
   ```
   Or store raw `PyObject` + `TaskLocals` separately and create the callback later.

3. Update the `#[new]` method signature to accept the callback:
   ```rust
   #[pyo3(signature = (engine_type, ..., engine_factory=None))]
   pub fn new(
       py: Python<'_>,  // Need py for get_current_locals
       // ... existing params ...
       engine_factory: Option<PyObject>,
   ) -> PyResult<Self> {
   ```

4. Inside `new()`, convert the Python callback to Rust:
   ```rust
   let engine_factory_callback: Option<EngineFactoryCallback> = engine_factory.map(|callback| {
       let locals = Arc::new(
           pyo3_async_runtimes::tokio::get_current_locals(py)
               .expect("Failed to get TaskLocals")
       );
       let callback = Arc::new(callback);

       let f: EngineFactoryCallback = Arc::new(move |card: ModelDeploymentCard| {
           let callback = callback.clone();
           let locals = locals.clone();

           Box::pin(async move {
               let py_future = Python::with_gil(|py| {
                   // Convert ModelDeploymentCard to Python object
                   let py_card = /* create Python ModelDeploymentCard wrapper */;

                   let coroutine = callback.call1(py, (py_card,))
                       .map_err(|e| anyhow::anyhow!("Failed to call engine_factory: {}", e))?;

                   pyo3_async_runtimes::into_future_with_locals(&locals, coroutine.into_bound(py))
                       .map_err(|e| anyhow::anyhow!("Failed to convert coroutine: {}", e))
               })?;

               let _py_result = py_future.await
                   .map_err(|e| anyhow::anyhow!("engine_factory failed: {}", e))?;

               // For spike: just log success, don't use the returned PythonAsyncEngine
               tracing::info!("[SPIKE] engine_factory callback succeeded!");
               Ok(())
           })
       });
       f
   });
   ```

### Step 4: Thread the callback through `make_engine`

**File:** `lib/bindings/python/rust/llm/entrypoint.rs`

In `make_engine()`, pass the callback to `LocalModelBuilder`:

```rust
if let Some(factory) = args.engine_factory.clone() {
    builder.engine_factory(Some(factory));
}
```

### Step 5: Add `engine_factory` field to `LocalModelBuilder` and `LocalModel`

**File:** `lib/llm/src/local_model.rs`

1. Add the field to `LocalModelBuilder`:
   ```rust
   pub struct LocalModelBuilder {
       // ... existing fields ...
       engine_factory: Option<EngineFactoryCallback>,
   }
   ```

2. Add builder method:
   ```rust
   pub fn engine_factory(&mut self, callback: Option<EngineFactoryCallback>) -> &mut Self {
       self.engine_factory = callback;
       self
   }
   ```

3. Add field to `LocalModel`:
   ```rust
   pub struct LocalModel {
       // ... existing fields ...
       engine_factory: Option<EngineFactoryCallback>,
   }
   ```

4. Add getter:
   ```rust
   pub fn engine_factory(&self) -> Option<&EngineFactoryCallback> {
       self.engine_factory.as_ref()
   }
   ```

5. Update `build()` to transfer the callback to `LocalModel`.

### Step 6: Thread through `EngineConfig`

**File:** `lib/llm/src/entrypoint.rs`

1. Update `EngineConfig::Dynamic` to include the callback:
   ```rust
   pub enum EngineConfig {
       Dynamic {
           model: Box<LocalModel>,
           engine_factory: Option<EngineFactoryCallback>,
       },
       // ... other variants unchanged
   }
   ```

2. Update `local_model()` method accordingly.

### Step 7: Pass to `run_watcher` in HTTP entrypoint

**File:** `lib/llm/src/entrypoint/input/http.rs`

1. Update `run_watcher` signature to accept the callback:
   ```rust
   async fn run_watcher(
       runtime: DistributedRuntime,
       model_manager: Arc<ModelManager>,
       router_config: RouterConfig,
       target_namespace: Option<String>,
       http_service: Arc<HttpService>,
       metrics: Arc<crate::http::service::metrics::Metrics>,
       engine_factory: Option<EngineFactoryCallback>,  // NEW
   ) -> anyhow::Result<()>
   ```

2. In `run()`, extract and pass the callback:
   ```rust
   EngineConfig::Dynamic { model, engine_factory } => {
       // ...
       run_watcher(
           distributed_runtime.clone(),
           http_service.state().manager_clone(),
           router_config.clone(),
           target_namespace,
           Arc::new(http_service.clone()),
           http_service.state().metrics_clone(),
           engine_factory,  // NEW
       ).await?;
   }
   ```

### Step 8: Pass to `ModelWatcher`

**File:** `lib/llm/src/discovery/watcher.rs`

1. Add field to `ModelWatcher`:
   ```rust
   pub struct ModelWatcher {
       // ... existing fields ...
       engine_factory: Option<EngineFactoryCallback>,
   }
   ```

2. Update `new()`:
   ```rust
   pub fn new(
       runtime: DistributedRuntime,
       model_manager: Arc<ModelManager>,
       router_config: RouterConfig,
       engine_factory: Option<EngineFactoryCallback>,  // NEW
   ) -> ModelWatcher {
       Self {
           manager: model_manager,
           drt: runtime,
           router_config,
           notify_on_model: Notify::new(),
           model_update_tx: None,
           engine_factory,  // NEW
       }
   }
   ```

### Step 9: Call the callback in `handle_put`

**File:** `lib/llm/src/discovery/watcher.rs`

In `handle_put()`, find the block that handles `card.model_type.supports_chat()` (around line 431). Add the callback invocation **before** calling `build_routed_pipeline`:

```rust
// SPIKE: Call the engine factory callback if present
if let Some(ref factory) = self.engine_factory {
    tracing::info!(
        model_name = card.name(),
        "[SPIKE] Calling engine_factory callback"
    );

    factory(card.clone()).await.map_err(|e| {
        tracing::error!(%e, "[SPIKE] engine_factory callback failed");
        e
    })?;

    tracing::info!(
        model_name = card.name(),
        "[SPIKE] engine_factory callback completed successfully"
    );
}

// Continue with existing build_routed_pipeline call...
if card.model_type.supports_chat() {
    let chat_engine = entrypoint::build_routed_pipeline::<...>(...).await?;
    // ...
}
```

## Build and Test

1. Navigate to bindings directory:
   ```bash
   cd lib/bindings/python
   ```

2. Build with maturin:
   ```bash
   maturin develop --uv
   ```

3. Run the frontend to test:
   ```bash
   python -m dynamo.frontend --help
   # Or with a test configuration
   ```

4. Expected output when a model is discovered:
   ```
   [SPIKE] engine_factory called with MDC: {"name":"...
   [SPIKE] Calling engine_factory callback
   [SPIKE] engine_factory callback completed successfully
   ```

## Challenges / Notes

1. **Clone for `EntrypointArgs`**: Since `EngineFactoryCallback` uses `dyn Fn`, it cannot be cloned directly. Options:
   - Wrap in `Arc` (recommended)
   - Store `PyObject` + `TaskLocals` separately and construct callback lazily
   - Remove `Clone` derive and adjust code accordingly

2. **Python object conversion**: The `ModelDeploymentCard` needs to be converted to its Python wrapper type. Check how this is done elsewhere in the bindings (see `model_card.rs`).

3. **GIL considerations**: All Python interactions must happen within `Python::with_gil()`. The actual async await happens outside the GIL.

4. **Error handling**: The spike can use simple error propagation. Production code should have more robust error handling.

## Files Changed Summary

| File | Changes |
|------|---------|
| `components/src/dynamo/frontend/main.py` | Add `engine_factory` callback and pass to kwargs |
| `lib/llm/src/entrypoint.rs` | Define `EngineFactoryCallback` type, update `EngineConfig::Dynamic` |
| `lib/bindings/python/rust/llm/entrypoint.rs` | Accept `engine_factory` in `EntrypointArgs`, convert to Rust callback |
| `lib/llm/src/local_model.rs` | Add `engine_factory` field to builder and model |
| `lib/llm/src/entrypoint/input/http.rs` | Thread callback through to `run_watcher` |
| `lib/llm/src/discovery/watcher.rs` | Store callback in `ModelWatcher`, call in `handle_put` |
