# Dynamo Enhancement Proposals

**Status**: Draft

**Authors**: [nnshah1](https://github.com/nnshah1), [ryanolson](https://github.com/ryanolson)

**Category**: Process

**Replaces**: N/A

**Replaced By**: N/A

**Implementation**: TBD

**Sponser**: [nnshah1](https://github.com/nnshah1)

**Required Reviewers**: [dzier](https://github.com/dzier), [statiraj](https://github.com/statiraj), team

**Review Date**: 2025-04-16

**Pull Request**: [PR#1](https://github.com/ai-dynamo/enhancements/pull/1)

**Issue(s)**: N/A

# Summary

A standard process and format for proposing and capturing
architecture, design and process decisions for the Dynamo project
along with the motivations behind those decisions. We adopt a similar
process as adopted by Kubernetes, Rust, Python, and Ray broadly
categorized as "enhancement proposals".

 * [Limited Template][limited]
 * [Complete Template][complete]

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

As the project evolves we need a way to propose, ratify and capture
architecture, design and process decisions quickly and thoughtfully in
a transparant, consistent, lightweight, maintainable way.

Borrowing from the motivation for KEPs:

> The purpose of the KEP process is to reduce the amount of "tribal knowledge" in our community. By moving decisions from a smattering of mailing lists, video calls and hallway conversations into a well tracked artifact, this process aims to enhance communication and discoverability.

Borrowing from the motivation for Rust RFCs:

> The freewheeling way that we add new features to Rust has been good for early development, but for Rust to become a mature platform we need to develop some more self-discipline when it comes to changing the system. This is a proposal for a more principled RFC process to make it a more integral part of the overall development process, and one that is followed consistently to introduce features to Rust.


## Goals

* **Useful** 

  Enhancement proposals and the process of writing and approving them
  should encourage the thoughtful evaluation of design, process, and
  architecture choices and lead to timely decisions with a clear
  record of what was decided, why, and what other options were
  considered.

* **Lightweight and Scalable**

  The format and process should be applicable both to small / medium
  sized changes as well as large ones. The process should not impede
  the rate of progress but serve to provide timely feedback,
  discussion and ratification on key proposals. The process should
  also support retroactive documents to capture and explain decisions
  already made.
	  
* **Single Document for Requirements and Design**

  Combine aspects of requirements documents, design documents and
  software architecture documents into a single document. Give one
  place to understand the motivation, requirements, and design of a
  feature or process.
	
* **Support Process, Architecture and Guideline Decisions**
	
  Have a single format to articulate decisions that effect process
  (such as github merge rules or templates) as well as code and
  design guidelines as well as features.

* **Clear** 

  Should be relatively clear when a document is required and when the
  review needs to be completed and by who and what the overall process
  is.
	
* **Encourage Collaboration**

  Should allow for easy collaboration and communication between
  authors and reviewers

* **Flexible**

  Format and process should be flexible enough to be used for
  different types of decisions requiring different levels of detail
  and formatting of sections.

	  	  
## Non Goals

* DEPs do not take the place of other forms of documentation such as user / developer facing documentation (including architecture documents, api documentation)
* Prototyping and early development are not gated by design / architectural approval.
* DEPs should not be a perfunctory process but lead to discussion and thought process around good designs. 
* Not all changes (bug fixes, documentation improvements) need a DEP - and many can be reviewed via tht normal GitHub pull request

# Proposal

Following successful open source projects such as [Kubernetes][kep]
and [Rust][rust-rfc] we adopt a markdown based enhancement proposal
format designed to support any decisions we need to capture as a
project. 

We will adopt an open, community-wide, discussion and comment process
using pull requests but enable `code owners` and `maintainers` to be
the final arbiters of `approval`. Subject area experts will be listed
as required approvers to ensure proposals are complete and reviewed
properly.

Enhancement proposals will be stored in github in a seperate
repository. We provide two templates [limited][limited] and
[complete][complete] where `limited` is a strict subset of `complete`
and both indicate which sections are `required` and which are
`optional`.

# Implementation Details

## Proposal Process

* Fork or create a branch in the `enhancements` repository

* Copy the [NNNN_limited_template.md][limited] or
[NNNN_complete_template.md][complete] to `deps/0000-my-feature.md`
(where `my-feature` is descriptive, don't assign an `DEP` number yet)

* Identify a `sponser` from the list of `maintainers` or
`code owners` to help with the process.

* Fill in the proposal template. Be sure to include all `required`
sections. Keep sections in the order prescribed in the template.

* Work with the `sponser` to identify the required reviewers and a
timeline for review. 

* Submit a pull request to the `enhancements` repository

* If discussion is needed the `sponser` can ask for
a slot in the weekly Engineering Sync or schedule an ad-hoc meeting
with the required reviewers.

* Iterate and incorporate feedback via the pull request.

* When review is complete The `sponser` will merge the request and update the status.

* `sponser` should assign an id 

* author and `sponser` should add issues and/or PRs as needed to track implementation


## When is a proposal required?

It is difficult to enumerate all the circumstances where a proposal
would be required or not requiured. Generally we will follow this
process when making "substantial changes". What is "substantial" is
evolving and mainly determined by the core team and community. 

When in doubt reach out to a `maintainer` or `code owner`.

Generally speaking a proposal would not be required for:

* Bug fixes that don't change advertised behavior. 

* Documentation fixes / updates.

* Minor refactors within a single module. 

Generally speaking proposals would be required for:

* New features which add significant functionality.

* Changes to existing features or code which require discussion. 

* Changes to public interfaces

* Responses to security related vulnerabilities found directly in the project code. 

* Changes to packaging and installation

* When a `maintainer` or `code owner` recommends that a change go
through the proposal process.

* Retroactively to capture current architecture, guideline or process 

## Minor Changes After Review

For minor changes / changes that are in the spirit of the review -
updates can be made to the document without a new proposal.

Example: links to implementation

## Significant Changes After Review

For significant changes - a new proposal should be made and the
original marked as replaced.

## Maintainence 

DEPs should be reviewed for updates / replace / archive on a
regular basis.

## Senstive Changes and Discussions

Certain types of changes need to be discussed and ratified before
being made public due to timing of non-disclosed information.

In such (rare) cases - drafts and reviews will be conducted offline by
`authors`, `code owners` and `maintainers` and the public proposals
updated when possible.

Example: when responding to undisclosed security vulnerabilities we
want to avoid inadvertantly encouraging zero day attacks for deployed
systems.

In such (rare) cases we may make use of a private repo on a temporary
basis to collect feedback before publishing to the public repo.

## Deferred to Implementation

* Definition of `code owners` and `maintainers`

* Whether or not to organize `deps` into sub directories for projects / areas

* Tooling around the creation / indexing of `deps`

# Alternatives

# Alternate Solutions

**\[Required, if not applicable write N/A\]**

List out solutions that were considered but ultimately rejected. Consider free form \- but a possible format shown below.

## Alt 1 Google Docs

**Pros:**

* Fits existing documents and templates used by many teams

**Cons:**

* Difficult to integrate with AI tools. 

* Difficult to search and index

**Reason Rejected:**

* Want to standardize around a simple text format and use AI tools also for diagramming, etc.


# Background

With the rise of Agile software development practices and large open source projects, software development teams needed to devise new and lightweight (w.r.t to previous software architecture documents) ways of recording architecture proposals and decisions. As Agile was born in part as a reaction to waterfall styles of planning and development and famously prioritized “Working softwareover comprehensive documentation” so too there was a need to replace monolithic large software design specifications with something lighter weight but that still encouraged good architecture. 

From this need for a new way of practicing software architecture a  body of work and theory has evolved around the concepts of “Architecture Decision Records” which in turn are also termed “Any Decision Record”, and RFCs or Enhancement proposals (PEP, KEP, REP). 

In each case the core requirements of the process are that the team document the problem, the proposal / design, the status of the proposal, implications / follow on work, and any alternatives that were considered using a standard template and review process.  

Just as in Agile planning, each team modifies the template and process to fit their needs.

## References

1. [Documenting Architecture Decisions (cognitect.com)](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)

2. [The most plagiarized Architecture Decision Record blog on the internet. | by Conall Daly | Medium](https://conalldalydev.medium.com/the-most-plagiarised-architecture-decision-record-blog-on-the-internet-c9dd2018c1d6)  
     
3. [adr.github.io](https://adr.github.io/)  
     
4. [When Should I Write an Architecture Decision Record \- Spotify Engineering : Spotify Engineering (atspotify.com)](https://engineering.atspotify.com/2020/04/when-should-i-write-an-architecture-decision-record/)  
     
5. [Scaling Engineering Teams via RFCs: Writing Things Down \- The Pragmatic Engineer](https://blog.pragmaticengineer.com/scaling-engineering-teams-via-writing-things-down-rfcs/)   
     
6. [Love Unrequited: The Story of Architecture, Agile, and How Architecture Decision Records Brought Them Together | IEEE Journals & Magazine | IEEE Xplore](https://ieeexplore.ieee.org/document/9801811)  
     
7. [ray-project/enhancements: Tracking Ray Enhancement Proposals (github.com)](https://github.com/ray-project/enhancements)

8. [Kubernetes Enhancement Proposals](https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/0000-kep-process/README.md)

9. [Rust RFC](https://github.com/rust-lang/rfcs/blob/master/text/0002-rfc-process.md)


[rust-rfc]: https://github.com/rust-lang/rfcs/blob/master/text/0002-rfc-process.md
[kep]: https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/0000-kep-process/README.md
[limited]: [../NNNN_limited_template.md]
[complete]: [../NNNN_complete_template.md]


