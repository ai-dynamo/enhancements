# Container Build Strategy 

**Status**: Draft

**Authors**: [nv-tusharma/Dynamo Ops team] 

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: saturley-hall, nv-anants

**Required Reviewers**: nnshah1, saturley-hall, nv-anants, nvda-mesharma, mc-nv, dmitry-tokarev-nv, pvijayakrish

**Review Date**: 2025-06-03

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

This document outlines a container strategy for Dynamo to further improve the build process by focusing on reducing build times and improving developer experience with pre-built Dynamo containers. With these goals in mind, This DEP covers various optimizations to the build process, including: 
- Restructuring the build process to provide a build-base container with Dynamo pre-built, enabiling specific backends to use the build base container to build the final container image.
- Improve CI registry service to provide pre-built Dynamo containers for developers to use from their CI pipelines. This registry should be available to external contributors as well.  
- Discuss possible further improvements including external caching to reduce build times. 

# Motivation

Dynamo is primarily built from a collection of Dockerfiles hosted in the /containers directory of the[Dynamo repository](https://github.com/NVIDIA/Dynamo). These Dockerfiles build Dynamo along with the specific backend (vLLM, TRT-LLM, etc) and NIXL, the high-throughput, low-latency point to point communication library used by the Dynamo to accelerate inference. This approach has several drawbacks, including:
1. Build times are long because all Dynamo, NIXL, and the selected backend are built each time. This is especially problematic as we continue to add more backends to Dynamo as we have to rebuilt the entire stack each time. 
2. Developers need to build Dynamo per code change, which is time consuming and results in a poor developer experience. 
3. Currently, we do not have a way to provide pre-built Dynamo containers to external contributors. The Github registry is not performant enough for public CI use-cases.


## Goals

* Reduce build times for Dynamo by providing a build-base container with Dynamo pre-built. This build-base container can be used by specific backends to build the final container image.

* Provide a CI registry service to provide pre-built Dynamo containers for developers to use from the CI pipelines. This registry should be available to external contributors as well. The registry should be fast and reliable. 

* Outline possible further improvements including external caching/multi-context docker builds to reduce build times. 

### Non Goals

- Slim backend-specific runtime containers to use for performance testing. 
- Unified build environment


## Requirements

**\[Optional \- if not applicable omit\]**

List out any additional requirements in numbered subheadings.

**\<numbered subheadings\>**

### REQ \<\#\> \<Title\>

Describe the requirement in as much detail as necessary for others to understand it and how it applies to the DEP. Keep in mind that requirements should be measurable and will be used to determine if a DEP has been successfully implemented or not.

Requirement names should be prefixed using a monotonically increasing number such as “REQ 1 \<Title\>” followed by “REQ 2 \<Title\>” and so on. Use title casing when naming requirements. Requirement names should be as descriptive as possible while remaining as terse as possible.

Use all-caps, bolded terms like **MUST** and **SHOULD** when describing each requirement. See [RFC-2119](https://datatracker.ietf.org/doc/html/rfc2119) for additional information.


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

**\[Optional \- if not applicable omit\]**

Add additional context and references as needed to help reviewers and authors understand the context of the problem and solution being proposed.

## References

**\[Optional \- if not applicable omit\]**

Add additional references as needed to help reviewers and authors understand the context of the problem and solution being proposed.

* \<hyper-linked title of an external reference resource\>

## Terminology & Definitions

**\[Optional \- if not applicable omit\]**

List out additional terms / definitions (lexicon). Try to keep definitions as concise as possible and use links to external resources when additional information would be useful to the reader.

Keep the list of terms sorted alphabetically to ease looking up definitions by readers.

| \<Term\> | \<Definition\> |
| :---- | :---- |
| **\<Term\>** | \<Definition\> |

## Acronyms & Abbreviations

**\[Optional \- if not applicable omit\]**

Provide a list of frequently used acronyms and abbreviations which are uncommon or unlikely to be known by the reader. Do not include acronyms or abbreviations which the reader is likely to be familiar with.

Keep the list of acronyms and abbreviations sorted alphabetically to ease looking up definitions by readers.

Do not include the full definition in the expanded meaning of an abbreviation or acronym. If the reader needs the definition, please include it in the [Terminology & Definitions](#terminology--definitions) section.

**\<Acronym/Abbreviation\>:** \<Expanded Meaning\>

