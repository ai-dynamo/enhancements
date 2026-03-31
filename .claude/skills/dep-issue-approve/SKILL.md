# Skill: Approve a DEP Issue

## Purpose

Post an approval on a DEP issue and maintain the approval checklist.

## When to Use

When a reviewer or PIC is ready to approve a DEP that is under review.

## Workflow

1. **Verify the issue is under review**:

```bash
gh issue view <number> --repo ai-dynamo/dynamo --json labels
```

   Confirm `dep:under-review` label is present.

2. **Post the approval comment**:

```bash
gh issue comment <number> --repo ai-dynamo/dynamo --body "/approve"
```

3. **Check the approval checklist** (pinned or first comment) to see
   if all required reviewers have approved.

4. **If all approvals are in**, update the label:

```bash
gh issue edit <number> --repo ai-dynamo/dynamo \
  --remove-label "dep:under-review" \
  --add-label "dep:approved"
```

5. **Report** the approval status to the user.

## Notes

- `/approve` comments are searchable across the repo for audit:
  `gh search issues --repo ai-dynamo/dynamo "/approve" in:comments`
- If a substantive spec revision occurs after approval, the PIC should
  re-request approval by resetting the checklist and notifying
  reviewers.
- The PIC is responsible for maintaining the approval checklist.
