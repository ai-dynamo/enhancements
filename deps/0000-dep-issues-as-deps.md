# DEP-0000: GitHub Issues as DEP Artifacts

| Field | Value |
|-------|-------|
| Status | Draft |
| Author | nnshah1 |
| Category | Process |
| Created | 2024-12 |
| Updated | 2025-04 |

## Summary

Use GitHub Issues on ai-dynamo/dynamo as the primary artifact for Dynamo Enhancement Proposals (DEPs). Issues replace the separate enhancements repository for most proposals, reducing context-switching and enabling agent-native workflows.

## Motivation

The current DEP process has a 67% stall rate. Analysis reveals three root causes:

1. **Context switching kills adoption** — Contributors must leave the dynamo repo, navigate the enhancements repo, understand a different file structure, and submit a PR. Each hop loses participants.

2. **Specs, plans, and implementation are artificially separated** — A DEP in the enhancements repo has no direct link to the code it governs. Implementation PRs reference DEPs by URL, but the connection is one-way and easily broken.

3. **Agents work better with issues** — Modern AI coding agents (Claude, Cursor, Copilot) have native GitHub Issue support. They can read, create, comment, and close issues. Markdown files in a separate repo require custom tooling.

Moving DEPs to issues solves all three problems while preserving the governance properties (review, approval, traceability) that motivated the original process.

## Goals

1. **Single pane of glass** — All DEP activity happens in ai-dynamo/dynamo. No repo switching.
2. **Lower barrier to entry** — Creating a DEP is as simple as opening an issue with a template.
3. **Auditable decisions** — All comments, approvals, and revisions are preserved in issue history.
4. **Agent-native** — Standard `gh` CLI commands work out of the box.
5. **Preserve governance** — Area PICs still review and approve. Merge gating still enforced.

## TPM Role

The TPM function supports the DEP process in four ways:

- **Triage driver** — Release readiness and bug triage are already part of the TPM workflow. DEP triage is the same motion: aging reports, stall detection, PIC follow-up. Most of this can be agent-assisted.
- **Implementation gap report** — REQ 5 (retroactive DEPs) needs someone flagging large PRs that merged without design docs. Reviewing merged PRs for release notes is already happening — adding a "needs DEP?" check is straightforward.
- **Cross-cutting coordinator** — Multi-area DEPs need someone tracking cross-area review who isn't an area PIC. The TPM fills this role.
- **DEP metrics** — Status, age, stall rate by area. Feeds into the weekly execution meeting.

## When Is a DEP Required?

A DEP is required when a change:

- Affects multiple components
- Introduces or modifies a public API
- Alters communication plane architecture
- Affects backend integration contracts

This test is derived from the Dynamo Governance proposal (pending merge). PICs use it to determine whether incoming PRs need a DEP reference.

## Non Goals

1. **Not changing what requires a DEP** — The threshold for when a DEP is needed remains unchanged. This proposal changes where DEPs live, not when they're required.
2. **Not defining CODEOWNERS** — CODEOWNERS restructure is a companion effort. This DEP documents the target state but doesn't implement it.
3. **Not building complex automation** — Phase 0 uses manual label management. Automation comes later if needed.

## Requirements

### REQ 1: Issue as Spec

The DEP specification lives in the issue body. The issue body is the source of truth.

- Issue title follows the format: `[DEP] <short title>`
- Issue body uses the DEP template (lightweight or full)
- Edits to the issue body are visible in issue history

### REQ 2: Plan as Attachment

Implementation plans attach to the DEP issue, not separate documents.

- Plans can be in the issue body (for simple DEPs) or linked task lists
- Complex plans can use GitHub Projects for tracking
- All plan artifacts link back to the parent DEP issue

### REQ 3: Label Lifecycle

DEP state is tracked via labels:

| Label | Meaning |
|-------|---------|
| `dep:draft` | Work in progress, not ready for review |
| `dep:review` | Ready for PIC review |
| `dep:approved` | Approved by area PIC(s) |
| `dep:implementing` | Implementation in progress |
| `dep:done` | Implementation complete and merged |
| `dep:deferred` | Postponed to future release |
| `dep:rejected` | Not approved, with documented rationale |
| `dep:replaced` | Superseded by another DEP |

Only one `dep:*` label should be active at a time.

### REQ 4: PIC Assignment via Area Labels

Area labels determine which PIC(s) must approve:

- Author applies `area/<name>` label(s) to the DEP issue
- GitHub CODEOWNERS (or manual assignment) notifies the appropriate PIC
- Multi-area DEPs require approval from all affected area PICs

### REQ 5: Approval via /approve

PICs approve DEPs by commenting `/approve` on the issue.

- The `/approve` comment is timestamped and attributed
- For multi-area DEPs, each PIC comments `/approve` separately
- Approval without comment is invalid — PICs must use the command

### REQ 6: Merge Gating

PRs implementing a DEP must reference the DEP issue.

- PR description includes `DEP: #<issue-number>` or `DEP: N/A`
- GitHub Action validates the reference format
- PRs without valid DEP reference are flagged (soft gate initially)

### REQ 7: Revision Traceability

DEP revisions are tracked in issue history.

- Major revisions should be summarized in a comment
- The issue body reflects the current approved state
- Historical versions are recoverable from GitHub's edit history

## Area PICs and CODEOWNERS Mapping

### Individual PIC Areas

| Area | Label | PIC | CODEOWNERS Team | Key Paths |
|------|-------|-----|-----------------|-----------|
| External API | area/external-api | Graham King (@grahamking) | @ai-dynamo/dynamo-rust-codeowners | lib/llm/src/http/, lib/llm/src/grpc/, lib/llm/src/protocols/ |
| Frontend | area/frontend | Rudy Pei (@PeaBrane) | @ai-dynamo/dynamo-rust-codeowners | components/src/dynamo/frontend/, lib/llm/src/preprocessor/ |
| Router | area/router | Rudy Pei (@PeaBrane) | @ai-dynamo/dynamo-rust-codeowners | components/src/dynamo/router/, lib/llm/src/kv_router/ |
| Backend: vLLM | area/backend-vllm | Alec (@alec-flowers) | @ai-dynamo/python-codeowners | components/src/dynamo/vllm/, examples/backends/vllm/ |
| Backend: TRT-LLM | area/backend-trtllm | Yuewei Na (@nv-yna) | @ai-dynamo/python-codeowners | components/src/dynamo/trtllm/, examples/backends/trtllm/ |
| Backend: SGLang | area/backend-sglang | Ishan Dhanani (@ishandhanani) | @ai-dynamo/python-codeowners | components/src/dynamo/sglang/, examples/backends/sglang/ |
| KV/Memory | area/kv-memory | Rudy Pei (@PeaBrane) | @ai-dynamo/dynamo-rust-codeowners | lib/memory/, lib/runtime/src/transports/ |
| Multimodal | area/multimodal | Ryan McCormick (@rmccorm4) | @ai-dynamo/python-codeowners | examples/multimodal/, components/src/dynamo/*/multimodal* |
| Planner | area/planner | Hongkuan Zhou (@tedzhouhk) | @ai-dynamo/python-codeowners | components/src/dynamo/planner/, components/src/dynamo/global_router/, components/src/dynamo/global_planner/, components/src/dynamo/profiler/ |
| Core Platform | area/core-platform | Graham King (@grahamking) | @ai-dynamo/dynamo-rust-codeowners | lib/runtime/, lib/bindings/, components/src/dynamo/sdk/ |
| Observability | area/observability | Neelay Shah (@nnshah1) | @ai-dynamo/Devops | deploy/observability/, docs/observability/ |
| Fault Tolerance | area/fault-tolerance | Neelay Shah (@nnshah1) | @ai-dynamo/python-codeowners | docs/fault-tolerance/, tests/fault_tolerance/ |

### Team-Owned Areas

| Area | Label | PIC | CODEOWNERS Team | Key Paths |
|------|-------|-----|-----------------|-----------|
| Inference Gateway | area/gateway | Anna Tchernych (@atchernych) | @ai-dynamo/dynamo-deploy-codeowners | deploy/inference-gateway/ |
| DevOps | area/devops | Harrison Saturley-Hall (@saturley-hall) | @ai-dynamo/devops | .github/, container/ |
| Documentation | area/docs | (team) | @ai-dynamo/dynamo-docs-codeowners (new) | docs/, fern/, *.md |
| Process | area/process | Neelay Shah, Dan Gil, David Zier | @ai-dynamo/dynamo-process-codeowners (new) | CODEOWNERS, CONTRIBUTING.md, .github/ISSUE_TEMPLATE/ |

### Emerging Areas (Potentially Co-Maintained)

| Area | Label | Current Owner | CODEOWNERS Team | Key Paths |
|------|-------|---------------|-----------------|-----------|
| K8s / DGDR | area/k8s | Hannah Zhang (@hhzhang16) | @ai-dynamo/dynamo-deploy-codeowners | deploy/operator/, deploy/helm/, deploy/snapshot/ |
| XPU / Intel | area/xpu | (team) | @ai-dynamo/devops | TBD |

> **Note:** The three backend PICs (vLLM, TRT-LLM, SGLang) coordinate on cross-backend design parity — shared APIs, common patterns, feature matrix alignment.

> **Note:** CODEOWNERS restructure is a companion PR, separate from this DEP. The table above documents the target state.

> **Note:** Two new GitHub teams need to be created: `dynamo-docs-codeowners` and `dynamo-process-codeowners`.

## Cross-Cutting DEPs

Cross-cutting is not a standing area with a dedicated PIC. When a DEP spans multiple areas:

1. The author applies area labels for all affected areas
2. The TPM assigns a **lead PIC** (typically from the most-affected area)
3. The lead PIC coordinates review with **consulted PICs** from other affected areas
4. All consulted PICs must post `/approve` or `/defer` before the lead PIC can approve
5. Core Maintainers are available for escalation if PICs disagree

## Proposal

### Minimal Viable DEP Flow

1. Author opens issue with `[DEP]` title prefix
2. Author applies `dep:draft` and `area/<name>` labels
3. Author writes spec in issue body using template
4. When ready, author changes label to `dep:review`
5. PIC reviews and comments `/approve` or requests changes
6. On approval, label changes to `dep:approved`
7. Author implements, referencing `DEP: #N` in PRs
8. On merge of final PR, label changes to `dep:done`

### End-to-End Workflow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           DEP LIFECYCLE                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐ │
│  │  draft   │───▶│  review  │───▶│ approved │───▶│implement │───▶│  done  │ │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘    └────────┘ │
│       │               │               │                              ▲       │
│       │               │               │                              │       │
│       ▼               ▼               ▼                              │       │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                        │       │
│  │ rejected │    │ deferred │    │ replaced │────────────────────────┘       │
│  └──────────┘    └──────────┘    └──────────┘                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Issue Anatomy

**Lightweight Template** (for small enhancements):

```markdown
## Summary
One paragraph describing the enhancement.

## Motivation
Why is this needed? What problem does it solve?

## Proposal
How will it work? Include API changes, configuration, behavior.

## Alternatives Considered
What other approaches were evaluated?
```

**Full Template** (for architectural changes):

```markdown
## Summary
One paragraph describing the enhancement.

## Motivation
Why is this needed? What problem does it solve?

## Goals
- Goal 1
- Goal 2

## Non Goals
- Non-goal 1

## Requirements
- REQ 1: Description
- REQ 2: Description

## Proposal
Detailed design including:
- Architecture changes
- API changes
- Configuration changes
- Migration path

## Alternatives Considered
| Alternative | Pros | Cons |
|------------|------|------|
| Alt 1 | ... | ... |

## Implementation Plan
- [ ] Phase 1: ...
- [ ] Phase 2: ...

## Open Questions
- Question 1?
```

### Agent Workflow

| Task | Command |
|------|---------|
| Create DEP | `gh issue create --title "[DEP] Title" --body-file dep.md --label dep:draft,area/router` |
| List DEPs | `gh issue list --label dep:draft` |
| View DEP | `gh issue view 123` |
| Add comment | `gh issue comment 123 --body "Comment text"` |
| Approve DEP | `gh issue comment 123 --body "/approve"` |
| Change status | `gh issue edit 123 --remove-label dep:draft --add-label dep:review` |
| Close DEP | `gh issue close 123 --reason completed` |

### Making DEPs Work for Agents

1. **DEPs should generate rules files** — When a DEP is approved, distill requirements into a `.cursor/rules/` or `.claude/` rule scoped to the relevant files. An agent working on the router automatically picks up router DEP constraints. The DEP is the source of truth; the rule is the delivery mechanism.

2. **READMEs should index relevant DEPs** — Any README (component, module, top-level) should link to the DEPs that apply to that area. Agents read READMEs when exploring code, so it's zero-friction discovery.

3. **Code should reference DEPs** — Key modules that implement a DEP should have a module-level reference, e.g., `// Design: DEP-0014 (Error Standardization)`. Agents see it in-context and can pull the full DEP. Lightweight, greppable.

4. **Structured requirements block** — A machine-readable YAML block alongside prose requirements:

```yaml
requirements:
  - id: REQ-1
    summary: Area-based ownership
    level: MUST
    areas: [all]
    verifiable: true
  - id: REQ-2
    summary: PIC approval required
    level: MUST
    areas: [all]
    verifiable: true
```

### Collaboration Patterns

**Low Intensity** (most DEPs):
- Author writes spec
- PIC reviews async
- Approval via `/approve` comment

**Medium Intensity**:
- Author writes spec
- PIC requests changes via comments
- Author revises, PIC re-reviews
- Approval via `/approve` comment

**High Intensity** (cross-cutting, controversial):
- Author writes spec
- Synchronous meeting to discuss
- Meeting notes added as comment
- Multiple revision cycles
- All affected PICs must `/approve`

## Implementation Phases

### Phase 0: Bootstrap

- Create DEP issue template in `.github/ISSUE_TEMPLATE/`
- Create `dep:*` and `area/*` labels
- Document process in CONTRIBUTING.md
- This DEP (DEP-0000) is the first issue-based DEP

### Phase 1: Default to Issues

- New DEPs use issue template by default
- Add `DEP: #N` / `DEP: N/A` field to `feat:` PR template (soft gate, no merge block — missing references surface in weekly gap report)
- Add GitHub Action for DEP reference validation (soft gate initially)
- PICs trained on `/approve` workflow
- Weekly DEP status report in execution meeting

### Phase 2: Migration and Archive

- Existing in-flight DEPs complete in enhancements repo
- Archive enhancements repo as read-only
- Historical DEPs remain accessible via archive
- All new DEPs use dynamo repo issues

### Phase 3: Agent Integration

- Generate `.cursor/rules/` files from approved DEPs
- Add DEP index links to component READMEs
- Add `// Design: DEP-NNNN` references to key implementation files
- Add YAML requirements blocks to DEP templates
- Collapse skills to 3: `/dep-create`, `/dep-status`, `/dep-triage`

## Agent Skills

Three consolidated skills replace the previous ten:

### /dep-create

Covers: dep-create, dep-retroactive, dep-issue-create

Creates new DEPs or retroactive DEPs for existing implementations. Detects context (PR-based vs issue-based) and adapts.

Usage:
- `/dep-create` — Interactive DEP creation
- `/dep-create --retroactive PR#123` — Create DEP for merged PR
- `/dep-create --from-pr PR#456` — Extract DEP from in-flight PR

### /dep-status

Covers: dep-status, dep-related, dep-issue-status

Queries DEP status, finds related DEPs, and generates status reports. Detects context and adapts.

Usage:
- `/dep-status` — List all active DEPs
- `/dep-status #123` — Status of specific DEP
- `/dep-status --area router` — DEPs affecting router area
- `/dep-status --stalled` — DEPs with no activity >14 days

### /dep-triage

Covers: dep-triage, dep-review, dep-issue-approve, dep-issue-plan

Triage and review workflows for PICs and TPM. Detects context and adapts.

Usage:
- `/dep-triage` — Generate triage report for all areas
- `/dep-triage --area backend-vllm` — Triage for specific area
- `/dep-triage --approve #123` — PIC approval workflow
- `/dep-triage --plan #123` — Generate implementation plan

## Alternate Solutions

### Alternative 1: Keep Enhancements Repo

**Pros:**
- No migration needed
- Clear separation of design vs implementation

**Cons:**
- Context switching remains
- 67% stall rate continues
- Agent tooling requires custom integration

**Decision:** Rejected — does not address root causes.

### Alternative 2: Docs Folder in Dynamo Repo

**Pros:**
- Single repo
- Version controlled with code

**Cons:**
- PRs for spec changes conflated with code PRs
- Review process less visible than issues
- No native discussion threading

**Decision:** Rejected — issues provide better collaboration primitives.

### Alternative 3: External Tool (Notion, Confluence)

**Pros:**
- Rich editing experience
- Better for long-form documents

**Cons:**
- Another tool to manage
- No native GitHub integration
- Agent support varies

**Decision:** Rejected — adds complexity without solving core problems.

## Portability and Migration Risk

**Risk:** GitHub Issues lock-in.

**Mitigation:**
- Issue bodies are markdown — portable
- GitHub API allows bulk export
- Issue history is preserved in API responses
- If migration needed, issues export to any markdown-based system

**Migration effort estimate:** Low. Issues are simpler to migrate than repository-based DEPs.

## Terminology

| Term | Definition |
|------|------------|
| DEP | Dynamo Enhancement Proposal |
| PIC | Person In Charge — area owner responsible for review and approval |
| Area | Functional domain of the codebase (e.g., router, frontend, backend) |
| Cross-cutting | DEP affecting multiple areas |
| Soft gate | Validation that warns but does not block merge |
| Hard gate | Validation that blocks merge until resolved |
