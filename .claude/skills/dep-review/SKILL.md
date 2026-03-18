---
name: dep-review
description: Review a DEP for staleness, supersession, or divergence from current implementation. Use to check if a DEP is still accurate or needs updating.
argument-hint: "[dep-number, area, or 'all']"
---

# Review DEP Currency

Check whether a DEP is still current, has been superseded by implementation changes, or needs a new DEP to cover evolved requirements.

## Steps

### 1. Identify Target DEPs
- If argument is a number, review that specific DEP
- If argument is an area, review all DEPs in that area
- If 'all' or no argument, review all approved/draft DEPs

### 2. For Each DEP, Check Currency

**Implementation divergence:**
- Read the DEP's proposal and requirements sections
- Find linked implementation PRs (from frontmatter or "Implementation PR" header field)
- Search for recent PRs in `ai-dynamo/dynamo` that touch the same area: `gh pr list --repo ai-dynamo/dynamo --state merged --search "<area keywords>" --limit 20 --json number,title,mergedAt,additions`
- Compare: does the current implementation match what the DEP proposed? Flag significant divergences.

**Supersession:**
- Check if newer DEPs in the same area cover overlapping scope
- Check if the DEP's `Replaced By` or `extends` fields reference newer work
- Look for PRs or Linear issues that explicitly supersede the DEP's approach

**Staleness indicators:**
- DEP is in Draft status for 60+ days with no PR activity
- DEP references components, APIs, or patterns that no longer exist in the codebase
- DEP's requirements have been partially implemented but the DEP was never updated to reflect final state
- Implementation PRs merged after the DEP that change the design without updating the doc

### 3. Recommend Actions

For each DEP reviewed, recommend one of:
- **Current** — DEP accurately reflects implementation, no action needed
- **Update needed** — Minor divergences; DEP should be updated to match current state
- **New DEP needed** — Significant evolution beyond original scope; a new DEP should extend or replace this one
- **Superseded** — Another DEP or implementation has replaced this; mark as `replaced` with pointer
- **Stale/Abandon** — Draft that is no longer relevant; recommend closing the PR and marking as `deferred` or `rejected`

## Output Format

```
## DEP Review — <date>

### <DEP-NNNN>: <title>
**Status**: <current status>
**Verdict**: <Current | Update needed | New DEP needed | Superseded | Stale>
**Evidence**: <brief explanation with PR/code references>
**Action**: <specific recommendation>
```
