# Enable Capture of Request/Response with Metadata Enrichment when store=true

**Status**: Draft

**Authors**: Ryan Lempka

**Category**: Architecture

**Sponsor**: [TBD]

**Required Reviewers**: [TBD]

**Pull Request**: https://github.com/ai-dynamo/dynamo/compare/main...rlempka/log-stdout-store-true

# Summary

Introduce structured JSON logging of inference requests and responses to stdout when explicitly enabled. This provides an initial mechanism for customers to capture model input/output for evaluation, distillation, and auditability.

# Motivation

Customers increasingly want access to raw request/response data to improve their own models, perform fine-tuning, or run safety/eval pipelines. OpenAI supports this via `store=true` and retrieval of stored completions; Azure documents the same feature (https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/stored-completions?tabs=python-secure).

Providing a similar capability in Dynamo allows:
- Reproducibility of inference.
- Collection of datasets for distillation into smaller models.
- Auditing and evaluation of tool use and safety guardrails.

This proposal starts with the simplest possible approach—structured stdout logs—so existing cluster log agents (e.g. Fluent Bit) can ship data with no new infrastructure.

## Goals

* Provide a gated mechanism (`store=true` + `DYN_AUDIT_STDOUT=1`) to emit structured JSONL to stdout.
* Ensure metadata passthrough is supported for enrichment.
* Establish a forward-compatible schema for downstream collection.
* Lay groundwork for streaming capture, sidecar ingestion, and future external targets.

### Non Goals

* This proposal does **not** define storage backends (MongoDB, S3, etc.).
* This proposal does **not** include PII redaction or advanced filtering (future work).
* This proposal does **not** introduce customer-visible APIs beyond existing `store=true`.

# Proposal

When both of the following are true:
- The request contains `store=true`.
- The environment variable `DYN_AUDIT_STDOUT=1` is set.

Then Dynamo will emit a single JSON line containing:
- Timestamps
- Request/response payloads (non-streaming)
- Model name
- Request ID
- Arbitrary passthrough metadata

The output format is JSONL, one record per line. Example:

    {
      "log_type": "audit",
      "ts_ms": 1757115457130,
      "request_id": "req_123",
      "endpoint": "/v1/chat/completions",
      "model": "meta/llama3-8b-instruct",
      "store": true,
      "metadata": {"ticket":"SN-1234"},
      "input": {"messages":[{"role":"user","content":"..."}]},
      "output": {"choices":[{"message":{"role":"assistant","content":"..."}}]},
      "usage": {"prompt_tokens":123, "completion_tokens":456, "total_tokens":579}
    }

# Implementation Phases

## Phase 1 — Stdout capture (non-streaming)

**Work Item(s):** https://github.com/ai-dynamo/dynamo/compare/main...rlempka/log-stdout-store-true

**Supported API / Behavior:**
- `store=true` + `DYN_AUDIT_STDOUT=1` → one JSONL log line to stdout.

**Not Supported:**
- Streaming responses
- Alternative sinks
- Redaction

## Phase 2 — Streaming support

Enable same logging for streaming completions. Each streamed chunk is logged once the response is complete.

## Phase 3 — Sidecar example

Publish reference sidecar to consume stdout logs and persist to a DB. MongoDB is a good default open source target for MVP.

## Phase 4 — Alternative targets (future DEP)

Evaluate support for outbound sinks (webhooks, S3, etc.). Requires follow-on DEP. Design choice will be whether to:
- Keep Dynamo stdout-only and push complexity into sidecars (`LOG_WRITE_TARGET` there), or
- Add `DYN_STORE_TARGET` in Dynamo itself.

# Alternate Solutions

## Alt 1 — Direct DB integration in Dynamo

**Pros:**
- Single component, fewer moving pieces.

**Cons:**
- Couples Dynamo to specific storage backends.
- Harder to operate in Kubernetes-native environments.

**Reason Rejected:** Too heavyweight; stdout is simpler and aligns with K8s best practices.

# Background

This proposal aligns with industry precedent:
- OpenAI `store=true` support
- Azure OpenAI stored completions (https://learn.microsoft.com/en-us/azure/ai-foundry/openai/how-to/stored-completions?tabs=python-secure)

Future enhancements may add PII redaction, truncation caps, and backpressure handling.
