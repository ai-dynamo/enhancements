# Test Guideline for Dynamo

## Summary

This document defines the comprehensive testing strategy for the Dynamo distributed inference framework. It establishes testing standards, organizational patterns, and best practices for validating a complex multi-language system with Rust core components, Python bindings, and multiple backend integrations.

## Motivation

Currently the Dynamo project has a number of different test strategies and implementations which can be confusing in particular with respect to what tests run, when, and where. There is not a guide for developers, QA or operations teams as to the general theory and basic set of tools, tests, or when and how they should be run. We need a set of guidelines and overarching structure to help form the basis for test plans.

## Requirements

1. Tests MUST be able to run locally as well as in CI. This is subject to appropriate hardware being available in the environment
2. Tests MUST be deterministic. Tests deemed "flaky" will be removed.
3. Tests SHOULD be written before beginning development of a new feature.

## Test Characteristics
- **Fast**: Unit tests < 10ms, Integration tests < 1s
- **Reliable**: No flaky tests, deterministic outcomes
- **Isolated**: Tests don't affect each other
- **Clear**: Test intent obvious from name and structure
- **Maintainable**: Tests updated with code changes

## Code Coverage Requirements
- **Rust**: Minimum 80% line coverage, 90% for critical paths
- **Python**: Minimum 85% line coverage, 95% for public APIs

---

## Testing Directory Structure
```

dynamo/
├── lib/
│   ├── runtime/
│   │   ├── src/
│   │   │   └── lib.rs          # Rust code + unit tests inside
│   │   └── tests/              # Optional Rust integration tests specific to runtime
│   |   └── benches/  
│   ├── llm/
│   │   └── src/
│   │       └── lib.rs          # Unit tests here
│   │   └── tests/              # Optional Rust integration tests specific to runtime
│   |   └── benches/  
│   └── ...
├── components/
│   ├── planner/
│   │   └── tests/              # Python unit tests for planner module
│   ├── backend/
│   │   └── tests/              # Python unit tests for backend module
│   └── ...
├── tests/                     # Top-level integration tests (Rust and Python)
    ├── integration/                   # Python integration tests, organized by component
    │   ├── planner/
    │   ├── router/
    │   └── common/
    ├── e2e/                          # Python end-to-end tests
    ├── benahmark/
    └── fault_tolerance/

```

---

## Test Categories and Levels

### 1. Unit Tests

#### Rust Unit Tests
**Location**: Inline with source code using `#[cfg(test)]`
**Purpose**: Test individual functions, structs, and modules in isolation
**Characteristics**: Fast (<1ms), deterministic, no I/O, no network

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use tokio_test;

    #[test]
    fn test_sync_function() {
        let result = my_function(input);
        assert_eq!(result, expected);
    }

    #[tokio::test]
    async fn test_async_function() {
        let result = async_function().await;
        assert!(result.is_ok());
    }

    #[test]
    fn test_error_conditions() {
        let result = function_with_errors(invalid_input);
        assert!(matches!(result, Err(ErrorType::InvalidInput)));
    }
}
```

#### Python Unit Tests
**Location**: `component_module/tests/`
**Purpose**: Test individual Python functions and classes
**Characteristics**: Fast (<10ms), isolated, mocked dependencies

```python
import pytest
from unittest.mock import Mock, patch

@pytest.mark.unit
def test_function_behavior():
    """Test specific function behavior in isolation"""
    result = my_function(test_input)
    assert result == expected_output

@pytest.mark.unit
@patch('external_dependency')
def test_with_mocked_dependency(mock_dep):
    """Test with external dependencies mocked"""
    mock_dep.return_value = mock_response
    result = function_using_dependency()
    assert result.is_valid()

@pytest.mark.unit
@pytest.mark.parametrize("input,expected", [
    ("valid_input", True),
    ("invalid_input", False),
])
def test_input_validation(input, expected):
    """Parameterized test for various inputs"""
    assert validate_input(input) == expected
```

### 2. Integration Tests

#### Rust Integration Tests
**Location**: `tests/` directory in each crate
**Purpose**: Test public APIs and component interactions
**Characteristics**: Medium speed (<100ms), realistic data, limited scope

```rust
// tests/component_integration.rs
use dynamo_runtime::Runtime;
use dynamo_llm::LLMEngine;

#[tokio::test]
async fn test_runtime_llm_integration() {
    let runtime = Runtime::new().await.unwrap();
    let engine = LLMEngine::new(&runtime).await.unwrap();
    
    let result = engine.process_request(test_request()).await;
    assert!(result.is_ok());
}

#[tokio::test]
async fn test_error_propagation() {
    let runtime = Runtime::new().await.unwrap();
    let engine = LLMEngine::new(&runtime).await.unwrap();
    
    let result = engine.process_request(invalid_request()).await;
    assert!(matches!(result, Err(LLMError::InvalidRequest(_))));
}
```

#### Python Integration Tests
**Location**: `tests/integration/`
**Purpose**: Test component interactions and Python-Rust bindings
**Characteristics**: Medium speed (<1s), real components, controlled environment

```python
@pytest.mark.integration
@pytest.mark.asyncio
async def test_python_rust_integration():
    """Test Python-Rust binding integration"""
    runtime = await Runtime.create()
    context = runtime.create_context()
    
    result = await context.process(test_data)
    assert result.status == "success"
    
    await runtime.shutdown()

@pytest.mark.integration
def test_multi_component_workflow():
    """Test workflow across multiple components"""
    planner = Planner()
    frontend = Frontend()
    backend = Backend("vllm")
    
    plan = planner.create_plan(request)
    processed = frontend.process(plan)
    result = backend.execute(processed)
    
    assert result.is_valid()
```

### 3. End-to-End Tests

#### Python: System E2E Tests
**Location**: `tests/e2e/`
**Purpose**: Validate complete system behavior
**Characteristics**: Slow (>5s), realistic scenarios, full system

```python
@pytest.mark.e2e
@pytest.mark.slow
@pytest.mark.gpu_required
async def test_complete_inference_workflow():
    """Test complete inference from request to response"""
    # Start full system
    system = await DynamoSystem.start(config)
    
    # Send realistic request
    request = InferenceRequest(
        model="test-model",
        prompt="Test prompt",
        max_tokens=100
    )
    
    response = await system.process_request(request)
    
    assert response.status == "completed"
    assert len(response.tokens) > 0
    assert response.latency_ms < MAX_ACCEPTABLE_LATENCY
    
    await system.shutdown()

@pytest.mark.e2e
@pytest.mark.multi_gpu
def test_distributed_inference():
    """Test inference across multiple GPUs"""
    system = DynamoSystem.start_distributed(gpu_count=2)
    
    # Test load balancing
    requests = [create_test_request() for _ in range(10)]
    responses = await system.process_batch(requests)
    
    assert all(r.status == "completed" for r in responses)
    assert_gpu_utilization_balanced()
```

### 4. Performance Tests

#### Rust Benchmarks
**Location**: `benches/` in each crate
**Tool**: Criterion.rs
**Purpose**: Track performance regressions

```rust
// benches/tokenizer_bench.rs
use criterion::{black_box, criterion_group, criterion_main, Criterion};
use dynamo_tokens::Tokenizer;

fn tokenizer_benchmark(c: &mut Criterion) {
    let tokenizer = Tokenizer::new("test-model").unwrap();
    let text = "Sample text for tokenization";
    
    c.bench_function("tokenize", |b| {
        b.iter(|| tokenizer.encode(black_box(text)))
    });
    
    c.bench_function("decode", |b| {
        let tokens = tokenizer.encode(text).unwrap();
        b.iter(|| tokenizer.decode(black_box(&tokens)))
    });
}

criterion_group!(benches, tokenizer_benchmark);
criterion_main!(benches);
```

#### Python Performance Tests
**Location**: `tests/performance/`
**Purpose**: Validate system performance characteristics

```python
@pytest.mark.benchmark
@pytest.mark.performance
def test_throughput_benchmark(benchmark):
    """Benchmark system throughput"""
    system = setup_test_system()
    
    def process_batch():
        requests = [create_test_request() for _ in range(100)]
        return system.process_batch_sync(requests)
    
    result = benchmark(process_batch)
    
    # Assert performance requirements
    assert result.throughput > MIN_THROUGHPUT_RPS
    assert result.p95_latency < MAX_P95_LATENCY

@pytest.mark.stress
@pytest.mark.slow
def test_sustained_load():
    """Test system under sustained load"""
    system = setup_test_system()
    
    start_time = time.time()
    duration = 300  # 5 minutes
    
    while time.time() - start_time < duration:
        response = system.process_request(create_test_request())
        assert response.status == "success"
        
        # Monitor resource usage
        assert_memory_usage_stable()
        assert_cpu_usage_reasonable()
```

### 5. Security Tests

#### Security/OSRB Test Framework
**Location**: `tests/security/`
**Purpose**: Validate security controls and detect OSRB exceptions.

```python
@pytest.mark.security
def test_input_sanitization():
    """Test that malicious inputs are properly sanitized"""
    malicious_inputs = [
        "'; DROP TABLE users; --",
        "<script>alert('xss')</script>",
        "../../../etc/passwd",
        "{{7*7}}",  # Template injection
    ]
    
    for malicious_input in malicious_inputs:
        response = system.process_request(malicious_input)
        assert response.status == "error"
        assert "sanitized" in response.message.lower()

@pytest.mark.security
@pytest.mark.network
def test_network_security():
    """Test network security controls"""
    # Test TLS enforcement
    with pytest.raises(ConnectionError):
        insecure_client = Client(use_tls=False)
        insecure_client.connect()
    
    # Test authentication
    unauthenticated_client = Client()
    response = unauthenticated_client.make_request()
    assert response.status_code == 401

@pytest.mark.security
def test_resource_limits():
    """Test resource exhaustion protection"""
    # Test memory limits
    large_request = create_large_request(size_mb=1000)
    response = system.process_request(large_request)
    assert response.status == "error"
    assert "resource limit" in response.message.lower()
```

### 6. Fault Tolerance Tests

#### Reliability Testing
**Location**: `tests/fault_tolerance/`
**Purpose**: Validate system behavior under failure conditions

```python
@pytest.mark.fault_tolerance
@pytest.mark.slow
async def test_network_partition_recovery():
    """Test system recovery from network partitions"""
    system = await create_distributed_system(nodes=3)
    
    # Introduce network partition
    await system.partition_network(nodes=[0], isolated_nodes=[1, 2])
    
    # System should continue operating with reduced capacity
    response = await system.process_request(test_request)
    assert response.status in ["success", "degraded"]
    
    # Heal partition
    await system.heal_network_partition()
    
    # System should return to full capacity
    await wait_for_system_recovery()
    response = await system.process_request(test_request)
    assert response.status == "success"

@pytest.mark.fault_tolerance
def test_graceful_degradation():
    """Test system degradation under resource pressure"""
    system = setup_test_system()
    
    # Gradually increase load
    for load_level in [10, 50, 100, 200, 500]:
        responses = system.process_concurrent_requests(load_level)
        
        success_rate = sum(1 for r in responses if r.status == "success") / len(responses)
        
        if load_level <= 100:
            assert success_rate >= 0.99  # High success rate under normal load
        else:
            assert success_rate >= 0.80  # Graceful degradation under high load
```


---

## Test Segmentation and Grouping

This section explains how tests are organized, segmented, and run within this project for both **Python** and **Rust** codebases. It covers the usage of **pytest markers** for Python tests and **Cargo features** for Rust tests, along with guidelines on running segmented tests efficiently. Please ensure that the marker names and features names across the toml files are consistent for Rust and Python.

---

### Python Tests Segmentation (pytest)

We use **pytest markers** to categorize tests by their purpose, requirements, and execution characteristics. This helps selectively run relevant tests during development, CI/CD, and nightly/weekly runs.

#### Test Types and Markers

| Marker                    | Description                                             |
|---------------------------|---------------------------------------------------------|
| `@pytest.mark.unit`       | Marks **unit tests**, testing individual components.    |
| `@pytest.mark.integration`| Marks **integration tests**, testing interactions between components. |
| `@pytest.mark.e2e`        | Marks **end-to-end (E2E) tests**, simulating user workflows. |
| `@pytest.mark.stress`     | Marks **stress tests** designed for load and robustness. |

### Further Classification (Integration Test Examples)

- **System Configuration Marks (Hardware Requirements):**
  - `@pytest.mark.gpus_needed_0` – No GPUs required.
  - `@pytest.mark.gpus_needed_1` – Requires 1 GPU.
  - `@pytest.mark.gpus_needed_2` – Requires 2 GPUs.
  
- **Life-Cycle Marks:**
  - `@pytest.mark.premerge` – Tests to run before code merge.
  - `@pytest.mark.postmerge` – Tests to run after merge.
  - `@pytest.mark.nightly` – Tests scheduled to run nightly.
  - `@pytest.mark.release` – Tests to run before releases.

- **Worker Framework Marks:**
  - `@pytest.mark.vllm`
  - `@pytest.mark.tensorrt_llm`
  - `@pytest.mark.sglang`

- **Execution Specific Marks:**
  - `@pytest.mark.fast` – Quick tests, often small models.
  - `@pytest.mark.slow` – Tests that take longer time >10 minutes.
  - `@pytest.mark.skip(reason="...")` – Skip tests with a reason.
  - `@pytest.mark.xfail(reason="...")` – Expected failing tests.

- **Component Specific Marks:**
  - `@pytest.mark.kvbm` – Tests for KVBM behavior.
  - `@pytest.mark.planner` – Tests for planner behavior.
  - `@pytest.mark.router` – Tests for router behavior.


### How to Run Python Tests by Marker

Run all tests with a specific marker:

```bash
pytest -m <marker_name>
```

### Rust Tests Segmentation using Cargo features

Tests can be conditionally compiled using `#[cfg(feature = "feature_name")]`. For example:

```rust
#[cfg(feature = "gpu")]
#[test]
fn test_gpu_acceleration() {
    // GPU-specific test code here
}
```
```rust
#[cfg(feature = "nightly")]
#[test]
fn test_nightly_only_feature() {
    // Nightly-only test code here
}
```

To combine features;

```rust
#[cfg(all(feature = "gpu", feature = "vllm"))]
#[test]
fn test_gpu_and_vllm() {
    // Test requiring both features
}

cargo test --features "gpu vllm"
```