---
name: dep-status
description: Show status of DEPs — all, by area, or a specific DEP. Use to check what proposals are in progress, stalled, or need attention.
argument-hint: "[area, dep-number, or 'all']"
---

# Check DEP Status

Provide a summary of Dynamo Enhancement Proposals.

## Steps

1. Search `deps/` for all `.md` files (exclude templates, session summaries, temp files)
2. Read each file's header to extract: Status, Title, Authors, Sponsor, Category, Review Date
3. If argument is an area name, filter to that area only
4. If argument is a number, show detailed status for that specific DEP
5. Otherwise show all DEPs grouped by status

## Output Format

For each DEP show one line: `DEP-NNNN: <title> [<status>] — <authors> (sponsor: <sponsor>)`

Group by status in this order: Under Review, Draft, Approved, Deferred, Rejected, Replaced

Highlight:
- DEPs with no sponsor assigned
- DEPs with review dates in the past
- Draft DEPs older than 30 days with no activity

## Also Check

- Open PRs in the enhancements repo: `gh pr list --repo ai-dynamo/enhancements --state open`
- Flag PRs with no reviewers assigned or no activity in 30+ days
