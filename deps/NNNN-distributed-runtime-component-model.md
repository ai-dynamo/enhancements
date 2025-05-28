# Distributed Runtime Component Model

**Status**: Draft

**Authors**: graham, nnshah1, biswa, ishan, maksim

**Category**: Architecture

**Replaces**: [Link of previous proposal if applicable]

**Replaced By**: [Link of previous proposal if applicable]

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary


# Motivation

The definition of a `component` and its relationship to the `runtime`
is not always clear as we move between `python` and `rust`. In
different parts of the documentation and code base similar terms such
as `client`, `worker`, `component`, `service` are used
interchangeabley which can be confusing.

We'll clarify the architecture and document the decsisions here to
help clarify and discuss any changes needed. In cases where no change
is neededd this will serve as a "retro-active" proposal.


## Goals

* Clearly define a `component`, its properties, and provide examples of `rust` and `python` components.

* Prescriptively define how components use the `runtime` to discover and access each other.

* Define how components are discovered and configured

* Define how components are identified - including namespaces

* Define how endpoints are hosted and used


### Non Goals

* Radically reset current terms and definitions if not needed



## Requirements

### REQ \<\#\> \<Title\>

TBD

# Proposal

## Namespace

A pipeline. Usually a model. Just a name.

If you run two models, that is two pipelines. An exception would be if doing speculative decoding. The draft model is part of the pipeline of a bigger model.

Examples: "llama_8b".

## Component

A load balanced service needed to run a pipeline. This typically has some configuration (which model to use, for example).

In a Prefill / Decode disaggregated setup you would have at least two components: `deepseek-distill-llama8b.prefill.generate` (possibly multiple instance of this) and `deepseek-distill-llama8b.decode.generate`.

Examples: "backend", "prefill", "decode", "preprocessor", "draft", etc.

## Endpoint

Like a URL.

Examples: "generate", "load_metrics".

The KV metrics publisher in VLLM adds a `load_metrics` endpoint to the current component. If the `llama3-1-8b.backend` component is using patched vllm it will also expose `llama3-1-8b.backend.load_metrics`.

## Instance

A process. Unique. Dynamo assigns each one a unique instance_id. The thing that is running is always an instance. Namespace/component/endpoint can refer to multiple instances.

If you run two instances of the same model ("data parallel") they are the same namespace+component+endpoint but different instances. The router will spread traffic over all the instances of a namespace+component+endpoint. If you have four prefill workers in a pipeline, they all have the same namespace+component+endpoint and are automatically assigned unique instance_ids.

## Event

## Property

## Runtime

## Request Flow

## Discovery Flow


# Alternate Solutions

**\[Required, if not applicable write N/A\]**

List out solutions that were considered but ultimately rejected. Consider free form \- but a possible format shown below.

## Alt \<\#\> \<Title\>

**Pros:**

\<bulleted list or pros describing the positive aspects of this solution\>

**Cons:**

\<bulleted list or pros describing the negative aspects of this solution\>

**Reason Rejected:**

\<bulleted list or pros describing why this option was not used\>

**Notes:**

\<optional: additional comments about this solution\>

