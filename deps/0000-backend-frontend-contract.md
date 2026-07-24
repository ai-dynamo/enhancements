# DEP: Backend-Frontend Contract Formalization

| Status | Draft |
|--------|-------|
| **Authors** | @nnshah1 |
| **Category** | Process |
| **Sponsor** | TBD |
| **Required Reviewers** | TBD |
| **Review Date** | TBD |
| **Related PRs** | #54 |

## Summary

This DEP defines the contract formalization strategy between Dynamo's frontend and backend workers (vLLM, TRT-LLM, SGLang). It addresses schema documentation, enforcement gaps, capability negotiation, and protocol versioning needed for a stable 1.0 release.

## Motivation

The current frontend-backend communication has several critical gaps:

1. **Lossy Python Bridge**: Typed Rust structs become `Dict[str, Any]` in Python with no runtime validation
2. **No Capability Negotiation**: Backends self-register without proving they support claimed features
3. **Undocumented Schemas**: MDC, KV events, and health checks lack formal specifications
4. **Silent Failures**: Invalid parameters are logged as warnings, not returned as errors
5. **No Version Negotiation**: Protocol changes can break compatibility silently

### Current Enforcement State

| Layer | Validation | Coverage |
|-------|------------|----------|
| HTTP (Rust) | 25+ validators | Strong |
| Protocol (Rust) | Serde deserialization | Partial |
| Python Bridge | None | **Gap** |
| Backend Response | Structure only | Minimal |

## Goals

1. Define formal schemas for all inter-component contracts
2. Add typed Python interfaces to replace `Dict[str, Any]`
3. Implement capability negotiation between frontend and backends
4. Add response validation for semantic correctness
5. Establish protocol versioning strategy
6. Create contract compliance test suite

## Non-Goals

1. Changing the underlying transport (NATS, gRPC, etc.)
2. Breaking backward compatibility with existing deployments
3. Modifying third-party backend APIs (vLLM, SGLang internals)

---

## Contract Inventory

### 1. Worker Registration Contract

#### Current State

Workers register via `register_llm()` with 14 parameters:

```python
async def register_llm(
    model_input: ModelInput,           # Tokens | Text
    model_type: ModelType,             # Chat | Completions | Embedding | Tensor
    endpoint: Endpoint,
    model_path: str,
    model_name: Optional[str] = None,
    context_length: Optional[int] = None,
    kv_cache_block_size: Optional[int] = None,
    router_mode: Optional[RouterMode] = None,
    migration_limit: int = 0,
    runtime_config: Optional[ModelRuntimeConfig] = None,
    user_data: Optional[Dict[str, Any]] = None,
    custom_template_path: Optional[str] = None,
    lora_name: Optional[str] = None,
    base_model_path: Optional[str] = None,
) -> None
```

#### Issues

| Issue | Severity | Description |
|-------|----------|-------------|
| Opaque MDC | High | `ModelDeploymentCard` has no public schema |
| No capability proof | High | Worker claims support without verification |
| etcd keys undocumented | Medium | Key structure is implicit |
| Optional fields unclear | Medium | Which combinations are valid? |

#### Proposed Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "ModelDeploymentCard",
  "type": "object",
  "required": ["display_name", "slug", "model_input", "model_type"],
  "properties": {
    "version": {
      "type": "string",
      "const": "1.0"
    },
    "display_name": {
      "type": "string",
      "minLength": 1
    },
    "slug": {
      "type": "string",
      "pattern": "^[a-z0-9-]+$"
    },
    "model_input": {
      "enum": ["tokens", "text"]
    },
    "model_type": {
      "type": "array",
      "items": {
        "enum": ["chat", "completions", "embedding", "tensor"]
      },
      "minItems": 1
    },
    "context_length": {
      "type": "integer",
      "minimum": 1
    },
    "kv_cache_block_size": {
      "type": "integer",
      "minimum": 1
    },
    "runtime_config": {
      "$ref": "#/$defs/RuntimeConfig"
    },
    "capabilities": {
      "$ref": "#/$defs/Capabilities"
    }
  },
  "$defs": {
    "RuntimeConfig": {
      "type": "object",
      "properties": {
        "total_kv_blocks": {"type": "integer", "minimum": 0},
        "max_num_seqs": {"type": "integer", "minimum": 1},
        "max_num_batched_tokens": {"type": "integer", "minimum": 1}
      }
    },
    "Capabilities": {
      "type": "object",
      "properties": {
        "supports_streaming": {"type": "boolean"},
        "supports_logprobs": {"type": "boolean"},
        "supports_guided_decoding": {"type": "boolean"},
        "supports_tool_calls": {"type": "boolean"},
        "supports_vision": {"type": "boolean"},
        "max_stop_sequences": {"type": "integer"},
        "supported_sampling_params": {
          "type": "array",
          "items": {"type": "string"}
        }
      }
    }
  }
}
```

#### etcd Key Structure

```
/{namespace}/models/{model_name}/
├── mdc                     # ModelDeploymentCard JSON
├── checksum                # MDC checksum for validation
├── endpoints/
│   ├── {worker_id_1}       # Worker endpoint info
│   ├── {worker_id_2}
│   └── ...
└── capabilities            # Aggregated capabilities
```

---

### 2. Request/Response Contract

#### Current Schema Locations

| Schema | Location | Format |
|--------|----------|--------|
| KServe Inference | `lib/llm/src/grpc/protos/kserve.proto` | Protobuf |
| Chat Completions | `lib/llm/src/protocols/openai/chat_completions.rs` | Rust/Serde |
| Completions | `lib/llm/src/protocols/openai/completions.rs` | Rust/Serde |
| Common Extensions | `lib/llm/src/protocols/openai/common_ext.rs` | Rust/Serde |
| NVIDIA Extensions | `lib/llm/src/protocols/openai/nvext.rs` | Rust/Serde |
| Backend Output | `lib/llm/src/protocols/common/llm_backend.rs` | Rust/Serde |

#### Request Structure

```
┌─────────────────────────────────────────────────────────────┐
│              NvCreateChatCompletionRequest                  │
├─────────────────────────────────────────────────────────────┤
│  inner: CreateChatCompletionRequest (OpenAI base)           │
│    - model: string                                          │
│    - messages: Message[]                                    │
│    - temperature, top_p, n, stream, stop, max_tokens...     │
├─────────────────────────────────────────────────────────────┤
│  common: CommonExt (Dynamo extensions)                      │
│    - ignore_eos, min_tokens, top_k, min_p                   │
│    - repetition_penalty                                     │
│    - guided_json, guided_regex, guided_grammar              │
├─────────────────────────────────────────────────────────────┤
│  nvext: NvExt (NVIDIA extensions)                           │
│    - backend_instance_id, token_data                        │
│    - prefill_worker_id, decode_worker_id                    │
│    - enable_local_updates, expected_output_tokens           │
├─────────────────────────────────────────────────────────────┤
│  unsupported_fields: HashMap (catch-all for unknown)        │
└─────────────────────────────────────────────────────────────┘
```

#### Enforcement Gap: Python Bridge

**Current** (`handlers.py`):
```python
# Type safety lost - Dict[str, Any]
for key, value in request["sampling_options"].items():
    if value is not None and hasattr(sampling_params, key):
        setattr(sampling_params, key, value)  # NO TYPE CHECK
```

**Proposed** - Add Pydantic models:
```python
from pydantic import BaseModel, Field, validator

class SamplingOptions(BaseModel):
    temperature: Optional[float] = Field(None, ge=0.0, le=2.0)
    top_p: Optional[float] = Field(None, ge=0.0, le=1.0)
    top_k: Optional[int] = Field(None, ge=-1)
    min_p: Optional[float] = Field(None, ge=0.0, le=1.0)
    frequency_penalty: Optional[float] = Field(None, ge=-2.0, le=2.0)
    presence_penalty: Optional[float] = Field(None, ge=-2.0, le=2.0)
    repetition_penalty: Optional[float] = Field(None, ge=0.0, le=2.0)

    class Config:
        extra = "forbid"  # Reject unknown fields

class PreprocessedRequest(BaseModel):
    token_ids: List[int]
    sampling_options: SamplingOptions
    stop_conditions: StopConditions
    output_options: OutputOptions

    class Config:
        extra = "forbid"
```

#### Response Validation

**Current**: Only structure validation (serde deserialization)

**Proposed**: Add semantic validation:
```python
class BackendOutput(BaseModel):
    token_ids: List[int]
    text: Optional[str] = None
    finish_reason: Optional[FinishReason] = None
    completion_usage: Optional[CompletionUsage] = None

    @validator("token_ids")
    def validate_token_ids(cls, v):
        if any(t < 0 for t in v):
            raise ValueError("token_ids must be non-negative")
        return v

    @validator("completion_usage")
    def validate_usage(cls, v):
        if v and v.completion_tokens < 0:
            raise ValueError("completion_tokens must be non-negative")
        return v
```

---

### 3. KV Cache Events Contract

#### Current Event Format

```python
# BlockStored
{
    "type": "BlockStored",
    "block_hashes": [signed_i64, ...],
    "parent_block_hash": signed_i64 | None,
    "token_ids": [int, ...],
    "block_size": int,
    "lora_id": int | None,
}

# BlockRemoved
{
    "type": "BlockRemoved",
    "block_hashes": [signed_i64, ...],
}

# AllBlocksCleared
{
    "type": "AllBlocksCleared",
}
```

#### Issues

| Issue | Severity | Description |
|-------|----------|-------------|
| No formal schema | High | Format is implicit in code |
| Block hash algorithm undocumented | High | How are hashes computed? |
| NATS subjects implicit | Medium | Subject naming not specified |
| Ordering guarantees unclear | Medium | Are events ordered per-worker? |

#### Proposed Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "KVCacheEvent",
  "oneOf": [
    {"$ref": "#/$defs/BlockStored"},
    {"$ref": "#/$defs/BlockRemoved"},
    {"$ref": "#/$defs/AllBlocksCleared"}
  ],
  "$defs": {
    "BlockStored": {
      "type": "object",
      "required": ["type", "block_hashes", "token_ids", "block_size"],
      "properties": {
        "type": {"const": "BlockStored"},
        "block_hashes": {
          "type": "array",
          "items": {"type": "integer"},
          "minItems": 1,
          "description": "Signed 64-bit block hashes"
        },
        "parent_block_hash": {
          "type": ["integer", "null"],
          "description": "Parent block hash for prefix chains"
        },
        "token_ids": {
          "type": "array",
          "items": {"type": "integer", "minimum": 0}
        },
        "block_size": {
          "type": "integer",
          "minimum": 1
        },
        "lora_id": {
          "type": ["integer", "null"],
          "description": "LoRA adapter ID, null for base model"
        }
      }
    },
    "BlockRemoved": {
      "type": "object",
      "required": ["type", "block_hashes"],
      "properties": {
        "type": {"const": "BlockRemoved"},
        "block_hashes": {
          "type": "array",
          "items": {"type": "integer"},
          "minItems": 1
        }
      }
    },
    "AllBlocksCleared": {
      "type": "object",
      "required": ["type"],
      "properties": {
        "type": {"const": "AllBlocksCleared"}
      }
    }
  }
}
```

#### NATS Subject Convention

```
dynamo.{namespace}.kv.{component}.{worker_id}.events
dynamo.{namespace}.kv.{component}.{worker_id}.metrics
```

#### Block Hash Algorithm

Document the hash computation:
```python
def compute_block_hash(token_ids: List[int], parent_hash: Optional[int]) -> int:
    """
    Compute block hash using FNV-1a algorithm.

    Args:
        token_ids: Tokens in this block
        parent_hash: Hash of parent block (None for root)

    Returns:
        Signed 64-bit hash (Python int, transmitted as i64)
    """
    # Implementation in Rust: lib/llm/src/kv_router/indexer.rs
```

---

### 4. Health Check Contract

#### Current State

Health checks are engine-specific payloads:

**vLLM (text mode)**:
```python
{"prompt": "Test", "temperature": 0.0, "max_tokens": 1}
```

**vLLM (token mode)**:
```python
{"token_ids": [1], "sampling_options": {"temperature": 0.0}}
```

**SGLang**:
```python
{"token_ids": [bos_token_id], "sampling_options": {...}}
```

#### Issues

| Issue | Severity | Description |
|-------|----------|-------------|
| No unified schema | Medium | Each engine has different format |
| Success criteria undefined | Medium | What makes a health check pass? |
| Timeout not specified | Low | How long to wait? |

#### Proposed Unified Contract

```json
{
  "title": "HealthCheckPayload",
  "type": "object",
  "required": ["type"],
  "properties": {
    "type": {
      "enum": ["generation", "embedding"]
    },
    "timeout_ms": {
      "type": "integer",
      "default": 5000,
      "minimum": 100
    }
  },
  "allOf": [
    {
      "if": {"properties": {"type": {"const": "generation"}}},
      "then": {
        "properties": {
          "token_ids": {"type": "array", "items": {"type": "integer"}},
          "max_tokens": {"type": "integer", "default": 1},
          "temperature": {"type": "number", "default": 0.0}
        }
      }
    }
  ]
}
```

#### Success Criteria

```python
class HealthCheckResult(BaseModel):
    success: bool
    latency_ms: float
    tokens_generated: int
    error: Optional[str] = None

def is_healthy(result: HealthCheckResult) -> bool:
    return (
        result.success and
        result.tokens_generated >= 1 and
        result.latency_ms < timeout_ms
    )
```

---

### 5. Metrics Contract

#### Current Metrics

```python
class WorkerStats:
    request_active_slots: int
    request_total_slots: int
    num_requests_waiting: int
    data_parallel_rank: Optional[int]

class KvStats:
    kv_active_blocks: int
    kv_total_blocks: int
    gpu_cache_usage_perc: float
    gpu_prefix_cache_hit_rate: float
```

#### Issues

| Issue | Severity | Description |
|-------|----------|-------------|
| Update frequency unspecified | Medium | How often should metrics update? |
| DP rank aggregation unclear | Medium | How to aggregate across ranks? |
| Staleness detection missing | Low | How to detect stale metrics? |

#### Proposed Contract

```python
class MetricsContract:
    # Update frequency
    MIN_UPDATE_INTERVAL_MS = 100
    MAX_UPDATE_INTERVAL_MS = 1000

    # Staleness threshold
    STALE_THRESHOLD_MS = 5000

    # Required metrics
    REQUIRED_WORKER_STATS = ["request_active_slots", "request_total_slots"]
    REQUIRED_KV_STATS = ["kv_active_blocks", "kv_total_blocks"]
```

---

## Enforcement Strategy

### Phase 1: Python Type Safety

Add Pydantic models for all Python interfaces:

```python
# lib/bindings/python/src/dynamo/contracts/
├── __init__.py
├── request.py      # PreprocessedRequest, SamplingOptions, etc.
├── response.py     # BackendOutput, LLMEngineOutput, etc.
├── registration.py # ModelDeploymentCard, RuntimeConfig, etc.
├── events.py       # KVCacheEvent, BlockStored, etc.
└── health.py       # HealthCheckPayload, HealthCheckResult
```

### Phase 2: Capability Negotiation

Add capability declaration to registration:

```python
class BackendCapabilities(BaseModel):
    supports_streaming: bool = True
    supports_logprobs: bool = False
    supports_guided_decoding: bool = False
    supports_tool_calls: bool = False
    supports_vision: bool = False
    max_stop_sequences: int = 4
    supported_sampling_params: List[str] = []

async def register_llm(
    ...,
    capabilities: BackendCapabilities,  # NEW: Required
) -> None:
    # Validate capabilities match model_type
    if ModelType.Chat in model_type and not capabilities.supports_streaming:
        raise ValueError("Chat models must support streaming")
```

### Phase 3: Response Validation

Add semantic validation to backend output processing:

```rust
// lib/llm/src/protocols/common/llm_backend.rs
impl BackendOutput {
    pub fn validate(&self) -> Result<(), ValidationError> {
        // Token IDs must be non-negative
        if self.token_ids.iter().any(|&t| t < 0) {
            return Err(ValidationError::InvalidTokenIds);
        }

        // Finish reason consistency
        if self.finish_reason == Some(FinishReason::Length)
            && self.token_ids.is_empty() {
            return Err(ValidationError::InconsistentFinishReason);
        }

        Ok(())
    }
}
```

### Phase 4: Protocol Versioning

Add version headers to all communications:

```python
# Request header
X-Dynamo-Protocol-Version: 1.0
X-Dynamo-Client-Version: 0.9.0

# Response header
X-Dynamo-Protocol-Version: 1.0
X-Dynamo-Server-Version: 0.9.0
```

Version negotiation:
```python
def negotiate_version(client_version: str, server_versions: List[str]) -> str:
    """Find highest compatible version."""
    client_major = int(client_version.split(".")[0])
    for server_version in sorted(server_versions, reverse=True):
        server_major = int(server_version.split(".")[0])
        if server_major == client_major:
            return server_version
    raise IncompatibleVersionError(client_version, server_versions)
```

---

## Implementation Checklist

### Phase 1: Schema Documentation (v0.9)

- [ ] Document MDC JSON schema
- [ ] Document KV event JSON schema
- [ ] Document health check schema
- [ ] Document etcd key structure
- [ ] Document NATS subject naming

### Phase 2: Python Type Safety (v0.10)

- [ ] Create Pydantic models for requests
- [ ] Create Pydantic models for responses
- [ ] Create Pydantic models for registration
- [ ] Update vLLM handlers to use typed models
- [ ] Update SGLang handlers to use typed models
- [ ] Update TRT-LLM handlers to use typed models

### Phase 3: Capability Negotiation (v0.11)

- [ ] Add BackendCapabilities to registration
- [ ] Validate capabilities at registration
- [ ] Expose capabilities via MDC
- [ ] Frontend checks capabilities before routing

### Phase 4: Response Validation (v0.11)

- [ ] Add semantic validation to BackendOutput
- [ ] Add token ID range validation
- [ ] Add finish reason consistency checks
- [ ] Add usage validation

### Phase 5: Protocol Versioning (v1.0)

- [ ] Add version headers to HTTP
- [ ] Add version field to MDC
- [ ] Implement version negotiation
- [ ] Document compatibility matrix

---

## Contract Compliance Testing

### Test Categories

```python
# tests/contracts/
├── test_mdc_schema.py           # MDC schema validation
├── test_request_schema.py       # Request schema validation
├── test_response_schema.py      # Response schema validation
├── test_kv_events_schema.py     # KV event schema validation
├── test_health_check.py         # Health check contract
├── test_capabilities.py         # Capability negotiation
├── test_type_safety.py          # Python type enforcement
└── test_version_compat.py       # Version compatibility
```

### Example Contract Test

```python
def test_backend_rejects_invalid_sampling_options():
    """Backend must reject invalid sampling parameters."""
    request = PreprocessedRequest(
        token_ids=[1, 2, 3],
        sampling_options=SamplingOptions(
            temperature=5.0,  # Invalid: > 2.0
        ),
    )

    with pytest.raises(ValidationError) as exc:
        request.validate()

    assert "temperature" in str(exc.value)
    assert "must be between 0 and 2" in str(exc.value)
```

---

## Backward Compatibility

### Migration Strategy

1. **v0.9**: Add schemas as documentation (no enforcement)
2. **v0.10**: Add Pydantic models with `extra="ignore"` (warn on unknown)
3. **v0.11**: Change to `extra="forbid"` (reject unknown)
4. **v1.0**: Full enforcement, version negotiation

### Deprecation Policy

- Unknown fields: Warn in v0.10, reject in v0.11
- Missing capabilities: Default to minimal in v0.10, required in v1.0
- Old MDC format: Accept in v0.10-v0.11, reject in v1.1

---

## Alternate Solutions

### Option A: Protobuf for All Communication

Use protobuf instead of JSON for type safety. **Rejected**:
- Breaks OpenAI API compatibility
- Requires client library changes
- JSON is more debuggable

### Option B: Runtime Type Checking in Rust Only

Keep Python untyped, validate at Rust boundary. **Rejected**:
- Errors surface far from source
- Python debugging harder
- Backend-specific issues missed

### Option C: Code Generation from Schemas

Generate Python/Rust from JSON Schema. **Deferred**:
- Good long-term approach
- Requires tooling investment
- Can adopt post-1.0

---

## Related Proposals

- DEP-0000: Kubernetes API Formalization
- DEP-0000: Frontend CLI Formalization
- DEP-0005: LLM Request Migration

---

## References

- [OpenAI API Reference](https://platform.openai.com/docs/api-reference)
- [KServe Inference Protocol](https://kserve.github.io/website/latest/modelserving/data_plane/v2_protocol/)
- [Pydantic Documentation](https://docs.pydantic.dev/)
- [JSON Schema Specification](https://json-schema.org/)
