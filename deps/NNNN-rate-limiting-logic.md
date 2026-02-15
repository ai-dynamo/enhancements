# Dynamo Rate Limit for Load Balancer Proposal


**Status**: Draft

**Authors**: [jorgeantonio](https://github.com/jorgeantonio21)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

# Summary

LLM inference providers face a critical challenge in managing concurrent request AI workloads effectively. Without proper rate limiting, peak traffic creates request queues that significantly increase Time To First Token (TTFT), degrading user experience for latency-sensitive applications. Conversely, poorly configured rate limits can reject requests even when the system has available capacity, resulting in lost revenue and suboptimal resource utilization.

This proposal introduces a dynamic rate limiting system for load balancers that uses real-time metrics to intelligently accept or reject requests. The system will be integrated with the `Frontend` service to return `TOO_MANY_REQUESTS` status codes when appropriate, optimizing the balance between user experience, throughput, and revenue.

# Motivation 

## Application Heterogeneity and Latency Requirements

LLM inference serves a highly heterogeneous set of applications with vastly different latency requirements and tolerance levels. This diversity creates complex optimization challenges for inference providers:

**Latency-Critical Applications** require ultra-low Time To First Token (TTFT) and Inter Token Latency (ITL):
- Interactive chat bots
- Real-time search engines
- Agentic workflows
- Vision recognition systems
- Live coding assistants

**Latency-Tolerant Applications** can accept higher TTFT and ITL without degrading user experience:
- Research multi-agentic systems
- Coding agents
- Summarization tasks
- Content generation

The failure to meet latency requirements for critical applications results in immediate customer churn, as users quickly abandon services that feel unresponsive. This creates a high-stakes environment where providers must carefully balance resource allocation.

## The Fundamental Trade-off: Latency vs. Throughput

LLM providers face a fundamental tension between optimizing for latency and maximizing throughput:

**Small Batch Sizes** deliver excellent latency metrics:
- Lower TTFT as requests start processing immediately
- Reduced ITL due to fewer competing requests
- Better user experience for latency-sensitive applications
- **Downside**: Suboptimal GPU utilization and lower revenue per GPU hour

**Large Batch Sizes** maximize hardware efficiency and revenue:
- Higher GPU utilization through parallel processing
- Better amortization of fixed overhead costs
- Increased tokens-per-second throughput
- **Downside**: Higher latency due to resource contention and queuing delays

This trade-off is particularly challenging because the optimal balance point shifts dynamically based on:
- Current request mix (latency-sensitive vs. latency-tolerant)
- Model complexity and context lengths
- Available hardware resources
- Business priorities (user experience vs. revenue optimization)

## The Need for Dynamic Rate Limiting

### Why Static Rate Limits Fail

Traditional static rate limiting approaches are inadequate for LLM inference due to the highly dynamic nature of the workload:

**Fixed Request Limits** ignore system state:
- A fixed limit of 100 concurrent requests might be perfect when processing short, simple queries
- The same limit becomes unsuitable when handling long-context, complex reasoning tasks
- Static limits cannot adapt to varying model sizes, hardware configurations, or request complexity

**Dynamic System State Factors**:
- **Request complexity**: Simple completions vs. complex reasoning tasks have vastly different resource requirements
- **Context length variability**: Requests range from dozens to hundreds of thousands of tokens
- **KV cache efficiency**: Cache hits dramatically reduce prefill overhead
- **Hardware utilization patterns**: GPU memory, compute, and bandwidth usage vary significantly
- **Temporal request patterns**: Traffic spikes, geographic load distribution, and application-specific usage patterns

### The Perfect Scenario vs. Reality

**Ideal State**: Each distributed LLM inference service operates at maximum efficiency:
- Every GPU runs at 100% utilization without idle cycles
- Zero requests wait in queues (no unnecessary TTFT increases)
- Perfect resource allocation across prefill and decode workers
- Optimal batch composition for current hardware and model configuration

**Practical Reality**: Providers must accept suboptimal GPU utilization to maintain service quality:
- Safety margins are required to handle traffic spikes and request variability
- Queue management becomes critical to prevent cascading latency increases
- Resource allocation must be conservative to guarantee SLA compliance
- The challenge is maximizing utilization while staying within these constraints

## Proposed Dynamic Rate Limiting Solution

### Real-Time Metrics Tracking

Our solution continuously monitors eight critical system metrics to make intelligent rate limiting decisions:

**Performance Metrics**:
1. **Time to First Token (TTFT)**: 90th percentile over rolling time windows to capture user-experienced latency
2. **Inter Token Latency (ITL)**: Average over rolling time windows indicating decode worker congestion
3. **Average throughput per request**: Tokens/second efficiency indicating overall system performance

**Capacity Metrics**:
4. **Running batch size**: Current number of requests being actively processed
5. **Waiting queue length**: Number of requests awaiting processing resources
6. **Average queue waiting time**: 90th percentile wait times indicating system saturation

**Resource Utilization Metrics**:
7. **KV Cache utilization rate**: Memory efficiency indicating capacity for new requests
8. **KV Cache hit rate**: Percentage of requests benefiting from cached computations

### Intelligent Decision Making for Complex Scenarios

The dynamic rate limiting system handles sophisticated scenarios that static rate limits cannot address:

**Scenario 1: High TTFT, Low ITL with Cache Opportunities**
- **Situation**: Prefill workers are overloaded (high TTFT) but decode workers have capacity (low ITL)
- **Traditional approach**: Reject all new requests due to high TTFT
- **Smart approach**: Accept requests with high KV cache hit rates, as prefill overhead will be minimal due to pre-computed cache entries
- **Business impact**: Maintains revenue from easily-served requests while protecting latency-sensitive applications

**Scenario 2: Low TTFT, High ITL with Long Prefill Requests**
- **Situation**: Prefill workers have capacity but decode workers are congested
- **Traditional approach**: Accept requests based only on current TTFT
- **Smart approach**: Accept requests with low cache hit rates if prefill phase duration allows decode congestion to clear
- **Business impact**: Maximizes hardware utilization by scheduling work optimally across processing phases

**Scenario 3: Latency-Tolerant Request Handling**
- **Situation**: New request is marked as non-latency-sensitive
- **Traditional approach**: Treat all requests identically
- **Smart approach**: Queue latency-tolerant requests during high-load periods, serving them during low-load windows
- **Business impact**: Enables tiered pricing models and prevents revenue loss from deferrals

### Economic and Strategic Benefits

This dynamic approach enables several strategic advantages:

**Revenue Optimization**: By accepting more requests when the system can handle them efficiently, providers increase revenue without degrading user experience.

**Customer Segmentation**: Different SLA tiers can be offered based on latency requirements, enabling more sophisticated pricing models.

**Competitive Advantage**: Better resource utilization allows for more competitive pricing while maintaining superior user experience.

**Scalability**: The system adapts automatically to new hardware, models, and usage patterns without manual reconfiguration.

# Goals 

## Core Infrastructure Goals

* **Real-time metrics tracking system**

Implement a comprehensive metrics collection system that tracks the eight critical metrics in real time, independently of the underlying inference backend (vLLM, SGLang, TensorRT, etc):
- Performance metrics: TTFT, ITL, throughput per request
- Capacity metrics: running batch size, waiting queue length, average queue waiting time  
- Resource utilization metrics: KV cache utilization rate, KV cache hit rate

* **Dynamic acceptance/rejection logic**

Implements dynamic rate limiting with intelligent decision-making that considers current system state, request characteristics, and resource availability through configurable thresholds for each metric.

## Intelligence and Decision Making Goals

* **Context-aware request handling**

Implement sophisticated scenarios that optimize resource utilization:
- Accept high KV cache hit rate requests when prefill workers are overloaded but decode workers have capacity
- Accept low cache hit rate requests when prefill capacity is available and prefill duration allows decode congestion to clear
- Handle latency-tolerant requests through intelligent queuing during high-load periods

* **Predictive load management**

Enable the system to make forward-looking decisions based on request characteristics, current system state, and expected resource availability patterns.

## Integration and Usability Goals

* **Frontend service integration**

Integrate rate limiting intelligence with the `Frontend` HTTP service to:
- Provide real-time system state awareness
- Return appropriate `TOO_MANY_REQUESTS` status codes when requests should be rejected
- Maintain low-latency decision making for request acceptance/rejection

* **Configurable and optional rate limiting**

Design the system to be:
- Fully configurable with provider-specific tuning capabilities
- Optional to enable/disable based on provider needs
- Adaptable to different hardware configurations, model sizes, and business requirements

## System Quality Goals

* **High availability and reliability**

Ensure the rate limiting system itself does not become a point of failure:
- Fast decision-making with minimal latency overhead
- Graceful degradation when metrics are temporarily unavailable
- Conservative fallback behavior to protect system stability

* **Observability and debugging**

Provide comprehensive visibility into:
- Rate limiting decisions and their rationale
- System performance impact of rate limiting logic
- Metrics for tuning and optimizing rate limiting parameters
- Clear audit trails for accepted/rejected requests


## Future Extensibility Goals (For future enhancements)

* **Priority queue infrastructure (Future)**

Establish the foundation for implementing priority-based request handling where:
- High priority requests are served first
- Low priority requests can be deferred to periods of lower system load
- Dynamic economic models can be established around request prioritization and tokenomics

* **Multi-cluster coordination (Future)**

Design the system to support future expansion to:
- Cross-cluster load balancing and re-routing capabilities
- Global optimization across multiple inference clusters
- Intelligent request distribution based on cluster-specific capabilities and current load

# Non Goals

## Business and Economic Concerns

This proposal focuses purely on the technical implementation of dynamic rate limiting. All business-related aspects are explicitly out of scope:

* **Pricing and billing strategies** - How providers set prices, bill customers, or structure payment models
* **Customer contracts and SLA definitions** - The specific terms, guarantees, or legal commitments made to customers  
* **Tokenomics and economic modeling** - Business logic around token pricing, usage credits, or economic incentives
* **Revenue optimization strategies** - Business decisions about profit margins, competitive pricing, or market positioning
* **Customer segmentation and tier definitions** - How providers categorize customers or define service tiers
* **Financial reporting and analytics** - Revenue tracking, cost analysis, or business intelligence

## Infrastructure and Platform Concerns

* **Hardware provisioning and scaling** - Decisions about GPU clusters, server capacity, or infrastructure scaling
* **Cross-cluster load balancing and re-routing** - Logic for distributing requests across multiple inference clusters or data centers

## Implementation and Integration Details

* **Specific inference backend implementations** - Internal rate limiting or lack of thereof workings of vLLM, SGLang, TensorRT, or other inference engines
* **Authentication and authorization systems** - User identity management, API keys, or access control (beyond basic request validation)
* **Comprehensive monitoring and alerting platforms** - Full observability stacks, dashboards, or alerting systems (basic metrics collection is in scope)
* **Configuration management interfaces** - UI/UX for configuring rate limiting parameters or administrative tools
* **Performance testing and benchmarking frameworks** - Load testing tools, performance validation, or capacity planning tools

## Security and Compliance

* **Security threat modeling and mitigation** - Protection against DDoS attacks, security vulnerabilities, or threat analysis
* **Data privacy and compliance requirements** - GDPR, HIPAA, or other regulatory compliance beyond basic request handling
* **Audit logging and compliance reporting** - Detailed audit trails for regulatory purposes (basic request logging is in scope)

## Advanced Features and Optimizations

* **AI-powered request prediction and optimization** - Machine learning models to predict optimal rate limiting parameters
* **Dynamic model switching and routing** - Intelligent routing between different models based on request characteristics
* **Request transformation and optimization** - Modifying or optimizing requests before processing

## Scope Clarification

**What IS in scope**: The technical implementation of a dynamic rate limiting service that makes accept/reject decisions based on real-time system metrics, integrates with the Frontend service, and provides basic observability for its own operations.

**What IS NOT in scope**: Any business logic, economic decisions, complex infrastructure management, comprehensive platform features, or advanced optimizations beyond the core rate limiting functionality.

# Proposal 

## Overview

We propose implementing a **Dynamic Rate Limiting Service** that intelligently manages LLM inference request acceptance based on real-time system state. Unlike traditional static rate limiters, this system continuously monitors system performance and resource utilization to make context-aware decisions that optimize the balance between user experience, throughput, and resource efficiency.

## Core Architecture

### Rate Limiting Decision Engine

The central component is a **Rate Limiting Decision Engine** that processes real-time metrics to determine whether to accept or reject incoming requests. The engine operates on a configurable rule-based system that evaluates multiple system dimensions simultaneously:

**Primary Decision Factors:**
1. **Performance Metrics**: Real-time TTFT (90th percentile), ITL (90th percentile), and average throughput per request
2. **Capacity Metrics**: Real-time running batch size, waiting queue length, and average queue waiting time (90th percentile)  
3. **Resource Utilization**: Real-time KV cache utilization rate and hit rate

**Decision Logic:**
- **Accept**: When all metrics are within acceptable thresholds OR when intelligent exceptions apply
- **Reject**: When critical thresholds are exceeded AND no exception scenarios are met
- **Queue**: For latency-tolerant requests during borderline system states

### Intelligent Exception Handling

The system implements sophisticated exception logic that goes beyond simple threshold checking:

**Scenario-Based Acceptance:**
- **High Cache Hit Scenarios**: Accept requests with e.g. >80% KV cache hit rate even when TTFT is elevated, if decode workers have capacity
- **Prefill Capacity Scenarios**: Accept low cache hit requests when prefill workers are available and processing time allows decode congestion to clear
- **Latency-Tolerant Requests**: Queue non-critical requests during high-load periods for processing during low-load windows

**Dynamic Threshold Adjustment:**
- Thresholds automatically adjust based on historical performance patterns
- Separate threshold sets for different request types (latency-critical vs. latency-tolerant)
- Time-of-day, geographical location, and load pattern awareness for optimal decision-making

## System Integration Architecture

### Frontend Service Integration

The rate limiting service integrates seamlessly with the existing `Frontend` HTTP service:

**Request Flow:**
1. **Pre-processing**: Frontend forwards request metadata (estimated complexity, cache hit prediction, latency sensitivity)
2. **Decision Query**: Rate limiter evaluates current system state against request characteristics  
3. **Response Handling**: Frontend receives accept/reject decision with optional queuing information
4. **Client Response**: Appropriate HTTP status codes (`200 OK`, `429 TOO_MANY_REQUESTS`, `202 ACCEPTED` for queued requests)

### Metrics Collection Framework

**Real-Time Data Pipeline:**
- Unified metrics collection from all inference backends (vLLM, SGLang, TensorRT)
- Streaming metrics with configurable aggregation windows (1s, 5s, 30s, 5m)
- Separate metrics tracking for different model types and hardware configurations

**Data Sources:**
- **Inference Engines**: TTFT, ITL, throughput, batch sizes, queue states
- **Resource Monitors**: GPU utilization, memory usage, KV cache statistics
- **Request Trackers**: Request complexity estimation, cache hit predictions

## Configuration and Adaptability

### Configurable Threshold System

**Multi-Dimensional Thresholds:**
```yaml
rate_limiting:
  performance_thresholds:
    ttft_p90_ms: 1000
    itl_avg_ms: 50
    min_throughput_tokens_per_sec: 20
  capacity_thresholds:
    max_running_batch_size: 256
    max_queue_length: 50
    max_queue_wait_p90_ms: 5000
  resource_thresholds:
    max_kv_cache_utilization: 0.85
    min_kv_cache_hit_rate_for_exceptions: 0.8
```

**Adaptive Configuration:**
- Provider-specific tuning capabilities for different business models
- Hardware-specific optimization (different GPU types, memory configurations)
- A/B testing support for threshold optimization
- Runtime configuration updates without service restart

### Extensibility Framework (Out of scope for this proposal)

**Plugin Architecture:**
- Custom decision algorithms for specific use cases
- Provider-specific exception logic implementations
- Integration hooks for external monitoring and alerting systems

**Future-Ready Design:**
- Foundation for priority queue implementation
- Multi-cluster coordination preparation
- Machine learning integration points for predictive decision-making


# Implementation Details

## Implementation Phases

### Phase 1: Core Infrastructure
- Metrics collection system implementation
- Basic decision engine with configurable thresholds
- Frontend service integration
- Essential observability and debugging capabilities

### Phase 2: Intelligence Layer
- Exception scenario implementation
- Advanced decision algorithms, based on KV cache hit rates, etc
- Predictive load management capabilities
- Comprehensive configuration management

### Phase 3: Advanced Features
- Priority queue foundation
- Multi-cluster coordination preparation
- ML-powered threshold optimization
- Advanced analytics and reporting

## Key Design Principles

**Performance First:**
- Sub-millisecond decision latency requirement
- Non-blocking architecture to prevent request processing delays
- Efficient real-time metrics aggregation and caching

**Reliability and Safety:**
- Conservative fallback behavior when metrics are unavailable
- Circuit breaker patterns to prevent cascade failures  
- Comprehensive error handling and graceful degradation

**Observability:**
- Detailed logging of all rate limiting decisions
- Real-time dashboards for system state monitoring
- Comprehensive metrics for tuning and optimization

**Flexibility:**
- Fully configurable without code changes
- Optional deployment - can be disabled entirely
- Backward compatibility with existing static rate limiting approaches

## Expected Outcomes

**Immediate Benefits:**
- Improvement in GPU utilization while maintaining SLA compliance
- Reduced unnecessary request rejections during available capacity periods
- Better handling of traffic spikes and load variability

**Strategic Advantages:**
- Foundation for tiered service offerings and dynamic pricing models
- Competitive advantage through superior resource efficiency
- Scalable architecture that adapts to new hardware, new models and new LLM inference backends, automatically

**Operational Improvements:**
- Reduced need for manual capacity planning and threshold tuning
- Better predictability in system behavior during varying load conditions
- Simplified operations through intelligent automation

## System Architecture Overview 

The dynamic rate limiting system consists of four core components that integrate with the existing infrastructure:

1. **Enhanced Metrics Collection Service** - Extends existing `PrometheusMetricsCollector` (see `components/metrics/src/lib.rs`)
2. **Rate Limiting Decision Engine** - New service for intelligent accept/reject decisions
3. **Request Context Analyzer** - Analyzes incoming requests for cache hit prediction and complexity estimation
4. **Frontend Integration Layer** - Connects rate limiting decisions to HTTP request handling

We provide a very high level overview of the system's architecture, and the implementation details of the core components.

**WARNING:** The Rust code snippets below are not complete and are not guaranteed to be compatible with the current Dynamo codebase. They are provided for illustrative purposes only.

## Component 1: Enhanced Metrics Collection Service

### Extending Existing Infrastructure

Build upon the current `PrometheusMetricsCollector` by extending the metrics collection to include rate limiting specific data:

```rs
/// Enhanced metrics collection for rate limiting
pub struct EnhancedPrometheusMetrics {
    // Existing metrics (preserved)
    kv_blocks_active: prometheus::GaugeVec,
    kv_blocks_total: prometheus::GaugeVec,
    requests_active: prometheus::GaugeVec,
    requests_total: prometheus::GaugeVec,
    load_avg: prometheus::GaugeVec,
    load_std: prometheus::GaugeVec,
    kv_hit_rate_percent: prometheus::GaugeVec,
    
    // New rate limiting metrics (available from the `HTTP` service)
    ttft_histogram: prometheus::HistogramVec,
    itl_histogram: prometheus::HistogramVec,
    throughput_gauge: prometheus::GaugeVec,
    queue_wait_time_histogram: prometheus::HistogramVec,
    batch_size_gauge: prometheus::GaugeVec,
    queue_length_gauge: prometheus::GaugeVec,
    
    // Decision tracking metrics
    rate_limit_decisions: prometheus::CounterVec, // accept/reject/queue
    exception_scenarios: prometheus::CounterVec,  // tracks which scenarios triggered
    decision_latency: prometheus::HistogramVec,   // time to make rate limiting decision
}
```

### Real-Time Metrics Aggregation

Implement efficient metrics aggregation with configurable time windows:

```rs
pub struct MetricsAggregator {
    // Rolling window aggregators for different time periods
    ttft_aggregator: RollingPercentileAggregator,
    itl_aggregator: RollingAverageAggregator,
    throughput_aggregator: RollingAverageAggregator,
    queue_time_aggregator: RollingPercentileAggregator,
    
    // Configurable window sizes
    window_configs: WindowConfigurations,
}

#[derive(Clone, Debug)]
pub struct WindowConfigurations {
    pub short_window: Duration,    // 1s - for immediate decisions
    pub medium_window: Duration,   // 30s - for trend analysis
    pub long_window: Duration,     // 5m - for pattern recognition
}
```

## Component 2: Rate Limiting Decision Engine

### Core Decision Logic Implementation

```rs
pub struct RateLimitingEngine {
    config: RateLimitingConfig,
    metrics_client: Arc<MetricsClient>,
    cache_analyzer: Arc<CacheHitPredictor>,
    decision_cache: Arc<RwLock<DecisionCache>>,
}

#[derive(Debug, Clone)]
pub struct RateLimitingConfig {
    pub performance_thresholds: PerformanceThresholds,
    pub capacity_thresholds: CapacityThresholds,
    pub resource_thresholds: ResourceThresholds,
    pub exception_rules: ExceptionRules,
}

#[derive(Debug, Clone)]
pub struct PerformanceThresholds {
    pub ttft_p90_ms_max: f64,
    pub itl_avg_ms_max: f64,
    pub throughput_tokens_per_sec_min: f64,
}

impl RateLimitingEngine {
    pub async fn make_decision(&self, request: &IncomingRequest) -> RateLimitingDecision {
        // 1. Fetch current system metrics
        let metrics = self.metrics_client.get_current_metrics().await?;
        
        // 2. Analyze request characteristics
        let request_context = self.analyze_request(request).await?;
        
        // 3. Apply decision logic
        let decision = self.evaluate_request(&metrics, &request_context).await?;
        
        // 4. Log decision for observability
        self.log_decision(&decision, &metrics, &request_context).await?;
        
        decision
    }
    
    async fn evaluate_request(
        &self,
        metrics: &SystemMetrics,
        context: &RequestContext,
    ) -> Result<RateLimitingDecision> {
        // Check basic thresholds first
        if self.basic_thresholds_exceeded(metrics) {
            // Check for exception scenarios
            if let Some(exception) = self.check_exceptions(metrics, context).await? {
                return Ok(RateLimitingDecision::Accept(exception));
            }
            
            // Determine if request should be queued or rejected
            if context.latency_tolerance == LatencyTolerance::Tolerant {
                return Ok(RateLimitingDecision::Queue(QueueInfo::default()));
            }
            
            return Ok(RateLimitingDecision::Reject(RejectionReason::SystemOverloaded));
        }
        
        Ok(RateLimitingDecision::Accept(AcceptanceReason::WithinThresholds))
    }
}
```

### Exception Scenario Implementation

```rs
impl RateLimitingEngine {
    async fn check_exceptions(
        &self,
        metrics: &SystemMetrics,
        context: &RequestContext,
    ) -> Result<Option<AcceptanceReason>> {
        // Scenario 1: High TTFT, Low ITL, High Cache Hit Rate
        if metrics.ttft_p90_ms > self.config.performance_thresholds.ttft_p90_ms_max
            && metrics.itl_avg_ms < self.config.performance_thresholds.itl_avg_ms_max * 0.7
            && context.predicted_cache_hit_rate > self.config.exception_rules.min_cache_hit_rate_for_ttft_exception
        {
            return Ok(Some(AcceptanceReason::HighCacheHitException));
        }
        
        // Scenario 2: High TTFT, High ITL, Long Prefill Expected
        if metrics.ttft_p90_ms > self.config.performance_thresholds.ttft_p90_ms_max * 0.95
            && metrics.itl_avg_ms > self.config.performance_thresholds.itl_avg_ms_max
            && context.estimated_prefill_duration_ms > self.config.exception_rules.min_prefill_duration_for_itl_exception
        {
            return Ok(Some(AcceptanceReason::LongPrefillException));
        }
        
        // Scenario 3: Resource availability despite metrics
        if metrics.kv_cache_utilization < self.config.resource_thresholds.max_kv_cache_utilization * 0.6
            && metrics.batch_size < self.config.capacity_thresholds.max_running_batch_size * 0.6
        {
            return Ok(Some(AcceptanceReason::ResourceAvailabilityException));
        }
        
        Ok(None)
    }
}
```

## Component 3: Request Context Analyzer

### Cache Hit Prediction Integration

```rs
pub struct RequestContextAnalyzer {
    cache_manager: Arc<KVCacheManager>,
    complexity_estimator: Arc<ComplexityEstimator>,
}

impl RequestContextAnalyzer {
    pub async fn analyze_request(&self, request: &IncomingRequest) -> Result<RequestContext> {
        // Get cache hit prediction from existing KV cache manager
        let cache_prediction = self.cache_manager
            .predict_cache_hit_rate(&request.prompt, &request.context)
            .await?;
        
        // Estimate request complexity
        let complexity = self.complexity_estimator
            .estimate_complexity(&request)
            .await?;
        
        Ok(RequestContext {
            predicted_cache_hit_rate: cache_prediction.hit_rate,
            estimated_prefill_duration_ms: complexity.estimated_prefill_ms,
            estimated_total_tokens: complexity.estimated_output_tokens,
            latency_tolerance: request.latency_tolerance.unwrap_or(LatencyTolerance::Critical),
            request_priority: request.priority.unwrap_or(RequestPriority::Normal),
        })
    }
}

#[derive(Debug, Clone)]
pub struct RequestContext {
    pub predicted_cache_hit_rate: f32,
    pub estimated_prefill_duration_ms: u64,
    pub estimated_total_tokens: u64,
    pub latency_tolerance: LatencyTolerance,
    pub request_priority: RequestPriority,
}
```

## Component 4: Frontend Integration Layer

### HTTP Service Integration

```rs
// In the Frontend service
pub struct EnhancedFrontendService {
    rate_limiter: Arc<RateLimitingEngine>,
    request_analyzer: Arc<RequestContextAnalyzer>,
    // ... existing fields
}

impl EnhancedFrontendService {
    pub async fn handle_request(&self, request: HttpRequest) -> Result<HttpResponse> {
        // 1. Parse and validate request
        let parsed_request = self.parse_request(request).await?;
        
        // 2. Check rate limiting
        let rate_limit_decision = self.rate_limiter
            .make_decision(&parsed_request)
            .await?;
        
        match rate_limit_decision {
            RateLimitingDecision::Accept(_) => {
                // Process request normally
                self.process_request(parsed_request).await
            }
            RateLimitingDecision::Reject(reason) => {
                // Return 429 Too Many Requests
                Ok(HttpResponse::TooManyRequests()
                    .json(RateLimitErrorResponse {
                        error: "rate_limit_exceeded".to_string(),
                        message: format!("Request rejected: {:?}", reason),
                        retry_after_seconds: self.calculate_retry_after().await?,
                    }))
            }
            RateLimitingDecision::Queue(queue_info) => {
                // Return 202 Accepted with queue information
                Ok(HttpResponse::Accepted()
                    .json(QueuedRequestResponse {
                        status: "queued".to_string(),
                        estimated_wait_time_seconds: queue_info.estimated_wait_time_seconds,
                        queue_position: queue_info.position,
                        request_id: queue_info.request_id,
                    }))
            }
        }
    }
}
```

### Decision Response Types

```rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum RateLimitingDecision {
    Accept(AcceptanceReason),
    Reject(RejectionReason),
    Queue(QueueInfo),
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum AcceptanceReason {
    WithinThresholds,
    HighCacheHitException,
    LongPrefillException,
    ResourceAvailabilityException,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum RejectionReason {
    SystemOverloaded,
    TTFTThresholdExceeded,
    ITLThresholdExceeded,
    QueueCapacityExceeded,
    ResourceConstraints,
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QueueInfo {
    pub request_id: String,
    pub position: u32,
    pub estimated_wait_time_seconds: u64,
}
```

## Configuration and Deployment

### Configuration Management

```rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RateLimitingConfiguration {
    pub enabled: bool,
    pub decision_timeout_ms: u64,
    pub metrics_refresh_interval_ms: u64,
    
    pub thresholds: ThresholdConfiguration,
    pub exceptions: ExceptionConfiguration,
    pub queuing: QueueConfiguration,
    
    // Provider-specific overrides
    pub provider_overrides: HashMap<String, ProviderSpecificConfig>,
}

// Load from YAML configuration file
impl RateLimitingConfiguration {
    pub fn from_file(path: &str) -> Result<Self> {
        let content = std::fs::read_to_string(path)?;
        Ok(serde_yaml::from_str(&content)?)
    }
    
    pub fn validate(&self) -> Result<()> {
        // Validate configuration consistency
        // Ensure thresholds are reasonable
        // Check for conflicting settings
        Ok(())
    }
}
```

### Service Initialization

```rs
pub async fn initialize_rate_limiting_service(
    config: RateLimitingConfiguration,
    metrics_client: Arc<MetricsClient>,
) -> Result<Arc<RateLimitingEngine>> {
    // Validate configuration
    config.validate()?;
    
    // Initialize components
    let cache_analyzer = Arc::new(CacheHitPredictor::new().await?);
    let complexity_estimator = Arc::new(ComplexityEstimator::new().await?);
    let request_analyzer = Arc::new(RequestContextAnalyzer::new(
        cache_analyzer,
        complexity_estimator,
    ));
    
    // Create rate limiting engine
    let engine = Arc::new(RateLimitingEngine::new(
        config,
        metrics_client,
        request_analyzer,
    ).await?);
    
    // Start background tasks for metrics aggregation
    engine.start_metrics_aggregation_task().await?;
    
    Ok(engine)
}
```
