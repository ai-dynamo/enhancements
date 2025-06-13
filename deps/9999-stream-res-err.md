# Wrap Streaming Responses in Result<U, E>

**Status**: **Draft** | Under Review | Approved | Replaced | Deferred | Rejected

**Authors**: [@kthui](https://github.com/kthui)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [@nnshah1](https://github.com/nnshah1)

**Required Reviewers**: [@ryanolson](https://github.com/ryanolson)

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

Each response streamed should be wrapped in a `Result<U, E>`, where `U` is the response type and `E`
is the error type, for propagating error occured while streaming responses from the server back to
the client response stream listener.

Change the
[`Ingress<SingleIn<T>, ManyOut<U>>`](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network/ingress/push_handler.rs#L20)
type to `Ingress<SingleIn<T>, ManyOut<Result<U, E>>>` and 
[`Egress<SingleIn<T>, ManyOut<U>>`](https://github.com/ai-dynamo/dynamo/blob/fcfc21f20e53908cedc41a91bbd594283ecf45db/lib/runtime/src/pipeline/network.rs#L246-L247)
type to `Egress<SingleIn<T>, ManyOut<Result<U, E>>>`.

The `Result<...>` wrapper around the opaque response type provides the ability for the
network/router to relay error information back to the client response stream consumer. Without the
additional wrapper, it is impossible for the network/router to yield an error response from a
stream, because the response object is opaque to the network/router.

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
