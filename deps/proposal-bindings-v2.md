# Current state

Bindings modules:
 - dynamo.runtime
 - dynamo.llm
 - dynamo.nixl_connect
 - dynamo.logits_processing

About 65 classes and functions, with no obvious organization, mainly in modules `dynamo.llm` and `dynamo.runtime`.

  | Category | Classes/Functions | Used By |
  |----------|------------------|---------|
  | Core Runtime | DistributedRuntime, Namespace, Component, Endpoint, Client, CancellationToken, Context | All |
  | Model Registration | register_llm, unregister_llm, fetch_llm, ModelType, ModelInput, ModelDeploymentCard, ModelRuntimeConfig | Backends |
  | Router Config | RouterMode, RouterConfig, KvRouterConfig | Frontend |
  | KV Indexing | KvIndexer, ApproxKvIndexer, RadixTree, OverlapScores | Router |
  | KV Publishing | KvEventPublisher, ZmqKvEventPublisher, ZmqKvEventPublisherConfig, ZmqKvEventListener | Backends |
  | KV Router | KvPushRouter, KvPushRouterStream | Router |
  | Worker Metrics | WorkerMetricsPublisher, ForwardPassMetrics, WorkerStats, KvStats, SpecDecodeStats | Backends |
  | HTTP/gRPC | HttpService, HttpAsyncEngine, KserveGrpcService, PythonAsyncEngine | Frontend |
  | Preprocessor | OAIChatPreprocessor, MediaDecoder, MediaFetcher, Backend | Frontend |
  | Entrypoint | EntrypointArgs, EngineConfig, EngineType, make_engine, run_input | Frontend |
  | Prometheus | RuntimeMetrics, Counter, IntCounter, CounterVec, IntCounterVec, Gauge, IntGauge, GaugeVec, IntGaugeVec, Histogram | All |
  | Block Mgmt | Layer, Block, BlockList, BlockManager, KvbmCacheManager, KvbmRequest | KVBM |
  | Planner | VirtualConnectorCoordinator, VirtualConnectorClient, PlannerDecision | Planner |
  | LoRA | LoRADownloader, lora_name_to_id | Backends |
  | Misc | KvRecorder, compute_block_hash_for_seq_py, log_message | Various |

# Goals:

1. Minimize the public interface, so we have less to support and more flexibility to evolve.
2. Organize the public interface to make it more intuitive.

# Proposal

## Remove deprecated, unused or should-not-be-used features.

Progress: https://github.com/ai-dynamo/dynamo/pull/5412 and https://github.com/ai-dynamo/dynamo/pull/5458
Action: Identify more removals. This is challenging, we don't know what people are using.

## Move internal bindings (meaning used by `components/` but not intended for public use otherwise) under new `_internal` module.

There should not be exposed: Namespace, Component, CancellationToken, Context, ModelDeploymentCard, ModelRuntimeConfig.

What about Endpoint? Can we merge creating an endpoint with serving it?

ACTION: Change public API to not need access to these, then move them to new `dynamo._internal` module.

## KV router

The KV routing system has 15+ classes with confusing overlaps:
• 3 indexer variants (KvIndexer, ApproxKvIndexer, RadixTree)
• 4 publisher variants (KvEventPublisher, ZmqKvEventPublisher, etc.)
• Stats classes (ForwardPassMetrics, WorkerStats, KvStats, SpecDecodeStats)

ACTION: Needs design from KV router team. I suspect it evolved organically, now is the time to tidy.

Opus suggests: Create unified KvRouter and KvPublisher that internally use:
• KvIndexer or ApproxKvIndexer based on mode
• ZmqKvEventPublisher or KvEventPublisher based on source

## Prometheus metrics

They allow Python users to use Prometheus metrics with Dynamo labels attached. They are used instead of the official Prometheus library, and team consensus appears to be that this is necessary.

ACTIONS:
 - Every metric type has an Int variant that may be unnecessary for Python (which handles int<->float seamlessly).
 - Can we have a single entry point? `Metrics.counter(..)`, `Metrics.gauge(..)`, so a single type?
 - If more than one type, put it under new `dynamo.metrics` module.

## Multi-modal

Evolving, and will not be bound by v1 promise, but:

ACTIONS: Design the interface. Don't let it just happen. Ideally a single type that interfaces to all things multi-modal.

## Misc tidy ups

. `register_llm` has a lot of required params, simplify.
. Look at `Client` (router) interface. Simplify, possibly rename. Merge with KV Router?

