# DEP Process Improvements

**Status**: Draft

**Authors**: [nnshah1](https://github.com/nnshah1)

**Category**: Process

**Replaces**: N/A (extends [DEP-0000: Dynamo Enhancement Proposals](./0000-dep-process.md))

**Replaced By**: N/A

**Sponsor**: [nnshah1](https://github.com/nnshah1)

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Improvements to the Dynamo Enhancement Proposal (DEP) process to address
inconsistent adoption, abandoned proposals, and the growing gap between
architectural decisions made informally in PRs and the design record in
the enhancements repository. Introduces area-based ownership with
designated PICs (Persons In Charge), a weekly triage cadence, GitHub
Issues for backlog tracking, and a lightweight path from implementation
PRs to retroactive design documentation.

# Motivation

The DEP process as defined in [DEP-0000](./0000-dep-process.md) has been
successful in fostering thoughtful design when used. However, as Dynamo
reaches 1.0 and opens to external governance, the informal approach that
got us here will not scale to what comes next. With 1.0, Dynamo begins
to prioritize stability as much as innovation — API compatibility,
deprecation policies, and backward-compatible evolution all require that
design decisions are recorded, reviewable, and traceable. At the same
time, a larger team, external contributors who lack tribal knowledge,
and a longer maintenance horizon mean today's decisions must be
understood and respected years from now.

Three systemic problems have emerged:

**1. Proposals filed but never driven to completion.** Of 72 enhancement
PRs to date, 48 (67%) remain open. Twenty of these have been open for
six or more months with near-zero engagement. No labels, no assigned
reviewers, no review dates — the process steps prescribed in DEP-0000
are not being followed because there is no one accountable for driving a
given proposal through to a terminal state.

**2. Substantial features landing without corresponding design review.**
Recent examples include the KV router scheduling overhaul (PRs #7260,
#7339, #7395, #7462), unix domain socket transport (#7197, +1,934
lines), concurrent KV event consumer (#7293, +1,276 lines), and
standalone KV indexer runtime (#7295, +1,139 lines). In each case
design discussion happened inline in the PR body with Linear ticket
references. The design work was done — but it is not captured in the
enhancements repository where it can be discovered, reviewed broadly,
and maintained.

**3. Architectural decisions captured outside the project record.** The
decision to consolidate configuration parameters across Dynamo
components — a cross-cutting change affecting every service — was
discussed and captured in a Google Doc rather than a DEP. The rationale,
alternatives considered, and migration plan are not discoverable in the
enhancements repository or linked from the implementing PRs. When a new
engineer or external contributor encounters the configuration system and
asks "why was it done this way?", the answer lives in a document they
don't know exists and may not have access to. This is exactly the kind
of tribal knowledge that DEPs are meant to eliminate, and it illustrates
what happens when the process is too heavy for the pace of work — teams
route around it.

These are not failures of individual engineers. The current process lacks
the ownership model, triage cadence, and lightweight on-ramps needed to
scale with the team. Engineers move fast and make good decisions — we
need the process to keep up by making the right thing the easy thing.

As Dynamo opens to external contributors and community governance, the
stakes increase. External contributors need a discoverable record of
architectural decisions to make meaningful contributions. Maintainers
need a way to evaluate proposed changes against established design
direction. Without this, external PRs will either be rejected for
violating undocumented conventions or accepted without adequate design
review — both outcomes erode trust and slow the project.

DEP-0000 itself deferred several decisions that are now load-bearing:

> * Definition of `code owners` and `maintainers`
> * Whether or not to organize `deps` into sub directories for projects / areas
> * Tooling around the creation / indexing of `deps`
> * Decisions / guidelines on when a DEP is needed.

This proposal resolves those deferred items.

## Goals

* **Consistency without drag** — Every substantial architectural
  decision is captured in the enhancements repository without slowing
  down high-priority implementation work.

* **Transparency** — PM, engineering managers, and the broader team can
  see at a glance what design decisions are pending, in progress, or
  completed, and who owns them.

* **Accountability** — Every DEP area has designated PICs who are
  responsible for shepherding proposals, reviewing implementations, and
  maintaining design quality in their area.

* **Velocity-friendly** — Prototyping and implementation are not gated
  by design approval. Engineers are encouraged to build. The process
  ensures design review happens before it is too late to change
  direction, not before work can begin.

* **External governance readiness** — As Dynamo opens to external
  contributors, the process provides a clear, documented path for
  proposing and reviewing changes.

## Non Goals

* Replacing the existing DEP format or templates — the document
  structure from DEP-0000 remains unchanged.

* Defining the specific CODEOWNERS file structure for the `ai-dynamo/dynamo`
  repository — that will be a separate DEP, though this proposal
  establishes the principle that area PICs should also be code owners.

* Creating a heavyweight approval gate that blocks all PRs — the goal is
  transparency and traceability, not bureaucracy.

## Requirements

### REQ 1 Area-Based Ownership

Every DEP area **MUST** have at least one designated PIC. The PIC
**MUST** be responsible for:

- Shepherding DEPs in their area to a terminal state (approved,
  deferred, or rejected)
- Reviewing implementation PRs that touch their area for design
  consistency
- Ensuring test coverage, API stability, and documentation standards
  are met for their area
- Triaging new proposals and setting review timelines

### REQ 2 Weekly Triage Cadence

PM and engineering managers **MUST** review the DEP backlog weekly
(Fridays). The review **MUST** identify:

- New proposals requiring triage
- Stalled proposals requiring escalation or closure
- Implementation PRs merged without corresponding design documentation
- Timeline assignments for upcoming reviews

### REQ 3 Backlog Visibility

The DEP backlog **MUST** be tracked via GitHub Issues on the
enhancements repository. Each open DEP **MUST** have a corresponding
GitHub Issue with:

- Area label
- Assigned PIC
- Priority (set during weekly triage)
- Target review date

### REQ 4 Design Review Before Merge

Area PICs **SHOULD** review the design basics of implementation PRs in
their area before merge. This review **MUST NOT** gate prototype or
exploratory work. For high-priority items, design review **MAY** happen
post-merge, but the design **MUST** be documented within a reasonable
timeframe (see REQ 5).

### REQ 5 Retroactive Design Documentation

When an implementation PR is merged without a corresponding DEP, the
area PIC **MUST** ensure design documentation is created. If the PR
body contains sufficient design detail, a DEP **MAY** be drafted
retroactively from it. PM/EM **SHOULD** flag merged PRs lacking design
docs during weekly triage.

### REQ 6 Review Period

All DEPs **MUST** have a review period before approval, even if short.
The minimum review period is one business afternoon to allow others to
provide input. The area PIC **MAY** set longer review periods for
proposals with broad impact. High-priority items **MUST NOT** be
blocked by the review period — they proceed and are documented per REQ 5.

# Proposal

## Area-Based Organization

The enhancements repository will organize DEPs into area-based
subdirectories under `deps/`. The areas align with the component
structure established in the Dynamo 1.0 Non-Feature Requirements
project:

| Area | Subdirectory | Scope |
|------|-------------|-------|
| Frontend | `deps/frontend/` | HTTP/gRPC ingress, request handling, protocol support |
| Router | `deps/router/` | KV-aware routing, scheduling policies, load balancing |
| Backends | `deps/backends/` | vLLM, TRT-LLM, SGLang engine integrations |
| KV Block Manager | `deps/kvbm/` | KV cache management, transfer, indexing |
| Python Bindings | `deps/bindings/` | Python SDK, public API surface |
| Deployment | `deps/deployment/` | Operator, Helm charts, CRDs, container strategy |
| Observability | `deps/observability/` | Metrics, tracing, logging |
| CI/CD | `deps/cicd/` | Build, test pipelines, release automation |
| Process | `deps/process/` | DEP process, guidelines, governance |
| Cross-Cutting | `deps/cross-cutting/` | Multi-area proposals, architecture-wide decisions |

Existing DEPs will be migrated to the appropriate subdirectory. DEP
numbering remains global and monotonically increasing.

## Area PICs

Each area will have one or two designated PICs drawn from the existing
engineering team. PICs are responsible for both the design quality
(enhancement proposals) and implementation quality (code review) in
their area.

The initial PIC assignments will be established based on the component
ownership already defined in the 1.0 Non-Feature Requirements Linear
project (DYN-2060 through DYN-2069). The PIC table will be maintained
in the enhancements repository README and kept in sync with the
CODEOWNERS file in the main dynamo repository (see Related Proposals).

**PIC responsibilities:**

1. **Triage** — When a new DEP or implementation PR touches their area,
   the PIC assigns priority and review timeline
2. **Shepherd** — Drive DEPs to a terminal state; do not let proposals
   sit idle
3. **Review** — Ensure implementation PRs are consistent with approved
   designs; review design basics before merge when practical
4. **Maintain** — Keep area DEPs current; mark replaced or deferred as
   the codebase evolves
5. **Delegate** — PICs may delegate review to domain experts but remain
   accountable

## Backlog Tracking via GitHub Issues

Each DEP will have a corresponding GitHub Issue in the enhancements
repository. Issues provide:

- **Labels** for area (`area/frontend`, `area/router`, etc.) and status
  (`status/draft`, `status/under-review`, `status/approved`, etc.)
- **Assignees** for the area PIC and author
- **Milestones** for release alignment (e.g., "Dynamo 1.1")
- **Cross-references** to Linear issues and implementation PRs

GitHub Issues are preferred over a static index file because:
- They integrate naturally with the PR workflow
- They can sync with Linear for PM visibility
- They support filtering, sorting, and search
- They provide notification and subscription mechanisms

A maintained index of active DEPs **SHOULD** also be kept in
`README.md` for at-a-glance visibility.

## Cadence and Communication

Design review happens at multiple levels, each serving a different
purpose. The goal is to make it easy for anyone to find where
discussions are happening and contribute — whether that's in a DEP PR,
a Slack thread, an area team meeting, or the engineering sync.

### Friday PM/EM Triage (15-30 min)

PM and engineering managers review the DEP backlog weekly on Fridays:

1. **New items** — Assign area, PIC, priority, and target review date
2. **In progress** — Check for stalls; escalate or reassign as needed
3. **Implementation gaps** — Flag merged PRs (large feature PRs) that
   lack corresponding design documentation; assign retroactive DEP
   authoring to the area PIC
4. **Completed** — Verify DEPs are merged with correct status and
   numbering

The triage output is a prioritized list visible in GitHub Issues. Area
PICs use this to plan their review work for the following week.

### Engineering Sync DEP Segment (5-10 min)

The weekly engineering sync includes a standing DEP segment where area
PICs briefly surface:

- What's newly proposed or under review
- What's been approved or decided
- What needs broader input or has cross-cutting implications

This is an **awareness and seeding** function, not a review forum. The
purpose is to keep the full engineering team informed of architectural
direction, spark cross-pollination of ideas, and give anyone a
low-friction way to say "I have context on that" or "have you
considered X?" before a decision is finalized. DEPs serve not only to
propose changes but to capture decisions and the rationale behind them
— the sync helps ensure the right people know where to find and
contribute to those discussions.

### Area Team Discussions

Most areas already have their own weekly meetings or working sessions.
These are the natural venue for detailed design discussions on
area-specific DEPs. Area PICs drive these conversations and bring in
relevant stakeholders.

Design discussions that happen in area meetings **SHOULD** be captured
in the DEP PR comments or the DEP document itself so the rationale is
preserved for the broader team and future contributors.

### Cross-Area Design Discussions

When a DEP has cross-cutting implications or needs input from multiple
areas, the PIC schedules a focused design discussion with relevant
stakeholders. These are a normal part of the process — not exceptional
— and should be scheduled proactively when the PIC identifies
cross-area impact during triage or area review.

### Async Review (Default)

The expectation is that most detailed design review happens
asynchronously in DEP PR comments, Slack threads, and offline
conversations. The synchronous touchpoints (triage, sync, area
meetings, design discussions) serve to keep things moving and ensure
nothing falls through the cracks — not to replace written review.

## Prototype-Friendly Design Review

The process explicitly supports the following workflow:

```
Engineer prototypes → PIC reviews design basics → PR merges → DEP finalized
```

Key principles:

- **Implementation is not gated by DEP approval.** Engineers should
  prototype and iterate. Area PICs encourage this.
- **Design review should happen before merge when practical.** The PIC
  reviews the design approach (not just code correctness) as part of
  the normal PR review process. Even a brief review period (as short as
  an afternoon) is valuable.
- **Post-merge documentation is acceptable for high-priority items.**
  When velocity demands it, the implementation merges first and the DEP
  follows. PM/EM triage catches any gaps.
- **Rich PR descriptions can seed retroactive DEPs.** When a PR body
  contains a design section, architecture rationale, or detailed
  proposal, the area PIC can draft a DEP from it. This should be rare
  but is preferable to having no design record at all.

## Agent-Friendly Format and Tooling

As AI coding agents become standard development tools, the DEP format
and workflow should be optimized for both human and agent use.

### Structured Metadata

DEPs **SHOULD** adopt YAML frontmatter for machine-readable metadata
in addition to the human-readable header block. This allows agents to
reliably extract status, area, PIC, and cross-references without
fragile parsing:

```yaml
---
dep: 0000
title: DEP Process Improvements
status: draft
area: process
authors: [nnshah1]
pic: nnshah1
extends: 0000-dep-process        # builds on an existing DEP
replaces: null                    # fully supersedes another DEP
replaced-by: null                 # pointer when this DEP is superseded
linear: []
implementation-prs: []
---
```

The `extends` field captures when a DEP builds directly on a prior one
— adding to or refining its scope without replacing it. This is
distinct from `replaces` (full supersession) and complements the
Related Proposals section by making the relationship machine-readable.
Agents and tooling use `extends` and `replaces`/`replaced-by` to trace
the lineage of decisions and detect when a DEP chain may need review.

The human-readable header block remains for readability. Frontmatter
is the source of truth for tooling and agent queries.

### Standardized Cross-References

DEPs, implementation PRs, and Linear issues **SHOULD** use a
consistent linking convention so agents can trace design-to-code:

- DEP to implementation: `implementation-prs` field in frontmatter
- PR to DEP: checkbox in PR template ("Does this require a DEP?")
- Linear to DEP: link in Linear issue description

### Terse Templates

The existing templates include inline editorial guidance (`[Required]`,
`[Optional]`, placeholder descriptions). A clean template variant
**SHOULD** be provided for agent use — section headers only, no inline
instructions. The guidance for agents is: **be as terse as possible,
include only the sections needed for the proposal at hand.** This
guidance will be maintained in a Claude Code skills file alongside the
enhancements repository so agents can create, review, and triage DEPs
using purpose-built commands.

### Agent Workflow Skills

Claude Code skills (`/dep-create`, `/dep-status`, `/dep-retroactive`,
etc.) **SHOULD** be provided to automate common DEP workflow steps.
These skills encode the process defined in this DEP so that agents
follow it consistently. See `.claude/skills/` in the enhancements
repository for available commands.

## Relationship to Code Owners

Area PICs should also serve as code owners for their area in the main
`ai-dynamo/dynamo` repository. This ensures the same person who
understands the architectural direction also reviews the implementation.

The current CODEOWNERS structure is organized primarily by language
(`dynamo-rust-codeowners`, `python-codeowners`) rather than by
architectural area. This means a reviewer may approve code correctness
without evaluating design consistency.

A separate DEP will propose restructuring CODEOWNERS to be area-based,
with area PICs as the required reviewers. The principle established
here is: **the person who reviews the design should also review the
code.**

# Implementation Phases

## Phase 0: Process Bootstrap

**Release Target**: Immediate (does not require code changes)

**Work Item(s)**: TBD

**Actions:**

- Create GitHub Issue labels for areas and statuses in the enhancements
  repo
- Create GitHub Issues for all existing open DEP PRs
- Assign initial area PICs based on 1.0 Non-Feature Requirements
  ownership
- Update `README.md` with area PIC table and backlog link
- Begin weekly Friday triage

**Not Supported:**

- Repository restructuring into subdirectories (Phase 1)
- CODEOWNERS changes (separate DEP)

## Phase 1: Repository Restructuring

**Release Target**: Within 2 weeks of Phase 0 approval

**Work Item(s)**: TBD

**Actions:**

- Create area subdirectories under `deps/`
- Migrate existing DEPs to appropriate subdirectories (update all
  cross-references)
- Add CODEOWNERS to the enhancements repository mapping area
  subdirectories to area PICs
- Triage and close stale enhancement PRs (the 20+ PRs open for 6+
  months with no activity)

## Phase 2: Integration and Tooling

**Release Target**: Within 4 weeks of Phase 0 approval

**Work Item(s)**: TBD

**Actions:**

- Set up GitHub-Linear sync for DEP issues
- Add PR template to `ai-dynamo/dynamo` with a checkbox: "Does this PR
  require a DEP? If yes, link it. If unsure, tag your area PIC."
- Consider lightweight automation (GitHub Actions) to flag large
  feature PRs without DEP references
- Add YAML frontmatter to all existing DEPs for machine-readable
  metadata
- Create clean (agent-ready) template variant — section headers only,
  no inline instructions
- Publish Claude Code skills for DEP workflow automation
  (`/dep-create`, `/dep-status`, `/dep-review`, `/dep-related`,
  `/dep-retroactive`, `/dep-triage`)

# Related Proposals

* [DEP-0000: Dynamo Enhancement Proposals](./0000-dep-process.md) — The
  original process definition; this proposal extends it
* [0000-dynamo-api-and-non-feature-requirements](./0000-dynamo-api-and-non-feature-requirements.md) — Defines
  component areas and PIC assignments for 1.0
* CODEOWNERS Restructuring (future DEP) — Will restructure code
  ownership by area rather than language

# Alternate Solutions

## Alt 1: Static Index File Instead of GitHub Issues

**Pros:**

- Simple, no tooling dependency
- Always visible in the repo root

**Cons:**

- Requires manual upkeep; will fall out of date
- No filtering, sorting, or notification support
- Cannot sync with Linear
- Merge conflicts when multiple people update it

**Reason Rejected:**

- GitHub Issues provide the same visibility with better tooling and
  less maintenance burden. A lightweight index in README can complement
  Issues without replacing them.

## Alt 2: Strict Gate — No Merge Without Approved DEP

**Pros:**

- Guarantees every feature has a design document before landing
- Maximum consistency

**Cons:**

- Slows velocity significantly for high-priority work
- Encourages gaming (rubber-stamp DEPs to unblock merges)
- Discourages prototyping and experimentation
- Does not match the team's working style

**Reason Rejected:**

- The goal is transparency, not gatekeeping. The team's informal
  process produces good designs — the problem is capturing them, not
  creating them. A strict gate would create friction without
  proportional benefit.

## Alt 3: Keep Current Process, Just Enforce It

**Pros:**

- No process changes needed
- Familiar to existing team

**Cons:**

- The current process has been in place for nearly a year and adoption
  has declined over time
- Lacks the ownership and triage mechanisms needed to prevent
  abandonment
- "Try harder" is not a sustainable strategy

**Reason Rejected:**

- The process itself is sound but missing structural support. Adding
  area ownership, triage cadence, and backlog tracking addresses the
  root causes rather than the symptoms.

# Background

## Current State Evidence

The following data informed this proposal (collected March 2026):

**Enhancement PRs:** 72 total — 48 open (67%), 18 merged (25%), 6
closed (8%). Twenty PRs have been open for 6+ months with minimal
engagement.

**Recent large feature PRs without DEPs:** KV router scheduling
overhaul, unix domain socket transport, concurrent KV event consumer,
standalone KV indexer runtime, FastTokens BPE integration, pluggable
scheduling policy, multimodal Anthropic Messages API support.

**CODEOWNERS:** Organized by language (`dynamo-rust-codeowners`,
`python-codeowners`) with few area-specific exceptions (`kvbm-v2`,
`/lib/memory/`).

**Linear:** Non-Feature Requirements project has 84 issues (36 in
Backlog, 29 Done). Component areas well-defined but linkage to
enhancements repo is ad-hoc.

## References

* [DEP-0000: Dynamo Enhancement Proposals](./0000-dep-process.md)
* [Kubernetes KEP Process](https://github.com/kubernetes/enhancements/blob/master/keps/sig-architecture/0000-kep-process/README.md)
* [Rust RFC Process](https://github.com/rust-lang/rfcs/blob/master/text/0002-rfc-process.md)
* [Scaling Engineering Teams via RFCs](https://blog.pragmaticengineer.com/scaling-engineering-teams-via-writing-things-down-rfcs/)

## Terminology & Definitions

| Term | Definition |
| :--- | :--- |
| **Area** | A functional subdivision of the Dynamo project (e.g., frontend, router, backends) used to organize DEPs and assign ownership |
| **PIC** | Person In Charge — the designated owner of a DEP area, responsible for shepherding proposals and reviewing implementations |
| **Terminal State** | A DEP status that indicates the proposal is no longer active: approved, deferred, or rejected |
| **Retroactive DEP** | A design document created after implementation has already merged, capturing the design rationale from PR descriptions and offline discussions |

## Acronyms & Abbreviations

**DEP:** Dynamo Enhancement Proposal

**PIC:** Person In Charge

**PM:** Product Manager

**EM:** Engineering Manager
