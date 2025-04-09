# Dynamo Enhancement Proposals

**Status**: Draft

**Authors**: [nnshah1](https://github.com/nnshah1)

**Category**: Process

**Replaces**: N/A

**Replaced By**: N/A

**Implementation**: TBD

# Summary

A standard process and format for proposing and capturing
architecture, design and process decisions for the Dynamo project
along with the motivations behind those decisions. We adopt a similar
process as adopted by Kubernetes, Rust, Python, and Ray broadly
categorized as "enhancement proposals".

# Motivation

With any software project but especially agile, open source projects
in the A.I space architecture, design, and process decisions are made
rapidly and for specific reasons which can sometimes be difficult to
understand after the fact. 

For Dynamo in particular many teams and community members are
collaborating for the first time and have varied backgrounds and
design philosophies. The Dynamo project's code base itself reflects
multiple previously independent code bases integrated quickly to meet
overall project goals. 

As the Triton Inference Serving project continues to grow in scope we need a way to capture architecture, design and process decisions in a consistent, light weight, maintainable way. 

As the project evolves we need a way to propose, ratify and capture
architecture, design and process decisions transparently and quickly
in a consistent, lightweight, maintainable way.

Borrowing from the motivation for KEPs:

> The purpose of the KEP process is to reduce the amount of "tribal knowledge" in our community. By moving decisions from a smattering of mailing lists, video calls and hallway conversations into a well tracked artifact, this process aims to enhance communication and discoverability.

Borrowing from the motivation for Rust RFCs:

> The freewheeling way that we add new features to Rust has been good for early development, but for Rust to become a mature platform we need to develop some more self-discipline when it comes to changing the system. This is a proposal for a more principled RFC process to make it a more integral part of the overall development process, and one that is followed consistently to introduce features to Rust.


## Goals

    * Design records and the process of writing and approving them should first and foremost encourage the thoughtful evaluation of design, process, and architecture choices and lead to timely decisions with a clear record of what was decided, why, and what other options were considered. 

	* Lightweight and Scalable
	  
	  The format and process should be applicable both to small / medium sized changes as well as large ones. The process should not impede the rate of progress but serve to provide timely feedback, discussion and ratification on key proposals. The process should also support retroactive documents to capture and explain decisions already made.
	  
	* Combine aspects of requirements documents, design documents and software architecture documents into a single document.
	  
	  Give one place to understand the motivation, requirements, and design of a feature or process.
	
	* Support process, architecture and guideline decisions
	
	  Have a single format to articulate decisions that effect process (such as github merge rules or templates) as well as code and design guidelines as well as features. 

    * Should be relatively clear when a document is required and when the review needs to be completed and by whom
	
    * Should allow for easy collaboration and communication between authors and reviewers

    * Format and process should be flexible enough to be used for different types of decisions requiring different levels of detail and formatting of sections.

	  	  
## Non Goals

   * DEPs do not take the place of other forms of documentation such as user / developer facing documentation (including architecture documents, api documentation)
   * Prototyping and early development are not gated by design / architectural approval.
   * DEPs should not be a perfunctory process but lead to discussion and thought process around good designs. 
   * Not all changes (bug fixes, documentation improvements) need a DEP - and can be reviewed via tht normal GitHub pull request


# Proposal

# Implementation Details

## Deferred to Implementation

# Implementation Phases

# Related Proposals

# Alternatives

# Background

## References

## Terminology & Definitions

## Acronyms & Abbreviateions


