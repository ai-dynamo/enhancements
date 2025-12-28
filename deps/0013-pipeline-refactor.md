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

This proposal introduces a modular, configuration-driven pipeline architecture for the Dynamo frontend. Instead of the current monolithic build_routed_pipeline_with_preprocessor function and complex internal branching, we propose:
Independent configuration knobs for preprocessing location, postprocessing location, routing behavior, and migration
A single build_pipeline(config, components) function that assembles pipeline segments based on configuration
Separation of concerns between worker routing decisions and disaggregated prefill/decode orchestration
Support for three primary use-cases: query-only mode (GAIE EPP stage 1), direct-to-known-workers mode (GAIE stage 2), and full discovery mode (dynamo default)

# Motivation

The current build_routed_pipeline_with_preprocessor function in lib/llm/src/entrypoint/input/common.rs has grown to handle multiple concerns in a tightly-coupled manner:
Preprocessing is always in the frontend: There is no clean way to skip frontend preprocessing when the engine backend handles its own tokenization and prompt formatting.
Routing logic is scattered: The decision to route to a specific worker is controlled through backend_instance_id in annotations/routing hints, which is not elegant. The RouterMode::Direct(instance_id) can be configured during the pipeline construction time, not per-request.
PrefillRouter does too much: It handles prefill/decode orchestration, GAIE Stage 1/2 state machine, bootstrap info discovery, worker selection, and fallback to aggregated mode—all in one component. In the llm-d and the architecture adopted by the Inference Gateway the decode worker calls prefill in a dedicated sidecar, not in the routing module.
No query-only mode: There is no clean way to run only preprocessing + routing decisions without executing the model (needed for external prefetch processors like GAIE).
Migration is always included: No way to opt-out of retry logic when it's not needed (e.g., single-worker scenarios).
These limitations make it difficult to support new use-cases like:
Inference Gateway API approach
Clients who want to handle their own preprocessing / post processing in their modules or rely on Inference Engines. See this [https://github.com/ai-dynamo/enhancements/pull/50/changes#diff-d06f262c7a840682785ad441a3dcaef9ba9f6938bc0e71d1895c89874e7f4086R177]



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
- Preprocessing location: Frontend or Engine
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
- The code **MUST** validate configuration combinations and return clear errors for invalid combinations:
- QueryOnly GAIE stage 1 routing with Engine preprocessing MUST be rejected
- Other invalid combinations SHOULD be identified and rejected with descriptive error messages

# Proposal

We propose replacing the current pipeline construction approach with a configuration-driven, single-function design. The function will take the PipelineConfig as a parameter with the following knobs:
preprocessing: Dynamo | Engine
postprocessing: Dynamo | Engine     
routing: QueryOnly | DirectToKnown | Discover 
migration_enabled: bool    

PipelineConfig:
/// Pipeline configuration - each knob is independent
```rust
pub struct PipelineConfig {
    /// Where does preprocessing happen?
    pub preprocessing: ProcessingLocation,
    
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
    Frontend,
    
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

Optional presets to satisfy the 3 use-cases.

```rust
impl PipelineConfig {
    /// Use-case 1: Preprocessing + routing query only
    /// Returns tokens and worker IDs without model execution
    pub fn query_only() -> Self {
        Self {
            preprocessing: ProcessingLocation::Frontend, // Required
            postprocessing: ProcessingLocation::Frontend, // Ignored for this mode
            routing: RoutingBehavior::QueryOnly,
            migration_enabled: false, // N/A for this mode
        }
    }
    
    /// Use-case 2: Execute with known workers
    /// Workers are pre-selected, route directly to them
    pub fn direct_execution() -> Self {
        Self {
            preprocessing: ProcessingLocation::Frontend,
            postprocessing: ProcessingLocation::Frontend,
            routing: RoutingBehavior::DirectToKnown,
            migration_enabled: true,
        }
    }
    
    /// Use-case 3: Full discovery + execution (current default)
    pub fn full_discovery() -> Self {
        Self::default()
    }
}
```

CLI usage:

```rust
# Use-case 1: Gaie stage 1 only (preprocessing + find workers, no execution)
dynamo-run \
  --routing query-only \
  --in http --out auto

# Use-case 2: Gaie stage 2 Execute with pre-selected workers, engine does preprocessing
dynamo-run \
  --preprocessing engine \
  --postprocessing engine \
  --routing direct \
  --in http --out auto

# Use-case 3: Full pipeline with discovery (current default)
dynamo-run \
  --routing discover \
  --in http --out auto

# Use-case 3 variant: Engine preprocessing, engine postprocessing
dynamo-run \
  --preprocessing engine \
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

Pipeline building function

```rust
/// Components needed to build the pipeline
pub struct PipelineComponents {
    pub card: Arc<ModelDeploymentCard>,
    pub tokenizer: HfTokenizer,
    pub client: Client,
    pub router: Arc<dyn WorkerRouter>,
    pub metrics: Arc<Metrics>,
}

/// Build a pipeline based on configuration
pub async fn build_pipeline<Req, Resp>(
    config: PipelineConfig,
    components: PipelineComponents,
) -> Result<ServiceEngine<SingleIn<Req>, ManyOut<Annotated<Resp>>>>
where
    Req: Data,
    Resp: Data,
{
    // Validate configuration
    validate_config(&config)?;
    
    let frontend = SegmentSource::<SingleIn<Req>, ManyOut<Annotated<Resp>>>::new();
    
    // Preprocessing (if frontend)
    let preprocessor = if config.preprocessing == ProcessingLocation::Frontend {
        let formatter = PromptFormatter::from_mdc(&components.card)?;
        Some(OpenAIPreprocessor::new_with_parts(
            (*components.card).clone(),
            formatter,
            components.tokenizer.clone(),
        )?.into_operator())
    } else {
        None
    };
    
    let backend_op = if config.preprocessing == ProcessingLocation::Frontend {
        Some(Backend::from_tokenizer(components.tokenizer.clone()).into_operator())
    } else {
        None
    };
    
    let migration = if config.migration_enabled 
        && config.routing != RoutingBehavior::QueryOnly 
    {
        Some(Migration::from_mdc(&components.card, components.metrics.clone()).into_operator())
    } else {
        None
    };
    
    let disagg = if config.routing != RoutingBehavior::QueryOnly {
        Some(DisaggOrchestrator::new(
            components.router.clone(),
            components.client.clone(),
        )?.into_operator())
    } else {
        None
    };
    
    // Service backend (based on routing behavior)
    let service_backend = match config.routing {
        RoutingBehavior::QueryOnly => {
            let query_backend = RouteQueryBackend::new(components.router.clone());
            ServiceBackend::from_engine(Arc::new(query_backend))
        }
        RoutingBehavior::DirectToKnown => {
            let transport = DirectTransport::new(components.client.clone()).await?;
            ServiceBackend::from_engine(Arc::new(transport))
        }
        RoutingBehavior::Discover => {
            let transport = RoutedTransport::new(
                components.router.clone(),
                components.client.clone(),
            ).await?;
            ServiceBackend::from_engine(Arc::new(transport))
        }
    };
        
    let mut pipeline = frontend.clone();
    
    // Forward edges
    if let Some(ref pp) = preprocessor {
        pipeline = pipeline.link(pp.forward_edge())?;
    }
    if let Some(ref be) = backend_op {
        pipeline = pipeline.link(be.forward_edge())?;
    }
    if let Some(ref mig) = migration {
        pipeline = pipeline.link(mig.forward_edge())?;
    }
    if let Some(ref dis) = disagg {
        pipeline = pipeline.link(dis.forward_edge())?;
    }
    
    pipeline = pipeline.link(service_backend)?;
    
    // Backward edges
    if let Some(ref dis) = disagg {
        pipeline = pipeline.link(dis.backward_edge())?;
    }
    if let Some(ref mig) = migration {
        pipeline = pipeline.link(mig.backward_edge())?;
    }
    
    if config.postprocessing == ProcessingLocation::Frontend {
        if let Some(ref be) = backend_op {
            pipeline = pipeline.link(be.backward_edge())?;
        }
    }
    if let Some(ref pp) = preprocessor {
        pipeline = pipeline.link(pp.backward_edge())?;
    }
    
    Ok(pipeline.link(frontend)?)
}

fn validate_config(config: &PipelineConfig) -> Result<()> {
    // QueryOnly requires frontend preprocessing (need tokens for routing)
    if config.routing == RoutingBehavior::QueryOnly 
        && config.preprocessing == ProcessingLocation::Engine 
    {
        bail!(
            "QueryOnly routing requires frontend preprocessing to extract tokens for routing. \
             Set --preprocessing=frontend or use --routing=discover or --routing=direct."
        );
    }
    
    Ok(())
}
```


