# Discovery, Configuration, Request, and Event Plane Infrastructure

**Status**: Draft

**Authors**: 

**Category**: Architecture

**Replaces**: N / A

**Replaced By**: N / A

**Sponsor**: neelays

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Proposal on how to abstract / replace Dynamo runtime infrastructure. 

# Motivation

When deploying Dynamo customers often have their own requirements on
infrastructure that can and can not be used and that must be deployed
and maintained. In particular use of `nats` or `etcd` may be a blocking
factor for customer adoption. In order to make Dynamo easy for
customers to adopt we need to explore ways to abstract usage of
infrastructure. 

## Goals

* Catalog everywhere we use `etcd` and NATS in Dynamo and how we use it.

* Proposal on how to remove / abstract / replace NATs and `etcd` for Discovery, Configuration, Request, and Event Plane Infrastructure

* Default and recommended path by which a user deploys Dynamo does not require NATs and Etcd.


# Proposal

> WIP 

## Deploying Dynamo in Kubernetes

## Deploying Dynamo outside Kubernetes

# Alternate Solutions

N / A

# Background

## Where and How Nats and `etcd` are Used in Dynamo

### etcd

`etcd` is an open source, distributed reliable, hierarchical key-value
store written in go and providing a gRPC interface. In addition to
generic key / value storage `etcd` provides the ability to `watch` for
changes within a prefix hierarchy as well as associate `leases` with
keys such that keys will expire if they are not refreshed withing a
specified time frame. `etcd` also provides a single source of truth
such that keys have a single value and transactions support that can
for example check for existance and create a key or return an existing
value.

#### Endpoint Instance Discovery

Dynamo uses `etcd` to register new worker instances with their
configuration so that the `frontend` and any clients such as the
`router` or a `decode worker` can:

a. Get events when endpoint `instances` are added or removed by `watching` a well known prefix (`instances/namespace/component/endpoint`).

b. Get information for how to reach `endpoint instances` via `nats` topics. `nats` topics are 1:1 with endpoint instances.

```
 {
    "key": "instances/dynamo/backend/clear_kv_blocks:694d98147d54bfde",
    "create_revision": 510,
    "mod_revision": 510,
    "version": 1,
    "value": "{\n  \"component\": \"backend\",\n  \"endpoint\": \"clear_kv_blocks\",\n  \"namespace\": \"dynamo\",\n  \"instance_id\": 7587888160958627806,\n  \"transport\": {\n    \"nats_tcp\": \"dynamo_backend.clear_kv_blocks-694d98147d54bfde\"\n  }\n}",
    "lease": 7587888160958627806
  },

```

#### Model / Worker Discovery

The frontend additionally:

b. Watches for `models` and uses model configuration to determine the model and model type. This includes the `component` and `endpoint` to use to watch for `instances`. Models also contain runtime configuration for load balancing used by the `router`

```
  {
    "key": "models/be57d697-d049-4fc3-b243-0069cc28ed8c",
    "create_revision": 509,
    "mod_revision": 509,
    "version": 1,
    "value": "{\n  \"name\": \"Qwen/Qwen3-0.6B\",\n  \"endpoint\": {\n    \"namespace\": \"dynamo\",\n    \"component\": \"backend\",\n    \"name\": \"generate\"\n  },\n  \"model_type\": \"Backend\",\n  \"runtime_config\": {\n    \"total_kv_blocks\": 24064,\n    \"max_num_seqs\": 256,\n    \"max_num_batched_tokens\": 2048\n  }\n}",
    "lease": 7587888160958627806
  }
```

c. Retrieves the `mdc` which contains additional  prepocessing instructions as well a tokenizer config.

```
{
    "key": "mdc/qwen_qwen3-0_6b",
    "create_revision": 508,
    "mod_revision": 508,
    "version": 1,
    "value": "{\"display_name\":\"Qwen/Qwen3-0.6B\",\"slug\":\"qwen_qwen3-0_6b\",\"model_info\":{\"hf_config_json\":\"nats://0.0.0.0:4222/qwen_qwen3-0_6b/config.json\"},\"tokenizer\":{\"hf_tokenizer_json\":\"nats://0.0.0.0:4222/qwen_qwen3-0_6b/tokenizer.json\"},\"prompt_formatter\":{\"hf_tokenizer_config_json\":\"nats://0.0.0.0:4222/qwen_qwen3-0_6b/tokenizer_config.json\"},\"gen_config\":{\"hf_generation_config_json\":\"nats://0.0.0.0:4222/qwen_qwen3-0_6b/generation_config.json\"},\"last_published\":null,\"context_length\":40960,\"kv_cache_block_size\":16,\"migration_limit\":0}"
  }
```
#### Worker Port Conflict Resolution 

Used to ensure unique ports for processes on the same host:

```
  {
    "key": "dyn://dynamo/ports/10.20.56.81/23739",
    "create_revision": 506,
    "mod_revision": 506,
    "version": 1,
    "value": "{\"worker_id\": \"vllm-backend-dp0\", \"reason\": \"zmq_kv_event_port\", \"reserved_at\": 1756927703.2237012, \"pid\": 17404}",
    "lease": 7587888160958627806
  },

```

#### Router State Sharing Synchronization
 
Used as a distributed lock to snapshot radix tree to nats.

Used to discover routers and cleanup router consumers in jetstream. 

```
  {
    xo"key": "kv_routers/dynamo/backend/27cc845d-5b93-494f-8978-af45d0ffba39",
    "create_revision": 544,
    "mod_revision": 544,
    "version": 1,
    "value": "{\n  \"overlap_score_weight\": 1.0,\n  \"router_temperature\": 0.0,\n  \"use_kv_events\": true,\n  \"router_replica_sync\": true,\n  \"max_num_batched_tokens\": 8192,\n  \"router_snapshot_threshold\": 10000,\n  \"router_reset_states\": false\n}",
    "lease": 7587888160958627942
  },
```

## NATS

NATS is a message oriented middleware that supports publish /
subscribe with subject / topic bsaed addressing, persistence and
extactly once semantics with JetStream, and Key Value object storage.

### Services and Endpoint Routing (NATS Service + NATS core)

Dynamo components use the NATS services api
[https://docs.nats.io/using-nats/developer/services] to create
automatic message topics for a group of endpoints. These include
automatic creaton of `PING` (health) `STATS` (metrics) and `INFO`
(discovery). 

While NATS `services` offer these - Dynamo uses the NATS services api
only for `STATS` and for the easy creation of message topics for each
worker instance. While NATS services can do load balancing, discovery
and health checks Dynamo creates unique service endpoints for each
`instance`.

Each `instance` has its own `queue` that it receives requests from. 

Note: requests are all directed and load balanced by the Dynamo itself
and not directly using load balancing of NATs. Requests are also
pushed into the worker as fast as they arrive so NATs is used
primarily as an addressing scheme and not a message queue.

`push_endpoint.rs`

Dynamo uses the `STATS` broadcast feature of NATS whereby a
request can be sent to all services to collect their stats at once.

Endpoints operate in a request / response mode. 

Responses are passed back over TCP / IP. 

### KV Event Publishing (Jet Stream)

Workers publish kv events over ZMQ and these are then forwarded to a
persistent, durable, stream and then consumed by all `routers` in the
system.

### KV Radix Tree Storage (Object Store / Jet Stream)

KV Router periodically persists RADIX tree state into NATS object store. 

### Tokenizer Storage (Object Store / Jet Stream)

### Router Sync (Nats Core)

Router instances sync with each other using component event publishing.

```
// for inter-router comms
pub const PREFILL_SUBJECT: &str = "prefill_events";
pub const ACTIVE_SEQUENCES_SUBJECT: &str = "active_sequences_events";
```

### Load / Forward Pass Metrics (Nats Core)

Forward pass metrics published over NATS - as well as available via
`load_metrics` endpoint.

### Service Info

```
Service Information

          Service: dynamo_backend (okiyFEOD9hlENGX2twGyxV)
      Description: Dynamo component backend in namespace dynamo
          Version: 0.0.1

Endpoints:

               Name: dynamo_backend-clear_kv_blocks-694d98147d54c0a8
            Subject: dynamo_backend.clear_kv_blocks-694d98147d54c0a8
        Queue Group: q
  
               Name: dynamo_backend-load_metrics-694d98147d54c0a8
            Subject: dynamo_backend.load_metrics-694d98147d54c0a8
        Queue Group: q
  
               Name: dynamo_backend-generate-694d98147d54c0a8
            Subject: dynamo_backend.generate-694d98147d54c0a8
        Queue Group: q

Statistics for 3 Endpoint(s):

  dynamo_backend-clear_kv_blocks-694d98147d54c0a8 Endpoint Statistics:

           Requests: 0 in group "q"
    Processing Time: 0s (average 0s)
            Started: 2025-09-04 15:15:11 (1m28s ago)
             Errors: 0

  Endpoint Specific Statistics:

    null

  dynamo_backend-load_metrics-694d98147d54c0a8 Endpoint Statistics:

           Requests: 0 in group "q"
    Processing Time: 0s (average 0s)
            Started: 2025-09-04 15:15:11 (1m28s ago)
             Errors: 0

  Endpoint Specific Statistics:

    {
        "worker_stats": {
            "data_parallel_rank": 0,
            "request_active_slots": 0,
            "request_total_slots": 256,
            "num_requests_waiting": 0
        },
        "kv_stats": {
            "kv_active_blocks": 0,
            "kv_total_blocks": 24064,
            "gpu_cache_usage_perc": 0.0,
            "gpu_prefix_cache_hit_rate": 0.0
        },
        "spec_decode_stats": null
    }

  dynamo_backend-generate-694d98147d54c0a8 Endpoint Statistics:

           Requests: 0 in group "q"
    Processing Time: 0s (average 0s)
            Started: 2025-09-04 15:15:11 (1m28s ago)
             Errors: 0

  Endpoint Specific Statistics:

    null
```

```
╭───────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                      Streams                                                      │
├──────────────────────────────────────────────┬─────────────┬─────────────────────┬──────────┬──────┬──────────────┤
│ Name                                         │ Description │ Created             │ Messages │ Size │ Last Message │
├──────────────────────────────────────────────┼─────────────┼─────────────────────┼──────────┼──────┼──────────────┤
│ namespace-dynamo-component-backend-kv-events │             │ 2025-09-03 16:35:56 │ 0        │ 0 B  │ never        │
╰──────────────────────────────────────────────┴─────────────┴─────────────────────┴──────────┴──────┴──────────────╯
```

```
╭────────────────────────────────────────────────────────╮
│                     Bucket Contents                    │
├────────────────────────┬─────────┬─────────────────────┤
│ Name                   │ Size    │ Time                │
├────────────────────────┼─────────┼─────────────────────┤
│ config.json            │ 726 B   │ 2025-09-04 15:15:30 │
│ tokenizer_config.json  │ 9.5 KiB │ 2025-09-04 15:15:30 │
│ tokenizer.json         │ 11 MiB  │ 2025-09-04 15:15:31 │
│ generation_config.json │ 239 B   │ 2025-09-04 15:15:31 │
╰────────────────────────┴─────────┴─────────────────────╯
```

## etcd:
Discovery:
     + Frontend discovers the existence of workers and learns the capabilities (which model they serve, whether they want pre-processing, etc).
     + Large files (tokenizer.json) will be represented by a nats object store URL - see below.
     + An etcd lease ensures cleanup on shutdown.
     + The /health URL uses etcd to list available instances, same objects as discovery.
KV data persistence (WIP): https://github.com/ai-dynamo/dynamo/pull/2756
grpc/kserve has an etcd client, check with Guan.
## nats:
Message queue frontend -> backend
     + Migration/failover code is NATS specific, catches errors the message queue.
Object store for model config: tokenizer and the other json files. I'm hoping we can replace this with modelexpress.
Service metrics for KV routing: KV events are requested/pulled from workers.
System status uses the nats service API similar to KV events. It also gathers component metrics. Check with Keiven.



### Nats

## References

* https://github.com/etcd-io/etcd

* https://nats.io/

* https://docs.nats.io/using-nats/developer/services

* https://github.com/nats-io/nats-architecture-and-design/blob/main/adr/ADR-32.md

* https://docs.nats.io/nats-concepts/core-nats/queue

* https://docs.rs/async-nats/latest/async_nats/service/struct.Service.html
