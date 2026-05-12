# Dedicated Data Synthesis Tool

**Status**: Draft

**Authors**: @alexbowe @debermudez

**Category**: Architecture

**Replaces**: [Link of previous proposal if applicable]

**Replaced By**: [Link of previous proposal if applicable]

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: 21/04/2026

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

Data synthesis is an active area of research, and complex in its own right.
It is also broadly useful

# Motivation

**\[Required\]**

Describe the problem that needs to be addressed with enough detail for
someone familiar with the project to understand. Generally one to two
short paragraphs. Additional details can be placed in the background
section as needed. Cover **what** the issue is and **why** it needs to
be addressed. Link to github issues if relevant.

Today, most tools can sample a few prewired distributions and limited conditionals, without a scalable way to specify realistic multidimensional workloads.

To fix that, I'm proposing a dedicated dataset synthesis tool built around a declarative YAML DSL that separates specification from synthesis implementation. 

Users would describe fields, derived fields, relationships, constraints, and target distributions in a stable format. The tool would then fit a generator to that spec using a (well-maintained) non-gradient optimizer.

The DSL will support:

defining fields and derived fields (e.g. io_ratio = isl/osl)
marginal targets such as percentiles, mean/std, named parametric distributions, empirical distributions, or exact values
multidimensional relationships via histograms, conditionals, and other extensible forms
constraints such as min/max bounds
user-friendly report generation and reproducibility


The synthesis engine would:

fit per-field samplers for non-derived fields
fit inter-field dependencies
support multiple optimization algorithms, with auto selection
expose a simple effort/quality control with a sane default runtime


Additional modes for synthesizing LLM prompts (by selection, blocks of tokens based on hash sequences, or eventually a generative model) will also be added.


## Goals

List out any additional goals in bullet points. Goals may be aspirational / difficult to measure but guide the proposal.

* Create a standalone tool for data synthesis that composes into existing workflows.

* Define a declaritive and sufficiently expressive dataset specification language to serve as the interface. This need not be complete, but should be extensible so we can add additional mathematical notations later. The dream is that this would support _any_ mathematical definition.

* Allow for different data synthesis engines, with an initial implementation. This should allow for the implementation of future state-of-the-art.

* Provide analysis tools to measure how well a dataset fits to a spec.

* Serialize to a stable intermediate representation. Benchmark runners can either import them (we provide a library to do so), or convert them to their preferred format.

* Relocate existing data synthesis functionality available in AIPerf.

### Non Goals

**\[Optional \- if not applicable omit\]**

List out any items which are out of scope / specifically not required in bullet points. Indicates the scope of the proposal and issue being resolved.

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
