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
- The pipeline conflates routing selection with request processing.


These limitations make it difficult to support new use-cases like:
- Inference Gateway API approach
- Clients who want to handle their own preprocessing / post processing in their modules or rely on Inference Engines. See [`backend fallback` draft](https://github.com/ai-dynamo/enhancements/pull/50/changes#diff-d06f262c7a840682785ad441a3dcaef9ba9f6938bc0e71d1895c89874e7f4086R177)


# Constraints
- Preprocessing must always be in the frontend because KV cache routing requires tokens to calculate hashes for matching against indexer hashes from KV events.
- Postprocessing can be frontend or engine depending on whether the user wants Rust processing (default) or native framework processing (fallback).

## Goals

* Cleanly separate routing decisions from transport and from disaggregation orchestration

* Support Inference Gateway API 

* Maintain backward compatibility with existing behavior as the default configuration


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

Separate the Concerns:
1. Standalone Router Binding for EPP (Route-Only)
Create a direct binding to the router's select_worker logic that Go can call: `lib/llm/src/kv_router/standalone.rs`
2. Enable it via C bindings
3. Create another pipeline similar to the existing one with direct routing: `async fn build_direct_routed_pipeline<Req, Resp>`
Create a config:

```bash
pub enum PipelineMode {
    /// Full routing - KvRouter selects workers
    FullRouting,
    /// Direct routing - worker IDs provided in request
    DirectFromRequest,
}

// In EngineConfig or LocalModel
pub struct RouterConfig {
    pub mode: RouterMode,
    pub pipeline_mode: PipelineMode,  // NEW
    // ...
}
````
and then u=in the HTTP entrypoint:

```bash
match config.pipeline_mode {
    PipelineMode::FullRouting => {
        build_routed_pipeline_with_preprocessor(...)
    }
    PipelineMode::DirectFromRequest => {
        build_direct_routed_pipeline(...)
    }
}
```
4. Change the kvPushRouter to handle new modes:
```bash
pub enum RouterMode {
    #[default]
    RoundRobin,
    Random,
    Direct(u64),            // Existing: hardcoded worker ID
    KV,                     // Existing: KV-cache aware routing
    DirectFromRequest,      // NEW: Read worker ID from request nvext
}

// In KvPushRouter::generate() or similar

impl KvPushRouter {
    pub async fn generate(&self, request: SingleIn<PreprocessedRequest>) 
        -> Result<ManyOut<...>> 
    {
        match self.router_mode {
            RouterMode::KV => {
                // Existing: compute best worker from KV cache overlap
                let worker_id = self.select_worker(&request).await?;
                self.push_router.direct(request, worker_id).await
            }
            RouterMode::DirectFromNvext => {
                // NEW: read worker ID from request
                let worker_id = request.routing
                    .as_ref()
                    .and_then(|r| r.decode_worker_id)
                    .ok_or_else(|| anyhow!("decode_worker_id required"))?;
                
                self.push_router.direct(request, worker_id).await
            }
            RouterMode::RoundRobin => {
                self.push_router.round_robin(request).await
            }
            RouterMode::Random => {
                self.push_router.random(request).await
            }
            RouterMode::Direct(worker_id) => {
                self.push_router.direct(request, worker_id).await
            }
        }
    }
}
```
