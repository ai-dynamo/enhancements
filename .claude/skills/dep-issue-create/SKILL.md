# Skill: Create a DEP as a GitHub Issue

## Purpose

Create a new Dynamo Enhancement Proposal (DEP) as a GitHub Issue on
`ai-dynamo/dynamo`. The issue number becomes the DEP number.

## When to Use

When the user wants to propose a new feature, architecture change, or
process improvement via the issue-based DEP workflow.

## Workflow

1. **Gather required fields** from the user (prompt if missing):
   - **Category**: Feature, Architecture, Process, or Guidelines
   - **Sponsor**: GitHub handle of the PIC or maintainer
   - **Summary**: One-paragraph description
   - **Motivation**: Why this change is needed
   - **Proposal**: Detailed description of the proposed change
   - **Alternate Solutions**: Other approaches considered

2. **Determine the area label** based on proposal content:
   `area/frontend`, `area/router`, `area/backend`, `area/kvbm`,
   `area/bindings`, `area/deployment`, `area/observability`,
   `area/cicd`, `area/process`, `area/cross-cutting`

3. **Decide template**: full or lightweight.
   Use lightweight if only Summary, Motivation, and Proposal are needed.

4. **Create the issue**:

```bash
gh issue create \
  --repo ai-dynamo/dynamo \
  --title "DEP: <short descriptive title>" \
  --label "dep:draft" \
  --label "area/<area>" \
  --assignee "<sponsor>" \
  --body "$(cat <<'EOF'
## Category
<category>

## Sponsor
@<sponsor>

## Summary
<summary>

## Motivation
<motivation>

## Proposal
<proposal>

## Alternate Solutions
<alternates>

## Requirements
<requirements or N/A>

## References
<references or N/A>

## Related Proposals
<related DEP issue numbers or N/A>
EOF
)"
```

5. **Report** the created issue number and URL to the user.

## Notes

- The issue body IS the spec — treat it as a living document.
- `dep:draft` is applied automatically. Sponsor changes to
  `dep:under-review` when ready.
- For lightweight DEPs, omit Requirements, References, and Related
  Proposals sections.
