# Highly Available and Fault-Tolerant Router

**Status**: Draft

**Authors**: @PeaBrane

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: @nnshah1

**Required Reviewers**: @ryanolson @grahamking

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

The overarching goal here is to have a Router design that allows for multiple Router instances to be deployed for fault tolerance.
That is, in case one goes down, the others will still be able to function normally.
This requires some sort of mechanism to sync the Router states periodically (either among the Routers themselves or via events from the backend engines),
and also a mechanism to "warm restart" the Router such that the Router can be brought back with up-to-date states.
Finally, the Router should be decoupled from the (http) frontend, such that the two can be scaled independently. 
(It is more likely that the frontend handling the pre-processing / tokenization would need to scale first before the Router does.)

# Motivation

As context, we have iterated over two designs of the Router that worked well in their own regard.

## Initial Design

First, we had a **near-stateless Router** listening on backend engines for KV events and load metrics. This is good because:
- Multiple Routers can be launched and synced naturally
- Easier Python binding for modular components, as the Router does not hold the output SSE stream, and simply needs to return the `best_worker_id`

But not good because:
- The radix tree of the `KvIndexer` is still very stateful, with no warm restart mechanism
- Huge performance hit under highly concurrent payloads, as KV / metric events cannot respond fast enough for the Router to keep track of the updated load states.

## Current Design

Now, we have a **stateful Router** still listening on backend engines for KV events (can opt out of via `ApproxKvIndexer`), 
but maintains the active block states locally from the request-response cycle. This is good because:
- The performance is good under high concurrency, because the Router never sees a stale load metric state, as we forced sequential processing of requests locally.
- It is highly general, as the Router can now interface with any backend engine, without the need for any event communication

But not good because:
- Due to its high statefulness, multiple Routers cannot be perfectly in sync, as a Router only sees a subset of requests / responses
- The Router holds the output SSE stream, so if the Router goes down, the stream will die along with it
- Harder to have modular components to bind to Python, as we require the entirety of `KvPushRouter` to handle the request-response cycles

## Goals

In short, a stateless Router is better for fault-tolerance, but a stateful Router is better for optimality of routing decisions.
The main motivation here is to have a design that incorporates the benefits of both, and eventually achieve a net win. 
More details would be provided in the following sections.

The overarching goals are then:
* The Router has to be performant over generic load balancers (e.g. round robin) under general settings, as it is now.
* The Router has to be a separate component that can be scaled (or not-scaled) independently from the frontend.
* Multiple Router has to be launched without losing routing optimality.
* A Router can go down without affecting the output SSE streams.
* A Router can come back up without losing its previous states or missing updates during the time it was down.

### Non Goals

N/A

## Requirements

N/A

# Proposal

**\[Required\]**

Describe the high level design / proposal. Use sub sections as needed, but start with an overview and then dig into the details. Try to provide images and diagrams to facilitate understanding.

# Implementation Details

**\[Optional \- if not applicable omit\]**

Add additional detailed items here including interface signatures, etc. Add anything that is relevant but seems more of a detail than central to the proposal. Use sub sections / bullet points as needed. Try to provide images and diagrams to facilitate understanding. If applicable link to PR.

## Deferred to Implementation

**\[Optional \- if not applicable omit\]**

List out items that are under discussion but that will be resolved only during implementation / code review. 

# Implementation Phases

**\[Optional \- if not applicable omit\]**

List out phases of implementation (can be single phase). Give each phase a monotonically increasing number; example “Phase 0” followed by “Phase 1” and so on. Give phases titles if it makes sense.

## Phase \<\#\> \<Optional Title\>

**Release Target**: Date

**Effort Estimate**: \<estimate of time and number of engineers to complete the phase\>

**Work Item(s):** \<one or more links to github issues\>

**Supported API / Behavior:**

* \<name and concise description of the API / behavior\>

**Not Supported:**

* \<name and concise description of the API / behavior\>

# Related Proposals

**\[Optional \- if not applicable omit\]**

* File

* File

* File

* File

* File

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

# Background

N/A

## References

* [KV Routing](https://docs.nvidia.com/dynamo/latest/architecture/kv_cache_routing.html)
* [KV Router Performance Tuning](https://docs.nvidia.com/dynamo/latest/guides/kv_router_perf_tuning.html)
* [SGL's stateful Router](https://lmsys.org/blog/2024-12-04-sglang-v0-4/)

## Terminology & Definitions

| \<Term\> | \<Definition\> |
| :---- | :---- |
| **KvIndexer** | A data structure for maintaining a global view of prefix caches of all workers |
| **Router** | A component for routing requests to backend workers that is aware of the current loads and prefix caches of each worker |

## Acronyms & Abbreviations

N/A
