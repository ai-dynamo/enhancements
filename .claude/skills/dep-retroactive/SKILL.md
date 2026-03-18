---
name: dep-retroactive
description: Draft a DEP retroactively from an existing merged PR or set of PRs. Use when implementation landed without a design doc.
argument-hint: "[PR number(s) or description]"
---

# Draft Retroactive DEP

Create a DEP from an already-merged implementation PR when the design was done inline.

## Steps

1. Fetch the PR details: `gh pr view <number> --repo ai-dynamo/dynamo --json title,body,author,mergedAt,additions,deletions,files,comments`
2. Read the PR body for design rationale, architecture decisions, and alternatives considered
3. Read PR comments for review discussion and design feedback
4. If multiple related PRs, gather all of them to understand the full picture
5. Check if any Linear issues are referenced and fetch context from those
6. Determine the area from the files changed
7. Draft a terse DEP that captures:
   - **Summary**: What was built and why
   - **Motivation**: The problem that drove the implementation
   - **Proposal**: The design as implemented (not aspirational — document what was actually built)
   - **Alternate Solutions**: What was considered, even if only mentioned in PR comments
8. Link back to the implementation PRs in frontmatter
9. Mark status as `approved` (the implementation is already merged and reviewed)

## Guidelines

- This is documentation of a decision already made, not a new proposal
- Be terse — the PR body and comments are the primary record; the DEP adds structure and discoverability
- Don't editorialize or suggest changes — document what was decided and why
- Note any open questions or follow-up work mentioned in the PR discussion
