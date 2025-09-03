# Dynamo Integration with the Gateway API Inference Extension's End Point Picker 

**Status**: Draft 

**Authors**: [Anna Tchernych](https://github.com/atchernych)

**Category**: Architecture

**Replaces**: [Link of previous proposal if applicable] 

**Replaced By**: [Link of previous proposal if applicable] 

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request](https://github.com/ai-dynamo/dynamo/pull/2786)

# Summary

This proposal outlines the integration of Dynamo components with the [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/), in particular the [EndPointPicker](https://gateway-api-inference-extension.sigs.k8s.io/guides/epp-configuration/config-text/) 

# Motivation


The prior [version of Dynamo Inference Gateway Integration](https://github.com/ai-dynamo/enhancements/blob/bis/inference-gw/0001-TBD-inference-gw-integration/0001-TBD-inference-gw-integration.md) leaves room for 2 enhancements to how Dynamo integrates with the EPP.

## HTTP call overhead

First, the EPP sends and HTTP request to the Dynamo FrontEnd to obtain the target worker instance id and the tokens. Even though the FrontEnd is deployed as a sidecar, this approach introduces additional latency. Given how well optimized the Dynamo system is, this latency would offset the performance gains provided by the highly efficient Dynamo Router.

## EPP offers richer routing control.

Second, even though the Dynamo Routing call is implemented as a plugin in accordance with the plugin interface EPP provides, Dynamo cannot support other EPP-provided routing mechanisms such as Routing Filters. EPP offers more flexibility to the end user on how to route and provides a nice declarative configuration yaml file.

For example, in Dynamo router there is a small chance that not the optimal worker will be picked:
 We can have A perfect KV match on an overloaded worker  but the request on this worker may be slower than a near-miss on an idle one.

### Worker A (Overloaded + Perfect Cache)
- **overlap** = 100% → `prefill_blocks = 0`  
- **active_blocks** = 1000 (very busy)  
- **Cost** = `1.0 × 0 + 1000 = 1000`  

### Worker B (Idle + Near Miss)
- **overlap** = 80% → `prefill_blocks = 20% of request`  
- **active_blocks** = 10 (idle)  
- **Cost** = `1.0 × (0.2 × request_size) + 10`  

If the request is small enough, Worker A could still win despite being overloaded.
The current model only uses active_blocks but ignores the queue depth (num_requests_waiting)


This situation can be mitigated by setting temperature but not eliminated. 
We can add the Queue aware penalty to our router. Alternatively, we can use an EPP filter **LowQueueFilter**. It enforces a hard ceiling on queue depth and will drop overloaded pods from the router consideration. 

## Goals 

* Expose Dynamo Router as a a Library for Go through c-bindings or a Binary Library crate.

* Provide support for EPP standard Routing filters, Pickers and Scorers so that an end user can mix an match Dynamo Routing approach with the EPP filters, pickers and scorers through a single yaml - based config file.

* Modularize Dynamo to enable Inference Gateway API usage with workers, without relying on the Dynamo Router.

### Non Goals

* Change existing Dynamo worker interfaces significantly
* Change existing Dynamo worker interfaces significantly

## Requirements

List out any additional requirements in numbered subheadings.

---

### REQ-1: Router as a Library
The router **SHOULD** incur minimal latency overhead when used as a library.  

Use all-caps, bolded terms like **MUST** and **SHOULD** when describing each requirement. See [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119) for additional information.

---

### REQ-2: Support for Standard EPP Filters
The user **SHOULD** be able to use standard EPP filter- and scorer-enabled routing **alongside** the Dynamo Router.

---

### REQ-3: Inference Gateway API Without Dynamo Router
The user **SHOULD** be able to use Dynamo workers directly, without requiring the Dynamo Router.


# Proposal

Below proposes the C-Bindings (FFI Layer) as a solution to the latency problem. 
Massaging of Dynamo metrics is proposed as a solution to enabling EPP standard filters.

# Implementation Details

## Library Implementation

Two approaches are possible:

### 1. Rust Binary Crate
Package the Dynamo Router as a Rust binary crate.  
- Invoke it from Go using `exec.Command`.  
- Communicate over **stdin/stdout/IPC**.  

This approach is simpler but introduces higher latency since every call crosses a **process boundary**.

---

### 2. C-Bindings (FFI Layer)

Client Request → Gateway API → Endpoint Picker → C Bindings → KV Router → Best Worker

Add calls to existing C-bindings and invoke them from Go (`cgo → extern "C" Rust`).  

- Expose a C-compatible FFI layer using `#[no_mangle] extern "C" fn`.  
- Build the crate into a `.so` / `.a` / `.dll` and call it from Go via `cgo`.  
- See [Draft PR #2786](https://github.com/ai-dynamo/dynamo/pull/2786) for a reference implementation.  

This approach has **higher maintenance overhead** but offers **lower latency**:  
- Avoids process boundary overhead (no syscalls, no kernel, no sockets).  
- Zero/low copy: pass `*uint32 + length` for tokens, get back a pointer, and free with a matching `free`—cheap and predictable.  
- Shared runtime: initialize the Rust tokenizer/router runtime once (bindings already use `OnceCell`) and reuse it across calls.  


## Enabling EPP filters

EPP filters rely on Prometheus metrics. We would have to make them available  in the *InferencePool* CR. 
We would have to rename the metrics of interest to EPP. For example, dynamo exposes the `num_requests_waiting` metric for the number of requests that have been routed to a specific worker but are waiting in that worker's internal queue to be processed. But EPP filters expect the `vllm:num_requests_waiting` and `nv_trt_llm_request_metrics{request_type=waiting}`.  For KV cache utilization EPP expects `vllm:gpu_cache_usage_perc` and `nv_trt_llm_kv_cache_block_metrics{kv_cache_block_type=fraction}`. 
It is also notable that the tokenization can be performed in a plugin to the Inference Gateway API BBR (Body Based Routing). At the time of the proposal tokenization is coupled with the routing and tokens are returned to the Gateway along with the worker instance id. 

EPP expects the following metrics:

LeastKVCacheFilter and LowQueueFilter (kvCacheUsagePercentageMetric):
For vLLM: vllm:gpu_cache_usage_perc
For TRTLLM: nv_trt_llm_kv_cache_block_metrics{kv_cache_block_type=fraction}
SGLang: sglang:token_usage

LeastQueueFilter (totalQueuedRequestsMetric):
For vLLM: vllm:num_requests_waiting
For TRTLLM: nv_trt_llm_request_metrics{request_type=waiting}
SGLang: sglang:num_queue_reqs



LoRA: (Dynamo does not yet support)
--loraInfoMetric="vllm:lora_requests_info"



The names can be configurable via env vars:
TOTAL_QUEUED_REQUESTS_METRIC="your_queue_metric_name"
KV_CACHE_USAGE_PERCENTAGE_METRIC="your_kv_cache_metric_name"
LORA_INFO_METRIC="your_lora_metric_name"



## Exposing Dynamo Workers Without the FrontEnd

**Flow:**  
`Client Request → Gateway API → Endpoint Picker → Best Worker`

This option relies on the routing choices provided by the **Standard Endpoint Picker**.

---

### Components Needed

1. **Keep Dynamo Runtime**
   - **etcd**
   - **NATS**

2. **Create an HTTP service in front of each worker**
   - Purpose: translate incoming requests into the worker’s **Dynamo endpoint call** and stream the response back.
   - The service should also subscribe to the same events via NATS.

   **Implementation options:**
   - **SGLang:** extend  
     `components/backends/sglang/src/dynamo/sglang/main.py`
   - **vLLM / TRT-LLM:** create a new service  
     `lib/llm/src/http/gateway_sidecar.rs`  
     or a lightweight Python equivalent similar to `main.py`

   **The HTTP service must expose standardized endpoints:**
   - `/ready`
   - `/health`
   - `/metrics`

   > Ensure `/health`, `/ready`, and `/metrics` return expected schemas.  
   > Expose worker metrics such as **queue depth** and **resource usage**.

3. **Deployment**
   - Deploy each worker with a **sidecar HTTP service**.
   - Configure the **InferencePool** to select these services.


## Deferred to Implementation

**\[Optional \- if not applicable omit\]**

List out items that are under discussion but that will be resolved only during implementation / code review. 


**Release Target**: Date

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to github issues\>

**Supported API / Behavior:**

* \<name and concise description of the API / behavior\>

**Not Supported:**

* \<name and concise description of the API / behavior\>

# Related Proposals


* [Biswa's Proposal](https://github.com/ai-dynamo/enhancements/blob/bis/inference-gw/0001-TBD-inference-gw-integration/0001-TBD-inference-gw-integration.md)



# Alternate Solutions

**\[Required, if not applicable write N/A\]**

N/A

## Alt \<\1\> \<TODO\>

**Pros:**

TODO

**Cons:**

TODO

**Reason Rejected:**

TODO


# Background

## 1. Main Routing Approaches: Queue Depth & KV-Cache Utilization
Both **Dynamo** and **EPP** can route requests using **queue depth** and **KV-cache pressure**.

- **Token awareness**  
  - EPP’s default approach is token-aware only *by approximation* because it relies on the **non-tokenized text** in the prompt to keep things generic.  
  - Dynamo, by contrast, uses a **token-aware KV algorithm**.  
  - The Dynamo Router runs the model’s tokenizer on the prompt, ensuring:  
    - Consistent tokenization with the serving runtime  
    - Fewer mismatches between routing and inference behavior  

---

## 2. EPP Pipeline Shape: *Filter → Score → Pick*
EPP’s architecture stages routing decisions explicitly:

- **Filters** enforce hard constraints and shrink the candidate set before ranking.  
  *Example: drop pods with queue depth above a threshold, or require LoRA affinity.*  

- **Scorers** express preferences across the remaining pods.  
  *Example: prefer lower queue depth or lower KV usage.*  

- **Pickers** select the final endpoint.  
  *Example: max-score, round-robin, prefix-aware, etc.*  

---

### Standard Filters Available in EPP
EPP provides a variety of filters that can be composed via YAML configuration:

- **DecisionTreeFilter** — try one path; if it fails, fall back to another.  
- **LeastKVCacheFilter** — keep only pods in the lowest KV-usage bucket.  
- **LeastQueueFilter** — keep only pods in the lowest queue-depth bucket.  
- **LoRAAffinityFilter** — prefer/require pods with the target LoRA already loaded.  
- **LowQueueFilter** — enforce a hard ceiling on queue depth (drop overloaded pods).  

---

### Declarative Configuration
EPP policies are defined in **YAML**.  
For example, a `DecisionTreeFilter` can encode logic like:  

> *“If LoRA is hot, continue; else, fall back to a low-queue path.”*  

This makes policies flexible, modular, and easy to express declaratively.




## References

- [Deep Dive into Inference Extensions](https://kgateway.dev/blog/deep-dive-inference-extensions/) — good introduction  
- [Smarter AI with Kubernetes Gateway API](https://kgateway.dev/blog/smarter-ai-reference-kubernetes-gateway-api/) — good overview read  
- [Gateway API Guides](https://gateway-api.sigs.k8s.io/guides/) — official guides  
- [Inference Extension Overview](https://gateway-api-inference-extension.sigs.k8s.io/gieps/overview) — official docs  
- [kGateway Documentation](https://kgateway.dev/docs/) — official docs  
- [kGateway API (gateway_extensions_types.go)](https://github.com/kgateway-dev/kgateway/blob/36969220f2cf95b262b881e52f68ae882671825d/api/v1alpha1/gateway_extensions_types.go#L19) — source code reference  
- [Inference Extension Plugins (EPP)](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/pkg/epp) — plugin implementations  
- [EPP Protocol Proposal (Dynamo fork)](https://github.com/atchernych/gateway-api-inference-extension-dynamo/tree/main/docs/proposals/004-endpoint-picker-protocol) — proposal document  
- [Gateway API Inference Extension (upstream source)](https://github.com/kubernetes-sigs/gateway-api-inference-extension) — main repository  


## Terminology & Definitions

| Term | Definition |
| :---- | :---- |
| **BBR — Body-Based Routing** | A [Gateway API Inference Extension](https://gateway-api-inference-extension.sigs.k8s.io/gieps/overview) plugin that extracts fields (e.g., `model`) from the request body and inserts them into headers for routing decisions. |
| **EPP — Endpoint Picker Plugin** | A [plugin mechanism](https://github.com/kubernetes-sigs/gateway-api-inference-extension/tree/main/pkg/epp) used by the Inference Gateway API to score and select the appropriate backend endpoint (e.g., `QueueScorer`, `MaxScorePicker`, `LoRAAffinityScorer`). |


