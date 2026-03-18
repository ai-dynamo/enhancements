---
name: dep-related
description: Find and summarize all DEPs related to a feature, capability, or architectural area. Use to understand the full design context around a topic.
argument-hint: "[feature, capability, or keyword]"
---

# Find Related DEPs

Given a feature or capability, find all DEPs that touch it and summarize the decision history.

## Steps

### 1. Search for Related DEPs
- Search all DEP files in `deps/` for the keyword/feature in titles, summaries, and body text
- Check frontmatter `extends` and `replaces` fields for lineage chains
- Check Related Proposals sections for cross-references
- Search for the keyword in GitHub Issues on the enhancements repo: `gh issue list --repo ai-dynamo/enhancements --search "<keyword>"`
- Search for related PRs: `gh pr list --repo ai-dynamo/enhancements --state all --search "<keyword>"`

### 2. Build the Relationship Map
For each related DEP found:
- Read its status, area, authors, and summary
- Identify its relationship to the topic: directly proposes, extends, partially covers, or tangentially related
- Check `extends`/`replaces` chains to find the full lineage
- Note any implementation PRs linked from the DEP

### 3. Summarize the Decision History
Present the related DEPs as a narrative:
- What was the original design intent?
- How has the design evolved across DEPs?
- What alternatives were considered and rejected (across all related DEPs)?
- What is the current state — which DEPs are active vs. superseded?
- Are there gaps — aspects of the feature not covered by any DEP?

## Output Format

```
## DEPs Related to: <feature/capability>

### Decision Timeline
1. **DEP-NNNN: <title>** (<status>, <date>) — <one-line summary of what it decided>
2. **DEP-NNNN: <title>** (<status>, <date>) — <one-line summary>, extends DEP-NNNN
...

### Summary
<2-3 paragraph narrative of the design rationale and evolution>

### Gaps
- <aspects not covered by any existing DEP>

### Related Implementation
- <PR links and Linear issues>
```

## Guidelines
- Follow `extends` and `replaces` chains to their roots
- Include tangentially related DEPs but mark them as such
- If no DEPs exist for the topic, say so and suggest whether one is needed
