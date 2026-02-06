# Native Framework Processing Fallback

**Status**: Draft

**Authors**: Alec in consultation with Ryan and Graham

**Category**: Architecture

**Sponsor**: Alec

**Required Reviewers**: Ryan McCormick, Itay Neeman, Neelay Shah

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary


# Motivation

Dynamo’s Rust-based frontend is providing excellent performance, but has recently been a source of friction for users. This is due to it implementing the pre- and post-processing pipelines for requests (tokenization, chat template application, reasoning parsers, etc), and if a particular piece of functionality is missing for some model, it requires work to add (even if the functionality is available in the underlying engine).

## Goals

Allow the user to select between dynamo processing or native engine processing with a command line flag. The default will be the fast path of dynamo based processing with the option to add a flag:

--native-framework-processing
--vllm-processing
--sglang-processing
--trtllm-processing

## Requirements

REQ 1: User SHOULD be able to select between dynamo processing and the backend enginge processing.

REQ 2: Preprocessing MUST happen on the frontend.
-  To enable KV Cache Routing we need to have tokens in frontend in order to calculate hashes to match against the indexers hashes created from KV events emitted from the backend.

REQ 3: PostProcessing SHOULD move to the backend engine
- Post-processing is currently done on the engine side, however the intention was to co-located it with the backend where there are many CPU's sitting idle alongside the GPU's.
- This also means that post-processing scales along with the number of engine instances.

# Proposal

## Pre-Processing

### Current Architecture

Rust preprocessor handles: prompt templating → tokenization → parameter extraction

```
NvCreateChatCompletionRequest → (Rust) → PreprocessedRequest {token_ids, sampling_options, ...}
```

### Proposed: Python Tokenizer Adapter

**Goal**: Use vLLM's native chat templates/tokenization while keeping parameter extraction in Rust.

**Boundary**: Only pass `(messages, model, tools)` → receive `token_ids`

#### Python Interface

```python
class TokenizerProtocol(ABC):
    def tokenize(
        self,
        messages: List[Dict[str, Any]],
        model: str,
        tools: Optional[List[Dict]] = None
    ) -> List[int]:
        """Combine chat template + tokenization in one call."""
        pass

class VllmTokenizer(TokenizerProtocol):
    def tokenize(self, messages, model, tools=None):
        prompt = self.tokenizer.apply_chat_template(
            messages, tokenize=False, add_generation_prompt=True, tools=tools
        )
        return self.tokenizer.encode(prompt)
```

#### Rust Adapter

```rust
pub trait TokenizerAdapter: Send + Sync {
    fn tokenize(
        &self,
        messages: &[Message],
        model: &str,
        tools: Option<&[Tool]>,
    ) -> anyhow::Result<Vec<u32>>;
}

pub struct PythonTokenizerAdapter {
    py_tokenizer: Py<PyAny>,
}

impl TokenizerAdapter for PythonTokenizerAdapter {
    fn tokenize(&self, messages: &[Message], model: &str, tools: Option<&[Tool]>)
        -> anyhow::Result<Vec<u32>>
    {
        Python::with_gil(|py| {
            let result = self.py_tokenizer.call_method1(
                py, "tokenize",
                (messages_to_pylist(py, messages)?, model, tools.map(|t| tools_to_pylist(py, t)).transpose()?)
            )?;
            Ok(result.extract(py)?)
        })
    }
}

// Helper functions convert Rust structs to Python dicts/lists
```

#### Integration

```rust
impl OpenAIPreprocessor {
    pub async fn preprocess_with_adapter(
        &self,
        request: &NvCreateChatCompletionRequest,
        tokenizer: &dyn TokenizerAdapter,
    ) -> Result<PreprocessedRequest> {
        let mut builder = self.builder(request)?;  // Rust: extract params
        let token_ids = tokenizer.tokenize(...)?;   // Python: tokenize
        builder.token_ids(token_ids);
        self.gather_multi_modal_data(request, &mut builder).await?;
        builder.build()
    }
}
```

#### Initialization

```rust
// Rust passes tokenizer path to Python factory
let tokenizer_path = card.get_tokenizer_path()?;
let adapter = Python::with_gil(|py| {
    let py_tokenizer = factory.call1(py, (tokenizer_path,))?;
    Ok(Arc::new(PythonTokenizerAdapter::new(py_tokenizer)))
})?;
```

```python
# Python factory creates tokenizer from path
def create_vllm_tokenizer(tokenizer_path: str) -> TokenizerProtocol:
    tokenizer = AutoTokenizer.from_pretrained(os.path.dirname(tokenizer_path))
    return VllmTokenizer(tokenizer)

# Main
tokenizer_factory = create_vllm_tokenizer if flags.vllm_processing else None
await run_input(runtime, mode, engine, tokenizer_factory)
```

## Post-Processing

### Current Architecture

**Frontend**: Post-processing (detokenization, tool calling) happens in Rust via `backend.backward_edge()` and `preprocessor_op.backward_edge()`

**Backend** (`serve_endpoint`): Minimal - just wraps `PythonAsyncEngine` in ingress handler

```rust
// Frontend pipeline includes Rust post-processing
.link(service_backend)?
.link(backend.backward_edge())?          // Detokenization in Rust
.link(preprocessor_op.backward_edge())?  // Tool calling in Rust
```

### Two Processing Modes

#### Mode 1: Rust Post-Processing on Backend (Proposed)

Move Rust detokenization/tool calling to backend servers (better CPU utilization):

```rust
// Backend serve_endpoint builds larger pipeline
let pipeline = frontend
    .link(tool_calling.forward_edge())?  // passthrough
    .link(backend.forward_edge())?       // passthrough
    .link(service_backend)?              // PythonAsyncEngine
    .link(backend.backward_edge())?      // Detokenization in Rust (on backend)
    .link(tool_calling.backward_edge())? // Tool calling in Rust (on backend)
    .link(frontend)?
```

**Benefits**: Offloads CPU work to backend servers, scales with engine instances

#### Mode 2: vLLM Native Processing (`--vllm-processing`)

Let vLLM handle detokenization and tool calling internally:

**Python Handler:**
```python
# Enable vLLM's native processing
sampling_params.detokenize = True

# vLLM returns text + tool calls
yield postprocessor.process_tokens(
    token_ids=output.token_ids,
    delta_text=output.text,  # Already detokenized
    ...
)
```

**VllmPostProcessor** uses vLLM's tool parsers:
```python
tool_parser = ToolParserManager.get_tool_parser(tool_parser_name)(tokenizer)
tool_parser.extract_tool_calls_streaming(...)
```

**Frontend Pipeline** skips Rust post-processing:
```rust
// Note: backend.backward_edge() and preprocessor_op.backward_edge() removed
let engine = frontend
    .link(preprocessor_op.forward_edge())?
    .link(service_backend)?
    .link(frontend)?
```
