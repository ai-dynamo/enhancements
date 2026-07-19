
Chat w/ Graham
- Replace model deployment cards fields nats:// URLs, with model express
- Direct call frontend -> backend for critical path of requests, replacing NATS mailbox
- Metrics fetching, euh, not sure.



### transports
 - ZeroMQ
 - Raw TCP
 - ZeroRPC
 - Redis
 - Kafka
 - SQS
 - GCP PubSub
 - Azure Service Bus

### Object store
 - S3
 - Redis
 - Shared filesystem 

LLM-d KV cache manager:
https://github.com/llm-d/llm-d-kv-cache-manager/tree/main


1. 


1. Use RocksDB / local storage to persist messages on producer side to guarantee at least once delivery.
2. RocksMQ

## KV Router are stateful.
- kv routers are stateful.
- Sharding 
- Replication
- KV events are broadcasted to all routers.



# #######################  LLM

Allow mini batching of requests to the LLM.

mini batching

https://github.com/pathwaycom/pathway?tab=readme-ov-file#deployment



## Rust <> Python interoperability

using pydantic dataclasses to eliminate manual code to match rust structs to python classes


