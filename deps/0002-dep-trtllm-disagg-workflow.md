# Prefill->Decode Disaggregated Workflow for TensorRT-LLM 

**Status**: 

**Authors**: Tanmay Verma

**Category**: Architecture

**Replaces**: N/A 

**Replaced By**: N/A 

**Sponsor**: @richardhuo-nv @nnshah1 @nvrohanv

**Required Reviewers**: @richardhuo-nv @nnshah1

**Review Date**: 07/15/2025

**Pull Request**: [#22](https://github.com/ai-dynamo/enhancements/pull/22)

**Implementation PR / Tracking Issue**: [#1884](https://github.com/ai-dynamo/dynamo/pull/1884)

# Summary

Currently, the disaggregated workflow examples require requests to first hit the decode worker and then the prefill worker. This DEP proposes an alternative approach that allows users to perform prefill operations on the first worker followed by decode operations on the second worker, providing more flexibility in workflow orchestration.

# Motivation

**\[Required\]**

Dynamo users have expressed strong interest in gaining control over request flow patterns. The current TensorRT-LLM disaggregated workflow routes requests first to a TensorRT-LLM worker, which then forwards them to a Prefill worker for remote prefill execution. This design constraint stems from the current limitation where only one worker per model can interact with the KV router(call `register_llm`), allowing KV routing to either Prefill workers or Decode workers depending on which receives the request first. In the existing implementation, users are restricted to KV routing on decode workers only. Providing flexibility to choose which worker type handles the initial request routing would address critical user requirements for workflow customization.

In the current implementation, the TensorRT-LLM prefill worker transfers all KV cache blocks to the decode workers, regardless of whether the decode workers already have a complete KV cache match. This means that, even with a 100% KV cache hit on the decode side, the prefill worker still sends all blocks, resulting in redundant data transfer. By routing requests to the prefill worker first when using KV routing, we can optimize efficiency and reduce unnecessary work during the prefill stage.

## Goals

**\[Optional \- if not applicable omit\]**

List out any additional goals in bullet points. Goals may be aspirational / difficult to measure but guide the proposal. 

### Short-Term Goal

* Goal Allow users to swap the order of prefill/decode workers in TensorRT-LLM disaggregated workflows in the short-term

* Goal Maintain feature parity between both orchestration methods (prefill-first and decode-first workflows)


### Long-Term Goal

* Propose a long-term goal of maintaining a unified disaggregated workflow. 

**\[Optional \- if not applicable omit\]**

List out any items which are out of scope / specifically not required in bullet points. Indicates the scope of the proposal and issue being resolved.

## Requirements

**\[Optional \- if not applicable omit\]**

List out any additional requirements in numbered subheadings.

**\<numbered subheadings\>**

### REQ \<1\> \<Option to control prefill-first or decode-first\>
User **MUST** be able to control whether the request flows from Prefill->Decode or Decode->Prefill.

### REQ \<2\> \<Both the workflows should be properly tested\>
We **SHOULD** properly test both the workflows.


# Proposal

**\[Required\]**

Current request disaggregated workflow looks like:

![Current Disaggregated Workflow](0001_images/current.png)

To unblock users in short-term, we are proposing an additional option (--disaggregation-strategy={prefill_first, decode_first}) to the worker script. The two workers are:
- First stage worker
- Second stage worker

Based on the cli option, the first stage worker can decide whether it wants to run prefill locally and then push the request to second stage for decode. Or forward the request to second stage for remote prefill and run decode locally. 

![Proposed Disaggregated Workflow](0001_images/proposed_design.png)


This should address the requirement in short-term. 

## Long-Term

We strongly believe that it should not be the workers job to determine what and where to perform execution. The KV router should be smart enough to handle prefill and decode worker request routing.
The long-term proposal is a WIP. 

# Alternate Solutions

N/A