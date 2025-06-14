# Wrap Streaming Response in Result<U, E>

**Status**: **Draft** | Under Review | Approved | Replaced | Deferred | Rejected

**Authors**: [@nnshah1](https://github.com/nnshah1) [@kthui](https://github.com/kthui)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [@ryanolson](https://github.com/ryanolson) [@grahamking](https://github.com/grahamking)

**Required Reviewers**: [@ryanolson](https://github.com/ryanolson) [@grahamking](https://github.com/grahamking)

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

Error occured while streaming responses from server back to client must be relayed back to the
client response stream listener.

# Motivation

The client response stream listener currently does not know why a stream is closed. Normally, a
stream is closed due to the server has done with producing all the responses, but it can also due to
failure of the server or the network connecting the client to the server.

Knowing why the stream is closed at the network/router layer is vital to the ability of detecting
fault while the server is streaming response back to the client. 

For instance, in the current
[router implementation](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/egress/addressed_router.rs#L165-L174),
if it is unable to restore the bytes back to the original object, it cannot relay the error back to
the client response stream consumer for proper handling, and instead it silently skips the response
that failed to restore.

## Goals

N/A

### Non Goals

N/A

## Requirements

N/A

# Proposal

Each response streamed to be wrapped in a `Result<U, E>`, where `U` is the response type and `E` is
the error type, for propagating error occured while streaming responses from the server back to the
client response stream listener.

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
defined:
```
 0: stream incomplete
```
and the `message` is optional and currently unused.

At the end of the stream, the server will stream an extra error response to the client, with bit 0
set, indicating the end of stream. If the client picks up the error response from the server with
the end of stream bit set and then the stream ends, the client ends its response stream. If the
stream ends without the error response with end of stream bit set, the client yields an extra error
response with the stream incomplete bit set to its response stream, and then ends its response
stream.

At the client response stream consumer, if all the responses are Ok, then there is no error. If a
response is Err, then an error occured while responses are being streamed from the server to the
client, and the error can be determined by checking the `StreamError.flags`.

The server response stream producer does not implement the `StreamError`, because it is used
exclusively for handling network/router layer error. Any error at the server above the
network/router layer should be handled within the opaque response type `U`.

# Alternate Solutions

## Alt 1 Handle Errors at the Client Implementation Layer

The opaque response type defined by the client implementation can always start with a `Result<...>`
wrapper, such that in `ManyOut<U>`, `U = Result<...>`.

**Pros:**

* No change to the current network/router interface, as the `U` opaque type is retained.

**Cons:**

* It is cumbersome for each and every client implementations to implement the same basic error
detection and reporting mechanism that can be easily done at the network/router layer.

**Reason Rejected:**

* This should be done at the network/router layer.

**Notes:**

N/A
