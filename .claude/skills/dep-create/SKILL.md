---
name: dep-create
description: Create a new Dynamo Enhancement Proposal (DEP). Use when starting a new architectural proposal, design document, or process change.
argument-hint: "[feature-name or description]"
---

# Create a new DEP

Create a new Dynamo Enhancement Proposal document. Be terse — include only the sections needed for this specific proposal.

## Steps

1. If no argument provided, ask the user for a brief description of the proposal
2. Determine the appropriate area from: frontend, router, backends, kvbm, bindings, deployment, observability, cicd, process, cross-cutting
3. Discuss the problem statement with the user before drafting — understand motivation, goals, and constraints
4. Research the codebase and existing DEPs for context:
   - Read relevant existing DEPs in `deps/`
   - Check GitHub PRs and Linear issues for related work
   - Understand the current state before proposing changes
5. Create `deps/0000-$ARGUMENTS.md` using YAML frontmatter:

```yaml
---
dep: 0000
title: <title>
status: draft
area: <area>
authors: []
pic: null
linear: []
implementation-prs: []
---
```

6. Follow the template structure from `NNNN-complete-template.md` but be terse:
   - **Required**: Summary, Motivation, Proposal, Alternate Solutions
   - **Include only if needed**: Goals/Non-Goals, Requirements, Implementation Details, Implementation Phases, Related Proposals, Background
   - Omit sections that don't apply — don't include empty optional sections
7. After drafting, suggest next steps: identify a sponsor, create a PR, link to Linear issues

## Guidelines

- Be as terse as possible while remaining clear
- Lead with the problem, not the solution
- Ground proposals in evidence — reference actual PRs, issues, code patterns
- Include concrete examples over abstract descriptions
- Requirements should be measurable (use RFC-2119 language: MUST, SHOULD, MAY)
- Alternate solutions section is required even if brief — shows you considered options
