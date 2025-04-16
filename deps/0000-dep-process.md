# Dynamo Enhancement Proposals

**Status**: Draft

**Authors**: [nnshah1](https://github.com/nnshah1), [ryanolson](https://github.com/ryanolson)

**Category**: Process

**Replaces**: N/A

**Replaced By**: N/A

**Implementation**: TBD

**Sponser / Steward**: [nnshah1](https://github.com/nnshah1)

**Required Reviewers**: dzier, suman, team

**Review Date**: TBD

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

* Lightweight and Scalable

  The format and process should be applicable both to small / medium
  sized changes as well as large ones. The process should not impede
  the rate of progress but serve to provide timely feedback,
  discussion and ratification on key proposals. The process should
  also support retroactive documents to capture and explain decisions
  already made.
	  
* Single Document for Requirements and Design

  Combine aspects of requirements documents, design documents and
  software architecture documents into a single document. Give one
  place to understand the motivation, requirements, and design of a
  feature or process.
	
* Support Process, Architecture and Guideline Decisions
	
  Have a single format to articulate decisions that effect process
  (such as github merge rules or templates) as well as code and
  design guidelines as well as features.

* Clear 

  Should be relatively clear when a document is required and when the
  review needs to be completed and by who and what the overall process
  is.
	
* Encourage Collaboration

  Should allow for easy collaboration and communication between
  authors and reviewers

* Flexible

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

Enhancement proposals will be stored in github in a seperate
repository. We provide two templates [limited][limited] and
[complete][complete] where `limited` is a strict subset of `complete`
and both indicate which sections are `required` and which are
`optional`.

# Implementation Details

* When is a proposal required?

It is difficult to enumerate all the circumstances where a proposal
would be required or not requiured. Generally we will follow this
process when making "substantial changes". What is "substantial" is
evolving and mainly determined by the core team and community. 

Generally speaking a proposal would not be required for:

** Bug fixes that don't change advertised behavior. 

** Documentation fixes / updates.

** Minor refactors within a single module. 

Generally speaking proposals would be required for:

** New features which add significant functionality.

** Changes to existing features or code which require discussion. 

** Responses to security related vulnerabilities found directly in the project code. 

** Changes to packaging and installation

** When a `maintainer` or `code owner` recommends that a change go
through the proposal process.

When in doubt reach out to a `maintainer` or `code owner`.

* Proposal Process

** Fork or create a branch in the `enhancements` repository

** Copy the [NNNN_limited_template.md][limited] or
[NNNN_complete_template.md][complete] to `deps/0000-my-feature.md`
(where `my-feature` is descriptive, don't assign an `DEP` number yet)

** Identify a `sponser / steward` from the list of `maintainers` or
`code owners` to help with the process.

** Fill in the proposal template. Be sure to include all `required`
sections. Keep sections in the order prescribed in the template.

** Work with the `steward` to identify the required reviewers and a
timeline for review.

** Submit a pull request to the `enhancements` repository

** Iterate and incorporate feedback via the pull request.

** When review is complete The `steward` will merge the request and update the status.

** `steward` should assign an id 

** author and `steward` should add issues and/or PRs as needed to track implementation

* Minor Changes After Review

For minor changes / changes that are in the spirit of the review -
updates can be made to the document without a new proposal.

Example: links to implementation

* Significant Changes After Review

For significant changes - a new proposal should be made and the
original marked as replaced.

* Maintainence 

** DEPs should be reviewed for updates / replace / archive on a
regular basis.

* Senstive Changes and Discussions

Certain types of changes need to be discussed and ratified before
being made public due to timing of non-disclosed information.

In such (rare) cases - drafts and reviews will be conducted offline by
`authors`, `code owners` and `maintainers` and the public proposals
updated when possible.

Example: when responding to undisclosed security vulnerabilities we
want to avoid inadvertantly encouraging zero day attacks for deployed
systems.

## Deferred to Implementation

* Definition of `code owners` and `maintainers`

* Whether or not to organize `deps` into sub directories

* Tooling around the creation / indexing of `deps`

# Alternatives

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


-----------------------------------------------------------------------------------------------

DRAFT Below Here


# Problem

As the Triton Inference Serving project continues to grow in scope we need a way to capture architecture, design and process decisions in a consistent, light weight, maintainable way. Currently while design and architecture decisions are captured using design docs, the format, naming, and review process for the design docs are variable and inconsistent. It’s largely unclear when a design doc is required and the timeline and process for review and approval are unknown. The goal of this document is to lay out the process and template that the team will use for documenting and recording architectural, design, and process decisions. 

# Requirements and Goals 

* Format and process should be lightweight enough to be adopted by all Triton engineering teams  
* Should be relatively clear when a document is required and when the review needs to be completed and by whom  
* Should allow for easy collaboration and communication between authors and reviewers including other teams at NVIDIA and external customers and partners (when relevant)  
* Format and process should be flexible enough to be used for different types of decisions requiring different levels of detail and formatting of sections.  
* Design records and the process of writing and approving them should first and foremost encourage the thoughtful evaluation of design, process, and architecture choices and lead to timely decisions with a clear record of what was decided, why, and what other options were considered.  

## Non Goals

* Decision records do not take the place of other forms of documentation such as user / developer facing documentation (including architecture documents, api documentation, [TAVA](https://confluence.nvidia.com/pages/viewpage.action?spaceKey=PRODSEC&title=TAVA+Guidance&sxr=1), etc.)  
* Prototyping and early development are not gated by design / architectural approval.  
* Decision records should not be a perfunctory process but lead to discussion and thought process around good designs. 

# Proposal

[Template (Complete)](https://docs.google.com/document/u/1/d/1CMH9g3lnIhrJ5445Ubm7OTjpfCDqBINvEzE1WeVn3Qg/edit),  [Template(Limited](https://docs.google.com/document/d/1ONZbl8HP-lU1DpB9j2GJzcuBpdzXLfgneUeTvxkQd7I/edit)),  [Folder](https://drive.google.com/drive/u/1/folders/1DoVUm4Mkz-d0qekGPMI6HwDvpDrNX3h9)

Triton server project engineering teams will adopt a lightweight decision record template similar to enhancement proposals, ADRs, and previous design doc templates. While some projects utilize both ADRs and RFCs \- as most of the sections are the same we will adopt a **single template and review process** for both.

Decision records will be kept in Google Docs in a folder labeled Decision Records with sub folders for different categories including “**Architecture and Design**”, “**Coding Guidelines**”, “**Process**” and a readme or index describing the docs included in each folder. Each decision record will be numbered within its folder as: **YY-MM-DD  \<title\>**.  

Because we are collapsing RFCs and ADRs into one template \-the records will generally propose and ratify a **collection of related decisions** (similar to a code PR is a related set of changes) as opposed to strictly being a **single decision**. We should strive to keep them as small as possible but also not separate them too finely. Generally this should come up and out of the draft process and should be clear by the time a doc is “under review”. 

Each **category** will have designated **architecture PICs** that will help organize and drive ratification. Each document will have a **primary author** and one or more secondary authors where the primary author is responsible for ensuring the decision process is moving forward.

**Note about template:** 

The sections of the template are listed as required / optional and should not be changed. The internal format of each section (sub sections, diagrams, tables) can be customized to the decision at hand. When there is a question \- the lead architect / category PIC will help clarify. 

# Implementation Details

* ### **When are decision records needed?** 

  This will largely be team driven and situation by situation but, Almost always (according to [spotify](https://engineering.atspotify.com/2020/04/when-should-i-write-an-architecture-decision-record/)). Some things to keep in mind:

1) Any time a public interface is changed / extended \- a design doc / decision record should be created.  
2) Any medium to large change proposal.  
3) When clarifications about scope / problem / options are needed.  
4) Large changes in code base \- even if they are refactors  
5) Changes to process / build / test / etc. \- any significant change to our software architecture, design, planning, etc.   
6) Developer decides when working on a problem that a design review would be useful  
7) If there is doubt \- consult with technical leads & architects.

   

* ### **When are decision records not needed?**

1. When a change is a fix to existing documented behavior. 

   
2. When it is a minor enhancement to an existing feature (example extending vLLM with echo capabilities)    
3. If there is doubt \- consult with technical leads & architects. 

* ### **When are decision records and reviewers assigned?**

  At sprint and program planning meetings.


  When assigned a target completion date and a target review date are added.


  Review date should be at most 2 weeks after the draft is complete (unless extenuating circumstances).


  Authors and reviewers should be assigned and JIRA created to track. JIRA should be a story in the larger feature is applicable. JIRA should have component field as “Enhancement Proposal”. 

* ### **What is the ratification process for decisions?**

	  
Documents start in an “**in progress”** state. Not all sections need to be filled out and authors should actively discuss the proposal.

Authors should meet and discuss and finalize the proposal and options. Authors can consult and have previous discussions with category PICs  to make sure the proposal is the correct size and background is sufficient.

	When a doc is ready for review the authors move the status to “**under review**”.   
	  
Reviewers are notified and notified of the target date for review. Typically 1 \- 2 weeks from draft completion. 

Authors can schedule a short call with reviewers ahead of the ratification date (one or two days) to address opens all at once.

By the specified review date, reviewers should indicate their approval, decision, and the status moves to **“approved”**, **“rejected”** or **“deferred”**. If the reviewers are not in agreement by the review date, the Category PIC and lead architect will have final approval / decision making authority.

If there are unresolved opens by the deadline that can not be deferred   
Authors / Architect / Category PIC can choose to extend the review deadline.

During implementation, if **major decisions** are revisited the doc should be updated and reviewed with a link to PR. If minor changes are done, the document should be updated with the PR for the implementation.

After implementation, if changes need to be made \- they should be addressed in a new document with the original marked as superseded and a link given to the new document. 

* ### **Where are decision records stored?**

  Google Docs within [Triton Inference Server \- DL System Software \- Google Drive](https://drive.google.com/drive/u/1/folders/15piQ1dgzqYaDS7T0RWw2EcVovNQTmpdL) / Triton Enhancement Proposals

  Each document can have an optional folder **\<TITLE\> \[Artifacts\]** to store diagrams, images, other artifacts that need to be stored with the document.

* ### **How are decision records named?**

	            2023\-01-19 Title	

* ### **How to plan small / medium / large proposals and implementations in a release cycle?**

  Triton inference server and related projects have a monthly planning and release cadence and a weekly program review. 


  During monthly planning, while planning small, medium and large features the team should identify proposal authors and reviewers along with a timeline for draft and review complete dates. 


  For small and medium features we can target  a 1  \- 2 week proposal and review followed by 1 \- 2 week implementation. Proposals should start at the beginning of a release cycle.


  For larger features with multiple implementation phases the proposals should be broken down into smaller incremental features. In such cases the proposals should start in the N-1 release cycle, giving more time for implementation in the N cycle (that is if a large feature is targeted for Release 24.05 the proposal should be drafted and reviewed within the Release 24.04 cycle).

  During program review \- if it’s clear that the proposal draft or review won’t be complete with at least 1 sprint remaining for implementation we should expect the feature to slip. 


  **Note:** 

  Prototyping and initial implementation in a branch ARE NOT gated by proposal approval and in many cases should be encouraged to help identify feasibility of solutions. 


  Final merge and publishing / marketing of the feature ARE gated by the proposal approval.

* ### **How are proposals updated / maintained during implementation?**

  During final  implementation it is expected that changes will be made from the solution described in the proposal. 


  If the changes are minor and in the spirit of the proposal then a link should be added to the user / developer facing document / code MR of the final implementation.


  If the changes are major / deviate from the spirit of the proposal then the document should be updated and re-approved or superseded with a new proposal that outlines the actual implementation with some rationale. Reviewers and authors should be kept in the loop. 


  On rare / exceptional cases due to business deadlines, if the implementation has to be released before the design is reviewed \- the implementation should be marked as provisional / BETA and the document must be updated / superseded.


  At least one of the authors / reviewers should be involved in the code review to give guidance on the scope of changes from the proposal and whether they constitute a minor or major change.


  For implementation phases / features that are not part of the final version \- they can be marked as deferred / moved to alternate proposals.

* ### **Categories**

  * Architecture / Design  
  * Process  
  * Build / Infrastructure  
  * Coding Guidelines / Design Principles

* ### **Proposal Status**


| Status | Description |
| :---- | :---- |
| Not Started | Assigned but draft not started |
| In Progress | Draft is being written by author(s). Not yet ready for review \- but early comments welcome. |
| Under Review | Draft is being reviewed by reviewers (date for ratification is set) |
| Approved | Proposal is approved and further discussions move to implementation (code review, etc.) |
| Implemented | Proposal has been implemented / released \- if appropriate \- link to implementation / architecture / api doc  is added |
| Rejected | Proposal will not be implemented |
| Deferred | Proposal is deferred to later time and will not be implemented. |
| Superseded | Proposal is superseded by a later proposal \- link to new proposal added. |


* ### **Reviewer Status**


| Status | Description |
| :---- | :---- |
| Not Started | Assigned but review not started |
| Reviewing | Reviewer is actively reviewing |
|  |  |
| Approved | Reviewer approves and further discussion would be moved to implementation. |
|  |  |
| Rejected | Proposal is rejected |
| Deferred | Reviewer has given  tacit consented, is unable to review due to time, defers to the guidance of others. |
| Changes Requested | Reviewer has requested changes that need to be addressed before approval / rejection can be given. |


# Deferred to Implementation

* Final category list and architecture / category PICs  
* Submitting template to google docs as template  
* Regular bi-weekly architecture forum to review / discuss / ratify. Meeting will be agenda driven and canceled if there are no topics.

# Implementation Phases

### **Phase 0**

**Release Target: Mar 26, 2024**  
**Effort Estimate: 2 days**  
**JIRA:** *TBD*

#### Supported API / Behavior

* Folder structure in google docs  
* Concise and Full Templates

#### Not Supported

* Automated indexes

# Related Decision Records

**N/A**

# Alternate Solutions

1. ## Use Markdown formatted ADRs / Decisions in a separate github repo

	**Pros:** Markdown is easy to format and don’t have to worry about editor weirdisms  
                      Support in community \- can script index creation, etc.   
                      Live near the code.	  
**Cons**: Not as easy to collaborate in real time.   
Not as easy to share selectively with other teams/ companies.(would need to control via github / gitlab)  
	**Reasons to reject**: Google docs easier to get started

2. ## Use Existing templates

   1. [Design Doc Template \- Google Docs](https://docs.google.com/document/d/1DFhT7Uz6qiTp2vAa5tsTucOAkAsCqeELctLxs1lUuo8/edit#heading=h.ykdv684u0rnx)  
   2. [Design Document Template \- Google Docs](https://docs.google.com/document/d/1MIkUlC5nJ4XzCU1fCZ3mwcnrereiIx718zIcdLUr_PI/edit#heading=h.dh5skq4lgud6)  
   3. [cuDNN Design DocuDNN Design Doc Templatec Template \- Google Docs](https://docs.google.com/document/d/1ZNdRvM3z9IqTCZlXddgiaA6O_qJ5tOhSL9dQ_BVWE2M/edit#heading=h.sk2q0bv4mmjl)

	**Pros:** Use existing template  
	**Cons**: Doesn’t allow team to own and change template as needed  
	            	  
	**Reasons to reject**: Team should own the template and process for Triton.   
       Formatting of [Design Document Template \- Google Docs](https://docs.google.com/document/d/1MIkUlC5nJ4XzCU1fCZ3mwcnrereiIx718zIcdLUr_PI/edit#heading=h.dh5skq4lgud6) is hard to read.

**Notes:**    
Each template has slightly different terms and emphasis.   
Where are some use the term ‘objective’ others use ‘problem’ or ‘issue’. Each also seems to have been created by a  specific team for their own purposes (such as cuDNN).  
[Design Doc Template \- Google Docs](https://docs.google.com/document/d/1DFhT7Uz6qiTp2vAa5tsTucOAkAsCqeELctLxs1lUuo8/edit#heading=h.ykdv684u0rnx) is the closest \- and the proposed [template](https://docs.google.com/document/d/1INMNSkKxN5umnBnc984zhmxU5gcm1vkCzvNXOK_G8WE/edit#heading=h.ykdv684u0rnx) can be viewed as a customization.

Finally just as an Agile team customizes sprint planning and JIRA templates \- the engineering teams should customize the architecture review process to best fit their project.

3. ## Use SW PLC templates 

[**\[PLC\] PLC-L1 template \- Google Docs**](https://docs.google.com/document/d/1W4Ucfmngu43u3_kF-EGwEDpNi1DwD1n8GFuJRLOQvnE/edit)

**Pros:** Use existing template  
	**Cons**: Doesn’t allow team to own and change template as needed  
	**Reasons to reject**: Team should own the template and process for Triton.   
        PLC templates seem verbose and a little intimidating.   
             **Notes:**    
As Triton projects are planned and executed much like other Agile, open source projects, Triton should adopt best industry practices with regards to Agile and open source projects. As ADRs and RFCs have been adopted by a larger number of Agile teams and open source projects \- they are a better fit for Triton than product life cycle oriented documents.  
Many docs are still formatted for print and archive \- and add unneeded clutter when pageless schemes are more common in OSS proposals.

4. ## Confluence

   Several teams at NVIDIA use confluence to store indexes / pointers to design proposals, but general do not use Confluence to author design or store actual proposals. 

   

   Examples:

   [**Design Docs and Proposals \- Computelab \- Confluence (nvidia.com)**](https://confluence.nvidia.com/display/COMPLAB/Design+Docs+and+Proposals)

   [**https://confluence.nvidia.com/display/TGM/EUD+Design+Doc?src=contextnavpagetreemode**](https://confluence.nvidia.com/display/TGM/EUD+Design+Doc?src=contextnavpagetreemode) 

   [**XLA Design Docs and Meetings \- Deep Learning \- Confluence (nvidia.com)**](https://confluence.nvidia.com/display/DL/XLA+Design+Docs+and+Meetings)

   [**Design Doc and Presentation Collection \- GPU Compute Arch \- Confluence (nvidia.com)**](https://confluence.nvidia.com/display/GCA/Design+Doc+and+Presentation+Collection)

   **Pros:** 

   Using Confluence to store indexes and or proposals can lead to use of more advanced features such as labeling, content reuse, approval flows.

   
	**Cons**:   
		Confluence editing features do not allow for as easy real time coordination between authors.   
		Confluence is more difficult to share with external partners.  
		Confluence is not used widely for design proposals within NVIDIA.

	**Reasons to reject**:   
		  
Familiarity and use of Google Docs to align with other best practices at NVIDIA and to align with existing proposal work flow.

             **Notes:**  

		Can be revisited if practices change / more information becomes available on better workflows.  


