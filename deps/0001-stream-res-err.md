# Error Propagation during Response Streaming

**Status**: Approved 

**Authors**: [@nnshah1](https://github.com/nnshah1) [@kthui](https://github.com/kthui)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [@ryanolson](https://github.com/ryanolson) [@grahamking](https://github.com/grahamking)

**Required Reviewers**: [@ryanolson](https://github.com/ryanolson) [@grahamking](https://github.com/grahamking)

**Review Date**: Jul 09 2025

**Pull Request**: https://github.com/ai-dynamo/enhancements/pull/15

**Implementation PR / Tracking Issue**: https://github.com/ai-dynamo/dynamo/pull/1671

# Summary

Network level errors may occur while streaming responses from the server back to the client. If they
occur, these errors must be made available to the client response stream listener.

# Motivation

The client response stream listener currently does not know why a stream is closed. Normally, a
stream is closed because the server has finished producing all the responses, but it can also be due
to failure of the server or the network connecting the client to the server.

Knowing why the stream is closed is vital to the ability to detect faults while the server is
streaming responses back to the client.

For instance, in the current
[router implementation](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/egress/addressed_router.rs#L165-L174),
if it is unable to restore the bytes back to the original object, it cannot relay the error back to
the client response stream consumer for proper handling, and instead it silently skips the response
that failed to restore.

## Goals

* Update the Response Streaming Interface to Support Propagating Network/Router Level Errors.

### Non Goals

* Mechanism for Detecting Network/Router Level Errors - i.e. Incomplete Stream Detection.

## Requirements

N/A

# Proposal

The client starts its RPCs from
[this `generate` method](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/egress/addressed_router.rs#L85),
which returns a stream in `Result<ManyOut<U>, Error>` type.

**Case 1**: The server is unreachable when the request is made

This case is handled by the existing `Result<...>` wrapper around the `ManyOut<U>` stream type.

**Case 2**: The server is disconnected while responses are being returned

The `ManyOut<U>` stream yields responses in the `U` type, defined by the client, that is opaque to
the network/router layer, so currently there is no way for the network/router layer to
1. Propagate errors detected at its level back to the client while streaming responses; and
2. Know if the `U` type is passing an error from the server back to the client.

The proposal will introduce a required `MaybeError` trait that the `U` type must implement, which
brings in methods for the network/router layer to accomplish the above two points:
```rust
pub trait MaybeError {
    /// Construct an instance from an error.
    fn from_err(err: Box<dyn std::error::Error>) -> Self;

    /// Construct into an error instance.
    fn err(&self) -> Option<Box<dyn std::error::Error>>;
}
```

When the network/router detects there is an error, a new error response
```rust
let error_response = U::from_err(...);
```
is constructed and returned to the response stream for propagating it back to the client.

For detecting if the server sent an error response at the network/router level
```rust
let response = response_stream.next();
if let Some(err) = response.err() {
    ...
}
```
The network/router can act on the error if needed and also propagate the error back to the client.

## Example: End-of-Stream Detection

End-of-Stream can be detected by wrapping each response type `U` in a new
```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct NetworkStreamWrapper<U> {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<U>,
    pub complete_final: bool,
}
```
such that each response is sent in the `data` field with `complete_final` set to `false` from the
server. Once the generate engine is done with producing responses, the server must send an extra
`NetworkStreamWrapper` response with no `data` and `complete_final` set to `true`.

At the client, the `NetworkStreamWrapper` is torn down and its `data` is yielded back to the stream
consumer. If the network stream ended before a response with `complete_final = true` is received, an
extra error response is yielded back to the stream consumer propagating the error back as described
by the [proposal](#proposal).

**Note**
* The `NetworkStreamWrapper` is only used while transmitting bytes over the network. It is wrapped
immediately before serializing responses into bytes and unwrapped immediately after deserializing
bytes into responses.
* The end-of-stream event can be detected via other methods, such as Server-Sent Events (SSE), and
the actual implementation is free to use any other detection methods as long as the interface
remains the same as described by the [proposal](#proposal).

# Alternate Solutions

## Alt 1 Handle Errors at the Client Implementation Layer

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

### Future Enhancements

#### Extension to NvExt

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

**Pros:**

* No change to the current network/router interface, as the `U` opaque type is retained.

**Cons:**

* It is cumbersome for each and every client implementation to implement the same basic error
detection and reporting mechanism that can be easily done at the network/router layer.

**Reason Rejected:**

* Network error handling should be done by the network/router layer.

**Notes:**

* Original design
[does NOT intend to handle error](https://github.com/ai-dynamo/enhancements/pull/15#pullrequestreview-2954974976)
at the network/router layer.

## Alt 2 Add a Fault Tolerance Layer on top of the current Router Layer

Add a FaultTolerance Layer that implements the `Result<...>` wrapper that will become the `U` type
at the current Router Layer. The FaultTolerance Layer should implement the same
[`PushWorkHandler`](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network.rs#L323)
trait and accepts objects implementing the
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

## Alt 3 Wrap Each Response in Result<U, anyhow::Error>

### Part 1: Add a New Generate Method for Propagating Error from Network/Router Layer

The client starts its RPCs from
[this `generate` method](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/egress/addressed_router.rs#L85),
which returns a stream in `Result<ManyOut<U>, Error>` type.

**Case 1**: The server is unreachable when the request is made

This case is handled by the existing `Result<...>` wrapper around the `ManyOut<U>` stream type.

**Case 2**: The server is disconnected while responses are being returned

The `ManyOut<U>` stream yields responses in the `U` type, defined by the client, that is opaque to
the network/router layer, so currently there is no way for the network/router layer to propagate any
error back to the client while streaming responses.

The proposal is to wrap the `U` type in a `Result<...>` wrapper, similar to how the stream is
currently wrapped, so errors detected at the network/router level can be propagated to the client.

To avoid disrupting existing behaviors, a new
```rust
async fn generate_with_error_detection(&self, request: SingleIn<AddressedRequest<T>>) -> Result<ManyOut<Result<U, Error>>, Error>
```
method is to be added alongside the existing `generate` method as a part of the
[`AddressedPushRouter` struct](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/egress/addressed_router.rs#L59).
The change can be progressively applied to existing implementations, by switching over from the
existing `generate` method to the new `generate_with_error_detection` method.

### Part 2: Implement End of Stream Detection into Network/Router Layer

The server handles RPCs with the
[`PushWorkHandler` trait](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/ingress/push_handler.rs#L20C24-L20C39),
specifically each response over the stream goes through
[here](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/ingress/push_handler.rs#L100-L109).
This part of the code can reliably capture the end of stream signal, which will be made available to
the client over the RPC.

Instead of sending each response from the worker `generate` method directly back to the client, each
response is contained in a new
```rust
#[derive(Serialize, Deserialize, Debug)]
pub struct StreamItemWrapper<U> {
    #[serde(skip_serializing_if = "Option::is_none")]
    pub data: Option<U>,
    pub complete_final: bool
}
```
where `data` holds the response and `complete_final` indicates if this is the last response.

While the worker's `generate` method is producing new responses, each response is sent with the
`complete_final = false`. After the last response is produced by the worker's `generate` method, an
extra response with no data and `complete_final = true` is sent to the client, before ending the
RPC.

The client receives and processes responses at the
[end of its `generate` method](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/egress/addressed_router.rs#L163-L174).
The response is first reconstructed into the `StreamItemWrapper<U>`, and then the `data` is
extracted and yielded back to the client.

There are two cases of connection stream error detectable by the client's `generate` method at the
network/router layer:

**Case 1**: Connection stream ended before `complete_final`

An `Err(...)` response, as discussed in
[Part 1](#part-1-add-a-new-generate-method-for-propagating-error-from-networkrouter-layer), is
yielded and then the response stream is closed.

**Case 2**: `complete_final` is received and then more responses arrive

An `Err(...)` response, as discussed in
[Part 1](#part-1-add-a-new-generate-method-for-propagating-error-from-networkrouter-layer), is
yielded upon arrival of the first response after the response with `complete_final`. After the
`Err(...)` response is yielded, the response stream is closed.

The `Err(...)`s will be constructed using
[`anyhow::Error::msg`](https://docs.rs/anyhow/1.0.98/anyhow/struct.Error.html#method.msg).
