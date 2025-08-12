# LLM Request Migration on Engine Failure

**Status**: Draft | **Under Review** | Approved | Replaced | Deferred | Rejected

**Authors**: [@nnshah1](https://github.com/nnshah1) [@kthui](https://github.com/kthui)

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [@grahamking](https://github.com/grahamking) [@ryanolson](https://github.com/ryanolson)

**Required Reviewers**: [@grahamking](https://github.com/grahamking)

**Review Date**: July 21 2025

**Pull Request**: https://github.com/ai-dynamo/enhancements/pull/25

**Implementation PR / Tracking Issue**: https://github.com/ai-dynamo/dynamo/pull/1930

# Summary

Enable Request Migration at the LLM Frontend for requests that failed to generate due to a loss of
connection between the LLM Frontend and the Generate Engine.

# Motivation

There are many different kinds of faults that can cause the loss of an Engine node. For instance,
the node may be unplugged or disconnected by mistake, or some hardware may fail because of prolonged
use. Since an Engine node is rarely left idle, when a fault happens, the node will be processing
some requests. The loss of the Engine node will cause those requests to fail, which:

* Requires additional steps for users to restart their requests
* Wastes the compute resources that generated the response up to the failure point
* Takes a longer time overall for users to get their response

This ultimately leads to poor user experience.

## Goals

* A migrated request should continue the response from where it failed.
* A migration should be seamless so that the user is unlikely to notice any difference.

### Non Goals

* Prevent faults.
* Improve performance.

## Requirements

N/A

# Proposal

## Overview

The LLM Request Migration feature introduces a retry mechanism at the LLM Frontend that allows
requests to continue processing on different engine workers when connection failures occur. The
solution implements a configurable migration limit and tracks partial responses to enable seamless
continuation of generation from the point of failure.

## Architecture Components

### 1. Migration Limit Configuration

A new `--migration-limit` command-line flag is added across all engine types (VLLM, SGLang,
TensorRT-LLM) that specifies the maximum number of times a request can be migrated to another
worker. This parameter is:

- Configurable per model registration with a default value of 0 (no migration)
- Validated to be used only on worker nodes, not ingress nodes
- Propagated through the entire pipeline from launch flags to the migration stage

**Design Considerations:**

- **Engine Override Capability**: The design allows for future engine implementations to override
  the user-specified migration limit. This is important because certain engines or models may not
  generate properly with the current migration mechanism, requiring the engine to disable or reduce
  migration attempts to ensure correctness.

- **Multi-Model Flexibility**: Since a frontend may serve multiple models, setting the migration
  limit at the model level provides users with the flexibility to configure different migration
  limits for different models based on their specific characteristics and reliability requirements.

### 2. Migration Pipeline Stage

A new `Migration` pipeline stage is introduced in the LLM processing pipeline that:

- Sits at the Frontend between the request tokenization and routing/networking stages.
- Implements retry logic for both new request failures and mid-stream disconnections
- Tracks partial responses and maintains request state for seamless continuation
- Handles two types of failures:
  - **New Request Migration**: When initial connection to a worker fails (NATS "No responders"
    error)
  - **Ongoing Request Migration**: When an existing stream disconnects mid-generation ("Stream
    ended before generation completed" error)

### 3. Request State Tracking

The migration system tracks request state by:

- Maintaining the original `PreprocessedRequest` with accumulated token IDs
- Appending newly generated tokens to the request's token_ids vector
- Using this accumulated state to enable continuation from the failure point when migrating to a
  new worker

### 4. Pipeline Integration

The migration stage is integrated into both chat completions and completions pipelines:

```
Frontend ➡️ Preprocessor ➡️ Backend ➡️ Migration ➡️ ServiceBackend
                                                                   ⬇️ (generate)
Frontend ⬅️ Preprocessor ⬅️ Backend ⬅️ Migration ⬅️ ServiceBackend
```

The bidirectional integration ensures that:
- Forward path: Requests are processed through the migration layer before reaching workers
- Backward path: Responses are tracked, and migration decisions are made based on response
  patterns

## Migration Operation

### Migration Triggering Scenarios

Two distinct error scenarios trigger migration:

1. **NATS NoResponders Error**: Indicates that a specific worker instance is unavailable
   - Triggers immediate retry with remaining migration count
   - Continues until a worker becomes available or retries are exhausted

2. **Stream Disconnection Error**: Indicates mid-generation connection loss
   - Captures partial response state before failure
   - Creates new stream with accumulated context
   - Continues generation from the failure point

### Retry Logic

The `RetryManager` implements retry logic that handles different failure scenarios
through a comprehensive retry mechanism. Based on comprehensive test cases, the system handles
three primary failure modes:

#### 1. No Migration Needed (Normal Operation)
- **Scenario**: All requests succeed without any worker failures
- **Behavior**: Single stream processes all responses successfully
- **Migration Count**: No retries consumed
- **Outcome**: Seamless generation with no user-visible delays

#### 2. New Request Migration (Initial Connection Failure)
- **Scenario**: Worker becomes unreachable when creating initial connection
- **Failure Pattern**: NATS "No responders available" error on first stream creation attempt
- **Retry Logic**:
  1. RetryManager detects NatsNoResponders error during initial stream creation
  2. Decrements remaining migration count and attempts new worker selection
  3. Creates fresh stream with original request (no partial state to preserve)
  4. Successfully processes all responses on retry
- **Key Insight**: The original request state is preserved since no generation has occurred yet

#### 3. Ongoing Request Migration (Mid-Stream Disconnection)
- **Scenario**: Connection lost during active generation after partial responses received
- **Failure Pattern**: "Stream ended before generation completed" error mid-stream
- **Retry Logic**:
  1. First stream successfully delivers partial responses
  2. RetryManager detects stream disconnection error and captures partial state
  3. Updates original request's token_ids vector with received tokens
  4. Creates new stream with accumulated context
  5. New worker continues generation from where the previous worker left off
  6. Delivers remaining responses to complete the generation
- **Key Insight**: Request state accumulation enables seamless continuation without restarting

#### Token State Tracking

The migration system maintains request continuity by:
- Starting with initial prompt tokens from the original request
- Appending successful generation tokens to the token_ids vector
- Using the accumulated token count to determine the continuation position for new streams
- Ensuring no token duplication or loss during worker transitions

## Benefits

1. **Seamless User Experience**: Users experience no interruption during worker failures
2. **Resource Efficiency**: Partial generations are preserved and continued rather than restarted
3. **Fault Tolerance**: System continues operating during individual worker failures
4. **Configurable Behavior**: Migration limits allow fine-tuning based on deployment needs

# Alternate Solutions

N/A
