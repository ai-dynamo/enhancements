# <Modular Pipeline Architecture for Dynamo Frontend>

**Status**: Draft 

**Authors**: [Name/Team] 

**Category**: Architecture

**Replaces**: Current build_routed_pipeline_with_preprocessor implementation in lib/llm/src/entrypoint/input/common.rs 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

This proposal calls for a modular, configuration-driven pipeline architecture for the Dynamo frontend. Instead of the current monolithic ``build_routed_pipeline_with_preprocessor`` function and complex internal branching, we propose:
- Independent configuration knobs for preprocessing location, postprocessing location, routing behavior, and migration
- A single build_pipeline(config, components) function that assembles pipeline segments based on configuration
- Separation of concerns between worker routing decisions and disaggregated prefill/decode orchestration
- Support for three primary use-cases: query-only mode (GAIE EPP stage 1), direct-to-known-workers mode (GAIE stage 2), and full discovery mode (dynamo default)

# Motivation

The current ``build_routed_pipeline_with_preprocessor`` function in ``lib/llm/src/entrypoint/input/common.rs`` has grown to handle multiple concerns in a tightly-coupled manner:
- Preprocessing is always in the frontend: There is no clean way to skip frontend preprocessing when the engine backend handles its own tokenization and prompt formatting.
- Routing logic is scattered: The decision to route to a specific worker is controlled through backend_instance_id in annotations/routing hints, which is not elegant. The RouterMode::Direct(instance_id) can be configured during the pipeline construction time, not per-request.
- PrefillRouter does too much: It handles prefill/decode orchestration, GAIE Stage 1/2 state machine, bootstrap info discovery, worker selection, and fallback to aggregated mode - all in one component. In the llm-d and the architecture adopted by the Inference Gateway the decode worker calls prefill in a dedicated sidecar, not in the routing module.
- No query-only mode: There is no clean way to run only preprocessing + routing decisions. It is handled by an annotation flag and a short-circut. 
- Migration is always included: No way to opt-out of retry logic when it's not needed (e.g., single-worker scenarios).


These limitations make it difficult to support new use-cases like:
- Inference Gateway API approach
- Clients who want to handle their own preprocessing / post processing in their modules or rely on Inference Engines. See [`backend fallback` draft](https://github.com/ai-dynamo/enhancements/pull/50/changes#diff-d06f262c7a840682785ad441a3dcaef9ba9f6938bc0e71d1895c89874e7f4086R177)


# Constraints
- Preprocessing must always be in the frontend because KV cache routing requires tokens to calculate hashes for matching against indexer hashes from KV events.
-  Postprocessing can be frontend or engine depending on whether the user wants Rust processing (default) or native framework processing (fallback).

## Goals

* Enable flexible pipeline composition through independent, orthogonal configuration knobs

* Support preprocessing and postprocessing in either frontend or engine backend

* Cleanly separate routing decisions from transport and from disaggregation orchestration

* Support Inference Gateway API 

* Maintain backward compatibility with existing behavior as the default configuration

* Make pipeline behavior explicit and configurable via CLI flags or config files

### Non Goals

* Changing the wire protocol between frontend and backend

* Modifying the underlying PushRouter or transport implementations

* Changing how discovery works

* Changing the OpenAI API compatibility layer

## Requirements

### REQ 1 Independent Configuration Knobs
- The pipeline **MUST** support independent configuration of:
- Preprocessing location: Frontend
- Postprocessing location: Frontend or Engine
- Routing behavior: QueryOnly, DirectToKnown, or Discover
- Migration: Enabled or Disabled
- Each knob **MUST** be independently configurable and **SHOULD** have sensible defaults that match current behavior.

### REQ 2 GAIE compatibility

- The system MUST support a query-only (EPP) mode where:
- Preprocessing runs inside the pipeline before routing or through bindings in the EPP Body based routing.
- Routing decisions are made (worker selection for both aggregated and disaggregated cases)
- The response contains tokens and selected worker IDs
- Model execution does NOT occur

- The system MUST support routing directly to pre-selected workers where:
- Worker IDs are specified in the request (via routing.backend_instance_id, routing.prefill_worker_id, routing.decode_worker_id)
- No routing logic executes to find workers
- The request is sent directly to the specified worker(s)

### REQ 3 Disaggregated serving
The decision to use disaggregated (prefill + decode) vs aggregated serving **MUST** be made per-request based on prefill worker availability, NOT as a pipeline configuration. This is the current behavior. But The DisaggOrchestrator component **MUST** be taken out of the router.

### REQ 4 Configuration Validation
- The code **MUST** validate configuration combinations and return clear errors for invalid combinations
- Other invalid combinations SHOULD be identified and rejected with descriptive error messages

# Proposal

We propose replacing the current pipeline construction approach with a configuration-driven, single-function design. The function will take the PipelineConfig as a parameter with the following knobs:
```md
preprocessing: Dynamo 
postprocessing: Dynamo | Engine     
routing: QueryOnly | DirectToKnown | Discover 
migration_enabled: bool    
```

The current pipeline architecture uses compile-time type chains where each .link() call returns a different type.
The advantage is that it assures type safety and uses generics, concrete types rather than runtime polymorphism consistent with idiomatic rust.

The downside it that this prevents runtime-conditional segment inclusion:
```rust
// NOT possible with current architecture:
let mut pipeline = frontend;
if config.has_preprocessing {
    pipeline = pipeline.link(preprocessor.forward_edge())?;  // Type changes here
}
```

Implementing such new interface requires a fair amount of work.
For the step 1 of this proposal we will code additional functions instead of the new pipeline.

PipelineConfig:
Pipeline configuration - each knob is independent
```rust
pub struct PipelineConfig {    
    /// Where does postprocessing happen?
    pub postprocessing: ProcessingLocation,
    
    /// How is routing handled?
    pub routing: RoutingBehavior,
    
    /// Enable migration/retry on worker failure?
    pub migration_enabled: bool,
}

pub enum ProcessingLocation {
    /// Processing happens in the frontend (Dynamo EPP side)
    #[default]
    Frontend / Dynamo,
    
    /// Processing happens in the engine backend
    Engine,
}

pub enum RoutingBehavior {
    /// Only find workers, return tokens + worker IDs, don't execute model
    QueryOnly,
    
    /// Workers are already in the request, route directly to them
    DirectToKnown,
    
    /// Find workers and route to them (current default behavior)
    #[default]
    Discover,
}
```

Stage 1 proposal:

```rust
/// Main entry point - selects pipeline variant based on config
pub async fn build_pipeline<Req, Resp>(
    config: &PipelineConfig,
    components: &PipelineComponents,
) -> anyhow::Result<ServiceEngine<SingleIn<Req>, ManyOut<Annotated<Resp>>>>
where
    Req: Data,
    Resp: Data,
{
    match (&config.routing, &config.postprocessing) {
        // 1. Query-only 
        (RoutingBehavior::QueryOnly, _) => {
            build_query_only_pipeline(components).await
        }
        
        // 2. Direct + Rust postprocessing
        (RoutingBehavior::DirectToKnown, ProcessingLocation::Frontend) => {
            build_direct_rust_pipeline(components, config.migration_enabled).await
        }
        
        // 3. Direct + Native postprocessing
        (RoutingBehavior::DirectToKnown, ProcessingLocation::Engine) => {
            build_direct_native_pipeline(components, config.migration_enabled).await
        }
        
        // 4. Discover + Rust postprocessing (current default)
        (RoutingBehavior::Discover, ProcessingLocation::Frontend) => {
            build_discover_rust_pipeline(components, config.migration_enabled).await
        }
        
        // 5. Discover + Native postprocessing
        (RoutingBehavior::Discover, ProcessingLocation::Engine) => {
            build_discover_native_pipeline(components, config.migration_enabled).await
        }
    }
}
```

We will have 5 pipeline functions:
```rust
/// Query-only pipeline: preprocessing + routing decision, no execution
async fn build_query_only_pipeline<Req, Resp>(
    components: &PipelineComponents,
) -> anyhow::Result<ServiceEngine<SingleIn<Req>, ManyOut<Annotated<Resp>>>> {
    let frontend = SegmentSource::<SingleIn<Req>, ManyOut<Annotated<Resp>>>::new();
    
    let preprocessor = build_preprocessor(components)?.into_operator();
    let backend = Backend::from_tokenizer(components.tokenizer.clone()).into_operator();
    let route_query = RouteQueryBackend::new(components.router.clone());
    let service_backend = ServiceBackend::from_engine(Arc::new(route_query));
    
    Ok(frontend
        .link(preprocessor.forward_edge())?
        .link(backend.forward_edge())?
        .link(service_backend)?
        .link(backend.backward_edge())?
        .link(preprocessor.backward_edge())?
        .link(frontend)?)
}

/// ... the rest of the cases. 
To shrink the amount of boilerplate code we can define a macro that generates pipeline functions.
```

CLI usage:

```rust
# Use-case 1: Gaie stage 1 only (preprocessing + find workers, no execution)
dynamo-run \
  --routing query-only \
  --in http --out auto

# Use-case 2: Gaie stage 2 Execute with pre-selected workers, engine does preprocessing
dynamo-run \
  --postprocessing engine \
  --routing direct \
  --in http --out auto

# Use-case 3: Full pipeline with discovery (current default)
dynamo-run \
  --routing discover \
  --in http --out auto

# Use-case 3 variant: Engine preprocessing, engine postprocessing
dynamo-run \
  --postprocessing engine \
  --routing discover \
  --migration-enabled false \
  --in http --out auto
```

New components:

Disag Orchestrator
```rust
/// Orchestrates disaggregated prefill/decode flow
/// Decision to use disagg is made PER-REQUEST based on prefill worker availability
pub struct DisaggOrchestrator {
    router: Arc<dyn WorkerRouter>,
    prefill_transport: Arc<PrefillTransport>,
}

impl DisaggOrchestrator {
    pub fn new(
        router: Arc<dyn WorkerRouter>,
        client: Client,
    ) -> Result<Self> {
        // ...
    }
}

#[async_trait]
impl Operator<
    SingleIn<PreprocessedRequest>,
    ManyOut<Annotated<LLMEngineOutput>>,
    SingleIn<PreprocessedRequest>,
    ManyOut<Annotated<LLMEngineOutput>>,
> for DisaggOrchestrator {
    async fn generate(
        &self,
        request: SingleIn<PreprocessedRequest>,
        next: ServerStreamingEngine<PreprocessedRequest, Annotated<LLMEngineOutput>>,
    ) -> Result<ManyOut<Annotated<LLMEngineOutput>>> {
        let req = request.data();
        
        // Per-request decision: check if prefill worker is available
        let use_disagg = self.router.has_prefill_worker().await;
        
        if !use_disagg {
            // Aggregated: pass through to decode (which handles everything)
            return next.generate(request).await;
        }
        
        // Disaggregated flow:
        // 1. Get prefill worker (from request or query router)
        // 2. Execute prefill
        // 3. Prepare decode request with prefill result
        // 4. Pass to decode (next in chain)
        
        // ... implementation ...
    }
}
```

# Alternative / Stage 2 proposal.
To preserve 1 pipeline we will have to resort to the type erasure / traits.
The downsides are the loss of compile-time safety, bugs will become run-time errors.
There is a performance overhead given the heap allocation and that every call to process() goes through a vtable lookup. But it can be considered negligible in the inference context?

```rust
// new trait for each stage

pub trait DynStage: Send + Sync + 'static {
    /// Process input, produce output stream
    fn process(
        &self,
        input: DynRequest,
    ) -> BoxFuture<'_, Result<BoxStream<'_, DynResponse>>>;

    /// Optional backward edge for bidirectional stages
    fn backward_edge(&self) -> Option<Arc<dyn DynStage>> {
        None
    }
}

// This trait will have to be implemented for the OpenAIPreprocessor, KvPushRouter, Migration, Backend, PrefillRouter, 

// Type-Erased Request/Response

pub struct DynRequest(Box<dyn Any + Send>);
pub struct DynResponse(Box<dyn Any + Send>);

impl DynRequest {
    pub fn new<T: Send + 'static>(value: T) -> Self { Self(Box::new(value)) }
    pub fn downcast<T: 'static>(self) -> Result<T> { /* ... */ }
}

// Pipeline Builder

pub struct DynPipelineBuilder {
    stages: Vec<Arc<dyn DynStage>>,
}

// this will determine which steps go in which order. 
impl DynPipelineBuilder {
    pub fn new() -> Self { Self { stages: vec![] } }

    pub fn add(mut self, stage: impl DynStage) -> Self {
        self.stages.push(Arc::new(stage));
        self
    }

    pub fn add_if(self, condition: bool, stage: impl DynStage) -> Self {
        if condition { self.add(stage) } else { self }
    }

    pub fn build(self) -> Result<DynPipeline> {
        // Links stages together, wires backward edges
        Ok(DynPipeline { /* ... */ })
    }
}

// The Built Pipeline 

pub struct DynPipeline {
    head: Arc<dyn DynStage>,
}

impl DynPipeline {
    pub async fn execute(&self, req: DynRequest) -> Result<BoxStream<'_, DynResponse>> {
        self.head.process(req).await
    }
}

// Usage:
pub async fn build_pipeline(config: &PipelineConfig) -> Result<DynPipeline> {
    DynPipelineBuilder::new()
        .add(ServiceFrontend::new())
        .add_if(config.preprocessing == Frontend, OpenAIPreprocessor::new())
        .add_if(config.routing == Discover, KvPushRouter::new())
        .add_if(config.routing == Direct, DirectTransport::new())
        .add_if(config.migration_enabled, Migration::new())
        .add_if(config.postprocessing == Frontend, Backend::new())
        .build()
}
```