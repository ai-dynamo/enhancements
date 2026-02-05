# Responses API

> **Linear Ticket**: [DYN-2045](https://linear.app/nvidia/issue/DYN-2045) - Investigate `v1/responses` being stateful
> **Implementation PR**: [dynamo#5854](https://github.com/ai-dynamo/dynamo/pull/5854) - Branch `responses-api-compliance`

## Current State (PR #5854)

Dynamo implements the OpenAI Responses API (`POST /v1/responses`) as a translation layer
over the existing chat completions pipeline. All 6 OpenResponses compliance tests pass.

### What Works

| Feature | Status |
|---------|--------|
| `Input::Text` (string input) | Working |
| `Input::Items` (structured messages with roles) | Working |
| `instructions` (system prompt) | Working |
| Image/multimedia content in `Input::Items` | Working |
| Multi-turn conversation history | Working |
| `stream: true` (SSE streaming) | Working |
| `tools` / `tool_choice` (function calling) | Working |
| Function call + function call output in `Input::Items` | Working |
| Client disconnect detection (streaming) | Working |
| Model, temperature, top_p, max_output_tokens | Working |
| NVIDIA extensions (nvext) passthrough | Working |

### What Does Not Work Yet

| Feature | Notes |
|---------|-------|
| `text.format` (json_object, json_schema) | Needs `response_format` forwarding |
| `reasoning` config | Needs `reasoning.effort` forwarding |
| `truncation` (auto) | Needs context window management |
| `background` execution | Needs async task + storage + polling |

### Recently Implemented (Phase A Complete)

| Feature | Status | PR |
|---------|--------|-----|
| `previous_response_id` | ✅ Working | dynamo#5854 |
| `store: true/false` | ✅ Working | dynamo#5854 |
| `GET /v1/responses/{id}` | ✅ Working | dynamo#5854 |
| `DELETE /v1/responses/{id}` | ✅ Working | dynamo#5854 |
| Streaming + storage | ✅ Working | dynamo#5854 |
| Session/tenant isolation | ✅ Working | dynamo#5854 |
| Cursor-based pagination | ✅ Working | dynamo#5854 |
| Session forking (`fork_session`) | ✅ Working | dynamo#5854 |
| Redis backend | ✅ Behind feature flag | dynamo#5854 |

---

## Architecture

```
POST /v1/responses
       |
       v
  NvCreateResponse
       |
       v
  TryFrom<NvCreateResponse> for NvCreateChatCompletionRequest
       |  - Input::Text -> user message
       |  - Input::Items -> system/user/assistant/tool messages
       |  - instructions -> system message (prepended)
       |  - tools -> ChatCompletionTool[]
       |  - tool_choice -> ChatCompletionToolChoiceOption
       |  - FunctionCall items -> assistant message with tool_calls
       |  - FunctionCallOutput items -> tool message
       v
  Chat Completions Engine (.generate() stream)
       |
       v
  Response conversion
       |  Non-streaming: aggregate stream -> NvResponse
       |  Streaming: ResponseStreamConverter state machine -> SSE events
       v
  HTTP Response
```

Streaming event sequence:
```
response.created -> response.in_progress ->
  response.output_item.added -> response.content_part.added ->
  N x response.output_text.delta ->
  response.output_text.done -> response.content_part.done ->
  response.output_item.done -> response.completed -> [DONE]
```

For function calls, `function_call_arguments.delta` and `function_call_arguments.done`
events are interleaved per tool call, each with its own `output_index`.

---

## Key Files

| File | Role |
|------|------|
| `lib/async-openai/src/types/responses/` | All Responses API types (~4,900 lines ported from upstream async-openai) |
| `lib/llm/src/protocols/openai/responses/mod.rs` | NvCreate/NvResponse wrappers, request/response conversion |
| `lib/llm/src/protocols/openai/responses/stream_converter.rs` | Streaming SSE event state machine |
| `lib/llm/src/http/service/openai.rs` | HTTP handler, validation, disconnect handling |

---

## Testing

**CI (pre_merge)**: `ResponsesPayload` and `ResponsesStreamPayload` run as part of
`test_sglang_deployment[aggregated]` against a real Qwen3-0.6B model via SGLang.

**Full compliance**: Use the [OpenResponses](https://www.openresponses.org/compliance) bun CLI:
```bash
bun run test:compliance --base-url http://localhost:9000/v1 --api-key test --model <model>
```

---

## Roadmap

### Phase A: Storage + `previous_response_id` + GET/DELETE ✅ IMPLEMENTED

> **Status**: Complete. See dynamo#5854 branch `responses-api-compliance`.
> 57+ tests passing including 1000-session concurrent load test.

The goal is to enable multi-turn conversations where the client passes
`previous_response_id` instead of re-sending the full message history. This requires
persisting responses and walking the chain at request time.

#### Storage trait (Implemented)

```rust
/// Located at: lib/llm/src/storage/response_storage.rs
#[async_trait]
pub trait ResponseStorage: Send + Sync {
    /// Store a response with tenant/session isolation
    async fn store_response(
        &self,
        tenant_id: &str,
        session_id: &str,
        response_id: Option<&str>,
        response: serde_json::Value,
        ttl: Option<Duration>,
    ) -> Result<String, StorageError>;

    /// Get a response, validating tenant and session
    async fn get_response(
        &self,
        tenant_id: &str,
        session_id: &str,
        response_id: &str,
    ) -> Result<StoredResponse, StorageError>;

    /// Delete a response
    async fn delete_response(
        &self,
        tenant_id: &str,
        session_id: &str,
        response_id: &str,
    ) -> Result<(), StorageError>;

    /// List responses with cursor-based pagination
    async fn list_responses(
        &self,
        tenant_id: &str,
        session_id: &str,
        limit: Option<usize>,
        after: Option<&str>,  // Cursor for pagination
    ) -> Result<Vec<StoredResponse>, StorageError>;

    /// Fork a session (branch conversation from a point)
    async fn fork_session(
        &self,
        tenant_id: &str,
        source_session_id: &str,
        target_session_id: &str,
        up_to_response_id: Option<&str>,
    ) -> Result<usize, StorageError>;
}
```

#### StoredResponse (Implemented)

```rust
/// Located at: lib/llm/src/storage/response_storage.rs
pub struct StoredResponse {
    pub response_id: String,        // Unique response identifier
    pub tenant_id: String,          // Tenant isolation
    pub session_id: String,         // Session/conversation context
    pub response: serde_json::Value, // The actual response data
    pub created_at: u64,            // Unix epoch seconds
    pub expires_at: Option<u64>,    // TTL expiration (optional)
}
```

Key differences from the original design:
- Added `tenant_id` and `session_id` for multi-tenant isolation
- Uses key pattern `{tenant_id}:{session_id}:responses:{response_id}`
- TTL-based expiration support
- `previous_response_id` stored in response metadata, not as a separate field

#### Context resolution

When `previous_response_id` is set:

1. `get_chain(previous_response_id, 100)` walks backward through the linked list
2. Each stored response yields `(input, output)` pairs
3. These are flattened into `ChatCompletionRequestMessage` sequence:
   - Stored `instructions` -> system message (from the original request, not re-applied)
   - Stored `input` items -> user/assistant/tool messages
   - Stored `output` items -> assistant messages
4. The current request's `input` and `instructions` are appended at the end
5. The full message list is sent to the chat completions engine

```
Chain: resp_001 -> resp_002 -> resp_003 (current request with previous_response_id=resp_003)

Messages built:
  [resp_001.instructions as system]
  [resp_001.input as user]
  [resp_001.output as assistant]
  [resp_002.input as user]
  [resp_002.output as assistant]
  [resp_003.input as user]
  [resp_003.output as assistant]
  [current request.instructions as system]  // override if provided
  [current request.input as user]
```

#### Default backend: in-memory (Implemented)

```rust
/// Located at: lib/llm/src/storage/manager.rs
pub struct InMemoryResponseStorage {
    storage: Arc<RwLock<HashMap<String, StoredResponse>>>,
}
```

This is the default for single-process deployments. No config needed. Data is lost on
restart, which is fine for dev/testing. Uses tokio `RwLock` for async-safe concurrent access.

#### Pluggable backends (Implemented)

The `ResponseStorage` trait is object-safe, so backends can be swapped at startup:

- `InMemoryResponseStorage` - default, zero-config ✅
- `RedisResponseStorage` - for multi-instance deployments, TTL-based retention ✅ (behind `redis-storage` feature)
- `PostgresResponseStorage` - for durable storage, SQL queries (future)

Backend selection via environment variables:
```bash
DYNAMO_RESPONSES_BACKEND=memory          # default
DYNAMO_RESPONSES_BACKEND=redis
DYNAMO_RESPONSES_REDIS_URL=redis://host:6379
DYNAMO_RESPONSES_DEFAULT_TTL_SECS=86400  # 24 hours
DYNAMO_RESPONSES_MAX_TTL_SECS=604800     # 7 days
DYNAMO_RESPONSES_MAX_PER_SESSION=1000
```

#### Session Locking (Implemented)

Added `SessionLock` trait for concurrent access control when multiple requests target
the same session with `previous_response_id`:

```rust
/// Located at: lib/llm/src/storage/session_lock.rs
#[async_trait]
pub trait SessionLock: Send + Sync {
    async fn acquire(&self, key: &str, timeout: Duration) -> Result<LockGuard, LockError>;
    async fn try_acquire(&self, key: &str) -> Result<LockGuard, LockError>;
    async fn is_locked(&self, key: &str) -> bool;
}
```

Implementations:
- `InMemorySessionLock` - per-key semaphores, single-process ✅
- `RedisSessionLock` - distributed locking via Redis ✅ (behind `redis-storage` feature)

#### Integration points

- `service_v2::State` gets an `Arc<dyn ResponseStore>`
- `responses()` handler stores completed responses before returning
- `responses()` handler resolves `previous_response_id` before building messages
- New routes: `GET /v1/responses/{id}`, `DELETE /v1/responses/{id}`
- `store: false` on the request skips persistence

#### SGLang reference

SGLang implements this pattern in `sgl-model-gateway/src/data_connector/`:
- `core.rs` - trait definitions (`ResponseStorage`, `StoredResponse`, `ResponseChain`)
- `memory.rs` - in-memory backend (HashMap + RwLock)
- `redis.rs` - Redis backend (HSET per response, ZSET for user indexing, TTL)
- `postgres.rs` - PostgreSQL backend

---

### Phase B: Conversation Graph

Phase A gives us linear chains (`previous_response_id` links). Phase B adds tree
structure for branching conversations.

#### Use cases

- **Branching**: User sends different follow-ups to the same response (A/B testing prompts)
- **Tree-of-thought**: Multiple reasoning paths from the same context
- **Visualization**: Show conversation as a tree with siblings, not just a single thread

#### Extension trait

```rust
#[async_trait]
pub trait ResponseStoreGraph: ResponseStore {
    /// Get the full tree rooted at the earliest ancestor.
    async fn get_tree(&self, id: &str) -> Result<ConversationTree>;

    /// Get immediate children of a response.
    async fn get_children(&self, id: &str) -> Result<Vec<StoredResponse>>;

    /// Get siblings (same parent).
    async fn get_siblings(&self, id: &str) -> Result<Vec<StoredResponse>>;

    /// Get leaf nodes (responses with no children).
    async fn get_leaves(&self, root_id: &str) -> Result<Vec<StoredResponse>>;
}
```

This is opt-in: the in-memory backend can implement it by maintaining a
`children: HashMap<String, Vec<String>>` index alongside the flat store.
Redis/Postgres backends can implement it with secondary indices.

#### Data model

No schema changes needed. The tree is implicit from `previous_response_id` links:
- Root: response with `previous_response_id: None`
- Branch point: response with multiple children (multiple responses pointing to it)
- Leaf: response with no children

The `ConversationTree` struct materializes this:
```rust
pub struct ConversationNode {
    pub response: StoredResponse,
    pub children: Vec<ConversationNode>,
}

pub struct ConversationTree {
    pub root: ConversationNode,
}
```

#### Optional API endpoints

These are admin/debug endpoints, not part of the OpenAI spec:

```
GET /v1/responses/{id}/tree      -> ConversationTree
GET /v1/responses/{id}/children  -> Vec<StoredResponse>
GET /v1/responses/{id}/siblings  -> Vec<StoredResponse>
```

---

### Phase C: Advanced Features

- **Text format**: `text.format` = `json_object` or `json_schema` -> forward as `response_format`
  on the chat completion request
- **Reasoning**: `reasoning.effort` -> forward to the engine (model-dependent)
- **Truncation**: `truncation: auto` -> trim oldest messages to fit context window
  (requires tokenizer access for accurate counting)
- **Background**: `background: true` -> return immediately with `status: queued`,
  run generation async, store result, client polls `GET /v1/responses/{id}`

---

## SGLang Reference

SGLang's model gateway implements a similar conversion-based approach:

| Component | Path (sgl-model-gateway/src/) |
|-----------|------|
| Types | `protocols/responses.rs` |
| Conversions | `routers/grpc/regular/responses/conversions.rs` |
| Stream emitter | `routers/grpc/common/responses/streaming.rs` |
| Storage trait | `data_connector/core.rs` |
| In-memory store | `data_connector/memory.rs` |
| Redis store | `data_connector/redis.rs` |
| Postgres store | `data_connector/postgres.rs` |
| Factory | `data_connector/factory.rs` |
