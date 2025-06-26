# Sending Complete Final Flag with Annotated<...> Wrapper

**Status**: Draft | **Under Review** | Approved | Replaced | Deferred | Rejected

**Authors**: [@nnshah1](https://github.com/nnshah1) [@kthui](https://github.com/kthui)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [@ryanolson](https://github.com/ryanolson) [@grahamking](https://github.com/grahamking) [@oandreeva-nv](https://github.com/oandreeva-nv)

**Required Reviewers**: [@ryanolson](https://github.com/ryanolson) [@grahamking](https://github.com/grahamking) [@oandreeva-nv](https://github.com/oandreeva-nv)

**Review Date**: Jun 24 2025

**Pull Request**: https://github.com/ai-dynamo/enhancements/pull/15

**Implementation PR / Tracking Issue**: https://github.com/ai-dynamo/dynamo/pull/1671

# Summary

Network level errors may occur while streaming responses from the server back to the client. If they
occur, these errors must be made available to the client response stream listener.

# Motivation

The client response stream listener currently does not know why a stream is closed. Normally, a
stream is closed because the server has finished producing all the responses, but it can also be due
to failure of the server or the network connecting the client to the server.

Knowing why the stream is closed is vital to the ability of detecting faults while the server is
streaming responses back to the client.

## Goals

N/A

### Non Goals

N/A

## Requirements

N/A

# Proposal

The
[`Annotated<...>`](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/runtime/src/protocols/annotated.rs#L32)
wrapper is typically used by
[higher level implementations](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/bindings/python/rust/engine.rs#L145-L146)
as the opaque `U` type in the `ManyOut<U>` type returned from the
[network/router layer](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/runtime/src/pipeline/network/egress/push_router.rs#L165).

The
[`event`](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/runtime/src/protocols/annotated.rs#L38)
field in the `Annotated<...>` object will see a new string `complete_final`, in addition to the
existing `error` string, that signals the stream is completed and the response with `complete_final`
is the last response to be sent by the server.

At the server, once response generation is completed, the server MUST send an additional empty
response with the `complete_final` flag set, before closing the stream.

At the client, if the `complete_final` response arrived and then the stream ended, the client can be
assured all responses intended to be sent by the server have been received. If the stream ended
without the `complete_final` response, the client can infer that one or more responses to be sent by
the server have not arrived, which indicates some error handling needs to be performed, for
instance, returning an Error to the upper level or restarting the request at where it was left off
at another node.

Two additional methods are to be added to the `Annotated<...>` implementation
```rust
impl<R> Annotated<R> {
    ...

    /// Create a new annotated stream with complete final event
    pub fn from_complete_final() -> Self {
        Self {
            data: None,
            id: None,
            event: Some("complete_final".to_string()),
            comment: None,
        }
    }

    ...

    pub fn is_complete_final(&self) -> bool {
        self.event.as_deref() == Some("complete_final")
    }

    ...
}
```
to facilitate constructing and checking for complete final.

## Future Enhancements

### Extension to NvExt

While this proposal is intended for enhancing
[dynamo/lib/runtime](https://github.com/ai-dynamo/dynamo/tree/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/runtime)
Python binding implementation, the same idea can also be applied to
[dynamo/lib/llm](https://github.com/ai-dynamo/dynamo/tree/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/llm)
OpenAI implementation, at
[`NVExt`](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/llm/src/protocols/openai/nvext.rs#L25-L64).

An EOS (end of stream) annotation can be appended to the
[NVExt.annotations](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/llm/src/protocols/openai/nvext.rs#L63)
list, signaling that the response is the last one to be sent by the server.

Ref: https://github.com/ai-dynamo/enhancements/pull/15#issuecomment-3002343978

The
[`NvCreateChatCompletionStreamResponse`](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/llm/src/protocols/openai/chat_completions.rs#L64-L68)
struct will need to include an optional
```rust
#[serde(skip_serializing_if = "Option::is_none")]
pub nvext: Option<NvExt>
```
field, similar to the
[request struct](https://github.com/ai-dynamo/dynamo/blob/2becce569d59f8dc064c2f07b7995d1e979ade66/lib/llm/src/protocols/openai/chat_completions.rs#L37-L44),
in order to pass the flag with responses.

**Open question**: The OpenAI API includes a
["finish_reason"](https://platform.openai.com/docs/api-reference/chat-streaming/streaming)
variable in its response JSON indicating the end of stream, for example:
```json
{..., "choices":[{"index":0,"delta":{"role":"assistant","content":""},"logprobs":null,"finish_reason":null}]}
{..., "choices":[{"index":0,"delta":{"content":"Hello"},"logprobs":null,"finish_reason":null}]}
....
{..., "choices":[{"index":0,"delta":{},"logprobs":null,"finish_reason":"stop"}]}
```
Since `NvCreateChatCompletionStreamResponse` contains the full OpenAI response in its `inner`, is
the duplicate end of stream flag in `NVExt` in `NvCreateChatCompletionStreamResponse` needed?

# Alternate Solutions

## Alt 1 Handle Errors at the Client Implementation Layer

Each response streamed will be wrapped in a `Result<U, E>`, where `U` is the response type and `E`
is the error type, for propagating errors that occur while streaming responses from the server back
to the client response stream listener.

For instance, change the
[`Ingress<SingleIn<T>, ManyOut<U>>`](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/ingress/push_handler.rs#L20)
type to `Ingress<SingleIn<T>, ManyOut<Result<U, E>>>` and 
[`Egress<SingleIn<T>, ManyOut<U>>`](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network.rs#L246-L247)
type to `Egress<SingleIn<T>, ManyOut<Result<U, E>>>`.

The `Result<U, E>` wrapper around the opaque response type `U` provides the ability for the
network/router to relay error information back to the client response stream consumer. Without the
additional wrapper, it is impossible for the network/router to yield an error response from a
stream, because the response object is opaque to the network/router.

The error type `E` is
```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct StreamError {
    pub flags: u32,
    #[serde(skip_serializing_if = "Option::is_none")]
    pub message: Option<String>,
}
```
where each bit on the `flags` carries a boolean message. Currently, only the most significant bit is
used:
```
 0: stream incomplete
 1: (reserved)
...
31: (reserved)
```
The `message` is optional and currently unused.

At the end of the stream, the server will stream an extra error response to the client, with bit 0
set, indicating the end of stream. If the client picks up the error response from the server with
the end of stream bit set and then the stream ends, the client ends its response stream. If the
stream ends without the error response with end of stream bit set, the client yields an extra error
response with the stream incomplete bit set to its response stream, and then ends its response
stream.

At the client response stream consumer, if all the responses are Ok, then there is no error. If a
response is Err, then an error occurred while responses were being streamed from the server to the
client, and the error can be determined by checking the `StreamError.flags`.

The server response stream producers do not implement the `StreamError`, because it is used
exclusively for handling network/router layer errors. Any error at the server above the
network/router layer should be handled within the opaque response type `U`, for instance the
`Annotated<R>` wrapper.

**Pros:**

* Network error is detected at the network/router layer and then propagated to the upper layers.

**Cons:**

* Interface change for current implementations on top of the network/router layer.

**Reason Rejected:**

* Original design
[does NOT intend to handle error](https://github.com/ai-dynamo/enhancements/pull/15#pullrequestreview-2954974976)
at the network/router layer.

**Notes:**

N/A

## Alt 2 Add a Fault Tolerance Layer on top of the current Router Layer

Add a FaultTolerance Layer that implements the `Result<...>` wrapper that will become the `U` type
at the current Router Layer. The FaultTolerance Layer should implement the same
[`PushWorkHandler`](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network.rs#L323)
trait and accept objects implementing the
[`AsyncEngine`](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/engine.rs#L104)
trait, so it shares the same interface as the Router.

**Pros:**

* No change to the current network/router interface, as the `U` opaque type is retained.
* Fault Tolerance implementations can be added to this layer, and written in Rust.

**Cons:**

* The additional layer is overly complicated, because the FaultTolerance Layer is basically an
extension to the Router Layer without overriding any Router functionalities.

**Reason Rejected:**

* Use the more generic `Annotated<...>` wrapper.

**Notes:**

The current Python bindings can be updated from
```
Python binding    |    runtime
|                 |    |
`--> Client       |    `--> component
     |            |    |    |
     |            |    |    `--> client <.
     |            |    |                 | owns an instance of; and
     |            |    `--> pipeline     | obtains available instances from etcd and tracks/reports downed ones
     |            |         |            |
     |            |         `--> network/router
     |            |                      ^
     `-----------------------------------'
       owns an instance of
```
to
```
Python binding    |    runtime
|                 |    |
`--> Client       |    `--> component
     |            |    |    |
     |            |    |    `--> client <.
     |            |    |                 | owns an instance of; and
     |            |    `--> pipeline     | obtains available instances
     |            |         |            |
     |            |         `--> network/router <--.
     |            |         |                      | owns an instance of; and
     |            |         `--> fault_tolerance --' reports downed instances over router to client
     |            |              ^
     `---------------------------'
       owns an instance of
```
when creating a client.
