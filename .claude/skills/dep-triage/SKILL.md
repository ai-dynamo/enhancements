---
name: dep-triage
description: Run weekly DEP triage — surface stalled proposals, large PRs without DEPs, and backlog priorities. Use during Friday PM/EM review.
argument-hint: "[weeks-back to scan, default 1]"
---

# DEP Triage

Support the weekly Friday triage by surfacing items that need attention.

## Steps

### 1. Stalled DEPs
- List all open enhancement PRs: `gh pr list --repo ai-dynamo/enhancements --state open --json number,title,author,createdAt,updatedAt`
- Flag PRs with no activity in 14+ days
- Flag PRs with no reviewers assigned

### 2. Large Feature PRs Without DEPs
- List recently merged feature PRs: `gh pr list --repo ai-dynamo/dynamo --state merged --limit 30 --json number,title,author,mergedAt,additions,labels`
- Filter to PRs with 200+ additions
- Check if PR body or title references a DEP (look for "DEP", "enhancement", or links to ai-dynamo/enhancements)
- Flag large PRs without DEP references

### 3. DEP Status Summary
- Run the dep-status check (read all DEPs, group by status)
- Highlight DEPs past their review date

### 4. Backlog Check
- Check GitHub Issues on enhancements repo if they exist: `gh issue list --repo ai-dynamo/enhancements`
- Cross-reference with Linear non-feature requirements project

## Output Format

Produce a triage summary with four sections:

```
## Triage Summary — <date>

### Needs Attention (stalled DEPs)
- ...

### Implementation Without Design Doc
- ...

### Ready for Review
- ...

### Recently Completed
- ...
```

Include suggested actions for each item (assign PIC, schedule review, draft retroactive DEP, close as stale).
