# Support for Responses API

**Status**: Approved

**Authors**: Paul Hendricks

**Category**: Architecture 

**Replaces**: N/A

**Replaced By**: N/A 

**Sponsor**: Ryan Olson

**Required Reviewers**: Ryan Olson, Neelay Shah, Maksim Khadkevich, Ryan McCormick

**Review Date**: 07/01/2025

**Pull Request**: https://github.com/ai-dynamo/enhancements/pull/16

**Implementation PR / Tracking Issue**: https://github.com/ai-dynamo/dynamo/pull/1694

# Proposal

### Background

NVIDIA NIM and NVIDIA Dynamo currently use the OpenAI ChatCompletions API for text-to-text capabilities. With OpenAI’s introduction of the Responses API — a stateful, event-driven surface that unifies truly multimodal (image, text, audio) inputs and outputs, built-in tools, and rich semantic SSE events — there is an opportunity for NVIDIA to define a Responses API to serve as the http frontend, powered by the NIM/Dynamo backend. This proposal details observations of several key aspects of the OpenAI Responses API.

### Motivation

OpenAI has stated they will continue to support the OpenAI ChatCompletions API indefinitely. However, while OpenAI will continue to support the OpenAI ChatCompletions API, no new features will be introduced there. And instead, OpenAI will be surfacing new features and model capabilities in their OpenAI Responses API. OpenAI encourages developers to start new, greenfield projects with their OpenAI Responses API over the ChatCompletions API.

The OpenAI Responses API also powers the OpenAI Assistants API, OpenAI Agents SDK, and OpenAI Realtime API for building agents, voice agents, multi-turn diaglog workflows, and GenAI applications.

The OpenAI Responses API is a significant project and undertaking with many pieces that are likely out of scope (e.g. web_search, computer_use_agent). The goal of this proposal is not to build it - but rather to explore core pieces and better understand how NVIDIA NIM and NVIDIA Dynamo could implement a compatible frontend API.

### Outline

The Insights section details observations of several key aspects of the OpenAI Responses API:

- Statefulness
- Semantic Events
- SSE Streaming
- Built-in tools/Agentic API primitives

The Proposal section details how we might add components like statefulness or rich, semantic SSE events to NIM/Dynamo.

### Background Materials

- [Responses vs. Chat Completions](https://platform.openai.com/docs/guides/responses-vs-chat-completions)
- [OpenAI Responses API Reference](https://platform.openai.com/docs/api-reference/responses)
- [NVIDIA NIM](https://docs.nvidia.com/nim/index.html)
- [NVIDIA Dynamo](https://github.com/ai-dynamo/dynamo)
- [OpenAI Realtime API](https://platform.openai.com/docs/guides/realtime)

### Insights

To better understand the OpenAI Responses API and how it differs from the OpenAI ChatCompletions API, let's look at a simple call:

```bash
curl https://api.openai.com/v1/responses \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "gpt-4.1",
    "tools": [{"type": "web_search_preview"}],
    "input": "What was a positive news story from today?"
  }'
```

There are a few things to note

- *Simplified Input*: `input` replaces `messages`. You no longer need to construct a structured sequence of user/assistant messages. This flattens the UX and shifts the burden of context handling to OpenAI’s internal tracking via `previous_response_id`.
- *Tool Use is First Class*: Built-in tools like `web_search`, `computer_use`, `code_interpreter`, `image_generation`, `local_shell`. Additionally, it becomes clear from the output there is an orchestration layer underneath that:
  1. Executes the initial "ChatCompletions"
  2. `if tool_use and tool_name in builtin_tools:`
         `execute_tool()`
  3. Execute the next "ChatCompletions"-esque object with the results of the tool and return response     
- *Output is Modular*: The `output` array is simplified and modular - each item is a specific output unit like a `web_search_call` or a `message`.
- *Instructions*: You can provide `instructions` to the model with differing levels of authority using the `instructions` API parameter or *message* roles.
- *Statefulness*: In the response below, the `id`: `resp_684*` can be combined with a `previous_response_id` API parameter. This suggests that each request can be treated as either a stateless request or optionally with state using `previous_response_id` - where the orchestration layer traverses the list of `previous_response_id`s and constructs a "ChatCompletions"-esque object or an internal representation of the request (e.g. similar to TensorRT-LLM, SGLang, or vLLM) to be executed.

```bash
{
  "id": "resp_684ad705ab58819fa7ac5761e452ae570f432eaa3a0586e2",
  "object": "response",
  "created_at": 1749735173,
  "status": "completed",
  "background": false,
  "error": null,
  "incomplete_details": null,
  "instructions": null,
  "max_output_tokens": null,
  "model": "gpt-4.1-2025-04-14",
  "output": [
    {
      "id": "ws_684ad706fb64819fbfc89e2960848c860f432eaa3a0586e2",
      "type": "web_search_call",
      "status": "completed"
    },
    {
      "id": "msg_684ad7095308819f9906c8145cb2b57d0f432eaa3a0586e2",
      "type": "message",
      "status": "completed",
      "content": [
        {
          "type": "output_text",
          "annotations": [
            {
              "type": "url_citation",
              "end_index": 381,
              "start_index": 296,
              "title": "2025 in science",
              "url": "https://en.wikipedia.org/wiki/2025_in_science?utm_source=openai"
            },
            {
              "type": "url_citation",
              "end_index": 940,
              "start_index": 797,
              "title": "7 Positive Environmental News Stories That Give Us Hope In 2025 \u2014 Home Planet By Grove",
              "url": "https://homeplanet.grove.co/blog-posts/positive-environmental-news-stories-that-give-us-hope-in-2025?utm_source=openai"
            },
            {
              "type": "url_citation",
              "end_index": 1385,
              "start_index": 1298,
              "title": "Global Good News - Good news from around the world - 05 May 2025",
              "url": "https://new.globalgoodnews.com/index.html?utm_source=openai"
            }
          ],
          "text": "On June 11, 2025, the European Space Agency's Solar Orbiter captured the first-ever images of the Sun's south pole. This groundbreaking achievement provides unprecedented insights into the Sun's polar regions, which are crucial for understanding solar dynamics and their impact on space weather. ([en.wikipedia.org](https://en.wikipedia.org/wiki/2025_in_science?utm_source=openai))\n\nAdditionally, in May 2025, a significant environmental milestone was reached when the European Union passed a law aimed at restoring 20% of the EU's land and sea areas by 2030, with the goal of restoring all degraded ecosystems by 2050. This initiative addresses the alarming decline of Europe's habitats, with more than 80% reportedly in poor condition, and represents a major step toward environmental recovery. ([homeplanet.grove.co](https://homeplanet.grove.co/blog-posts/positive-environmental-news-stories-that-give-us-hope-in-2025?utm_source=openai))\n\nFurthermore, in April 2025, a study found that older individuals who use smartphones and other digital devices experience lower rates of cognitive decline. This research challenges previous concerns about technology's impact on mental health and suggests that digital engagement may have protective effects against cognitive deterioration in older adults. ([new.globalgoodnews.com](https://new.globalgoodnews.com/index.html?utm_source=openai))\n\nThese developments highlight significant progress in scientific exploration, environmental conservation, and public health. "
        }
      ],
      "role": "assistant"
    }
  ],
  "parallel_tool_calls": true,
  "previous_response_id": null,
  "reasoning": {
    "effort": null,
    "summary": null
  },
  "service_tier": "default",
  "store": true,
  "temperature": 1.0,
  "text": {
    "format": {
      "type": "text"
    }
  },
  "tool_choice": "auto",
  "tools": [
    {
      "type": "web_search_preview",
      "search_context_size": "medium",
      "user_location": {
        "type": "approximate",
        "city": null,
        "country": "US",
        "region": null,
        "timezone": null
      }
    }
  ],
  "top_p": 1.0,
  "truncation": "disabled",
  "usage": {
    "input_tokens": 310,
    "input_tokens_details": {
      "cached_tokens": 0
    },
    "output_tokens": 314,
    "output_tokens_details": {
      "reasoning_tokens": 0
    },
    "total_tokens": 624
  },
  "user": null,
  "metadata": {}
}⏎
```

Output:

```bash
curl "https://api.openai.com/v1/responses" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $OPENAI_API_KEY" \
    -d '{
    "model": "gpt-4.1",
    "input": "Can you summarize in headlines?",
    "previous_response_id": "resp_6840ca1f146081a09facc01cc098b8a70b640c996ab34e9c"
}'

{
  "id": "resp_684add336d4481a085f7760dc1cc43a40b640c996ab34e9c",
  "object": "response",
  "created_at": 1749736755,
  "status": "completed",
  "background": false,
  "error": null,
  "incomplete_details": null,
  "instructions": null,
  "max_output_tokens": null,
  "model": "gpt-4.1-2025-04-14",
  "output": [
    {
      "id": "msg_684add340c8081a0b1583b169ac726de0b640c996ab34e9c",
      "type": "message",
      "status": "completed",
      "content": [
        {
          "type": "output_text",
          "annotations": [],
          "text": "Certainly! Here are the positive news stories from today, summarized in headlines:\n\n**1. Senegalese Communities Restore Vital Mangrove Ecosystems, Boosting Biodiversity and Climate Action**  \n**2. Dutch Advertising Agencies Join 'Fossil No Deal,' Refuse Collaboration with Fossil Fuel Companies**\n\nLet me know if you\u2019d like more positive headlines!"
        }
      ],
      "role": "assistant"
    }
  ],
  "parallel_tool_calls": true,
  "previous_response_id": "resp_6840ca1f146081a09facc01cc098b8a70b640c996ab34e9c",
  "reasoning": {
    "effort": null,
    "summary": null
  },
  "service_tier": "default",
  "store": true,
  "temperature": 1.0,
  "text": {
    "format": {
      "type": "text"
    }
  },
  "tool_choice": "auto",
  "tools": [],
  "top_p": 1.0,
  "truncation": "disabled",
  "usage": {
    "input_tokens": 283,
    "input_tokens_details": {
      "cached_tokens": 0
    },
    "output_tokens": 72,
    "output_tokens_details": {
      "reasoning_tokens": 0
    },
    "total_tokens": 355
  },
  "user": null,
  "metadata": {}
}⏎
```

#### Streaming

Previous ChatCompletions with `"stream": true`:

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {"role": "user", "content": "Tell me a joke."}
    ],
    "stream": true
  }'
```

Output:

```bash
data: {"id":"chatcmpl-Bhclm7gmRtX7lyRL30jphQGOQnsVK","object":"chat.completion.chunk","created":1749736834,"model":"gpt-4o-2024-08-06","service_tier":"default","system_fingerprint":"fp_07871e2ad8","choices":[{"index":0,"delta":{"role":"assistant","content":"","refusal":null},"logprobs":null,"finish_reason":null}]}

data: {"id":"chatcmpl-Bhclm7gmRtX7lyRL30jphQGOQnsVK","object":"chat.completion.chunk","created":1749736834,"model":"gpt-4o-2024-08-06","service_tier":"default","system_fingerprint":"fp_07871e2ad8","choices":[{"index":0,"delta":{"content":"Why"},"logprobs":null,"finish_reason":null}]}

<omitted for brevity...>

data: {"id":"chatcmpl-Bhclm7gmRtX7lyRL30jphQGOQnsVK","object":"chat.completion.chunk","created":1749736834,"model":"gpt-4o-2024-08-06","service_tier":"default","system_fingerprint":"fp_07871e2ad8","choices":[{"index":0,"delta":{},"logprobs":null,"finish_reason":"stop"}]}

data: [DONE]
```

Input:

```bash
curl "https://api.openai.com/v1/responses" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $OPENAI_API_KEY" \
        -d '{
        "model": "gpt-4.1",
        "input": "Can you summarize in 3 words?",
        "previous_response_id": "resp_6840ca1f146081a09facc01cc098b8a70b640c996ab34e9c",
        "stream": true
    }'

```

Output:

```bash
event: response.created
data: {"type":"response.created","sequence_number":0,"response":{"id":"resp_684adddd9c0481a0a6eca05f4081217f0b640c996ab34e9c","object":"response","created_at":1749736925,"status":"in_progress","background":false,"error":null,"incomplete_details":null,"instructions":null,"max_output_tokens":null,"model":"gpt-4.1-2025-04-14","output":[],"parallel_tool_calls":true,"previous_response_id":"resp_6840ca1f146081a09facc01cc098b8a70b640c996ab34e9c","reasoning":{"effort":null,"summary":null},"service_tier":"auto","store":true,"temperature":1.0,"text":{"format":{"type":"text"}},"tool_choice":"auto","tools":[],"top_p":1.0,"truncation":"disabled","usage":null,"user":null,"metadata":{}}}

event: response.in_progress
data: {"type":"response.in_progress","sequence_number":1,"response":{"id":"resp_684adddd9c0481a0a6eca05f4081217f0b640c996ab34e9c","object":"response","created_at":1749736925,"status":"in_progress","background":false,"error":null,"incomplete_details":null,"instructions":null,"max_output_tokens":null,"model":"gpt-4.1-2025-04-14","output":[],"parallel_tool_calls":true,"previous_response_id":"resp_6840ca1f146081a09facc01cc098b8a70b640c996ab34e9c","reasoning":{"effort":null,"summary":null},"service_tier":"auto","store":true,"temperature":1.0,"text":{"format":{"type":"text"}},"tool_choice":"auto","tools":[],"top_p":1.0,"truncation":"disabled","usage":null,"user":null,"metadata":{}}}

event: response.output_item.added
data: {"type":"response.output_item.added","sequence_number":2,"output_index":0,"item":{"id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","type":"message","status":"in_progress","content":[],"role":"assistant"}}

event: response.content_part.added
data: {"type":"response.content_part.added","sequence_number":3,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"part":{"type":"output_text","annotations":[],"text":""}}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":4,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":"Re"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":5,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":"fore"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":6,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":"station"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":7,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":","}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":8,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":" sustainability"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":9,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":","}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":10,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":" commitment"}

event: response.output_text.delta
data: {"type":"response.output_text.delta","sequence_number":11,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"delta":"."}

event: response.output_text.done
data: {"type":"response.output_text.done","sequence_number":12,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"text":"Reforestation, sustainability, commitment."}

event: response.content_part.done
data: {"type":"response.content_part.done","sequence_number":13,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"part":{"type":"output_text","annotations":[],"text":"Reforestation, sustainability, commitment."}}

event: response.output_item.done
data: {"type":"response.output_item.done","sequence_number":14,"output_index":0,"item":{"id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","type":"message","status":"completed","content":[{"type":"output_text","annotations":[],"text":"Reforestation, sustainability, commitment."}],"role":"assistant"}}

event: response.completed
data: {"type":"response.completed","sequence_number":15,"response":{"id":"resp_684adddd9c0481a0a6eca05f4081217f0b640c996ab34e9c","object":"response","created_at":1749736925,"status":"completed","background":false,"error":null,"incomplete_details":null,"instructions":null,"max_output_tokens":null,"model":"gpt-4.1-2025-04-14","output":[{"id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","type":"message","status":"completed","content":[{"type":"output_text","annotations":[],"text":"Reforestation, sustainability, commitment."}],"role":"assistant"}],"parallel_tool_calls":true,"previous_response_id":"resp_6840ca1f146081a09facc01cc098b8a70b640c996ab34e9c","reasoning":{"effort":null,"summary":null},"service_tier":"default","store":true,"temperature":1.0,"text":{"format":{"type":"text"}},"tool_choice":"auto","tools":[],"top_p":1.0,"truncation":"disabled","usage":{"input_tokens":285,"input_tokens_details":{"cached_tokens":0},"output_tokens":9,"output_tokens_details":{"reasoning_tokens":0},"total_tokens":294},"user":null,"metadata":{}}}
```

Insights:

With the traditional ChatCompletions API:
- Streaming is handled via delta chunks of text with a flat structure.
- Each chunk represents part of a single, flat message.
- There is no semantic-ness to the events - i.e. `created`, `in_progress`, `completed`
- The response completes when finish_reason is "stop" or via the sentinal value `[DONE]`.
- Example event:

```bash
data: {"choices":[{"delta":{"content":"Why"}}]}
```

With the new OpenAI Responses API:
- Streaming is event-driven, and uses semantic events instead of just deltas.
- Events are structured, meaning they encode not only raw text deltas but also:
  - lifecycle events (response.created, response.in_progress, response.completed)
  - tool calls (tool_call.started, tool_call.completed)
  - content structure (response.output_item.added, response.output_text.delta)
- It allows partial delivery of different types of content: tool invocations, messages, annotations, media, etc.
- The structure is hierarchical:
  - response → output_item → content_part → delta
- Example event:

```bash
event: response.content_part.done
data: {"type":"response.content_part.done","sequence_number":13,"item_id":"msg_684addde2f2881a0ae9d83a13ca16c140b640c996ab34e9c","output_index":0,"content_index":0,"part":{"type":"output_text","annotations":[],"text":"Reforestation, sustainability, commitment."}}
```

### Proposal

The goal is to prototype and incrementally evolve a local Responses API that mimics the OpenAI Responses API as closely as possible. This enables NIM/Dynamo to use a Responses API as a frontend endpoint and power any workflows that utilize the `/v1/responses` API.

This proposal outlines several key components necesssary to emulate OpenAI Responses API behaviors and how the Dynamo http frontend/backend might implement them.

#### Statefulness

OpenAI Responses API supports optional stateful behavior via the `previous_response_id` parameter. It appears the backend reconstructs message history from the `response_id` chain, allowing a stateless API surface but with internal state tracking and continuity.

To suppport this in Dynamo, we could:
- Store each generated `Response` object in a `History`-esque struct
- Introduce a recursive history resolution function that walks `previous_response_id` until the base message.
- Construct `ChatCompletionRequestMessage` list from each stored `Response` object
- Introduce trait `ConversationHistoryProvider` with `get`, `put`, and `reconstruct_history(id)` methods` for different forms of persistence - e.g. in-memory, disk, database, etc.


```rust
#[derive(Clone)]
pub struct AppState {
    pub openai_base_url: String,
    pub openai_api_key: String,
    pub http_client: Client,
    pub conversation: Conversation,
}

#[derive(Clone)]
pub struct Conversation {
    pub history: InMemoryHistory,
}

#[async_trait]
pub trait HistoryProvider {
    async fn get(&self, id: &str) -> Option<Response>;
    async fn insert(&self, id: &str, response: Response);
}

pub struct InMemoryHistory {
    map: Arc<DashMap<String, Response>>,
}
```

#### Semantic Events / SSE Streaming

Unlike ChatCompletions which streams raw text deltas, the Responses API emits structured event: types including lifecycle (response.created, response.completed), hierarchical content (output_item.added, output_text.delta), and toolchain awareness (tool_call.started).

- Currently we do not have structured event streams in this codebase.
- We can possibly leverage the `Annotation` in Dynamo or SSE streaming in axum:

```rust
// Streaming mode: call OpenAI ChatCompletion with stream:true
let stream = stream_chat_completion();
let sse = Sse::new(stream).keep_alive(
    axum::response::sse::KeepAlive::new()
        .interval(Duration::from_secs(1))
        .text("keep-alive-text"),
);
sse.into_response()
```

#### Built-in tools/Agentic API primitives

Eventually, we can introduce BYOT (Bring-Your-Own-Tools) with MCP servers. This is currently out of scope for the current proposal and deserves its own proposal.
