# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview
This is the Dynamo Enhancement Proposals (DEP) repository for managing architecture, design, and process decisions for the Dynamo distributed generative AI serving platform. It follows an RFC-style process similar to Kubernetes KEPs and Rust RFCs for proposing and tracking substantial changes to the project.

## Repository Structure
```
enhancements/
├── README.md                    # Introduction and authoring guidelines
├── NNNN-complete-template.md    # Complete DEP template
├── NNNN-limited-template.md     # Simplified DEP template
└── deps/                        # Enhancement proposal documents
    ├── 0000-dep-process.md      # DEP process definition
    ├── 0008-testing-strategy.md # Testing framework and strategy
    ├── 0010-container-strategy.md # Container build optimization
    └── [other numbered DEPs]
```

## Enhancement Proposal Process
1. **Templates**: Start with either `NNNN-complete-template.md` (comprehensive) or `NNNN-limited-template.md` (simplified)
2. **Naming**: Copy template to `deps/0000-my-feature.md` (descriptive name, no number assigned yet)
3. **Sponsorship**: Identify a maintainer/code owner to shepherd the process
4. **Iteration**: Submit draft PR and collaborate with co-authors and sponsor
5. **Review**: Mark ready for review when complete, sponsor sets review date
6. **Approval**: Sponsor merges after successful review and assigns final ID

## When DEPs Are Required
**Generally Required For:**
- New features adding significant functionality
- Changes to public interfaces
- Security vulnerability responses
- Packaging/installation changes
- Architecture or process decisions
- When recommended by maintainers

**Generally Not Required For:**
- Bug fixes without behavior changes
- Documentation updates
- Minor single-module refactors

## DEP Document Structure
All DEPs must include these **required** sections:
- Status, Authors, Category, Sponsor, Required Reviewers, Review Date
- Summary (brief description)
- Motivation (why this is needed)
- Goals and Non Goals
- Implementation Details or Proposal
- Related Proposals (if applicable)
- Alternate Solutions (what was considered and rejected)

## Key Architectural Documents
- **0000-dep-process.md**: Defines the enhancement proposal process itself
- **0008-testing-strategy.md**: Comprehensive testing taxonomy, CI strategy, and test lifecycle
- **0010-container-strategy.md**: Container build optimization and Dockerfile restructuring

## DEP Workflow Skills

Claude Code skills are available for common DEP operations:

- `/dep-create [feature-name]` — Create a new DEP with proper structure. Discusses problem statement first, researches context, then drafts.
- `/dep-status [area|number|all]` — Show status of DEPs grouped by status. Flags stalled proposals and missing sponsors.
- `/dep-review [dep-number|area|all]` — Check if a DEP is still current, superseded, or diverged from implementation. Recommends actions.
- `/dep-related [feature or keyword]` — Find all DEPs related to a feature/capability, trace lineage, and summarize the decision history.
- `/dep-retroactive [PR-number]` — Draft a DEP from an already-merged PR when design was done inline.
- `/dep-triage` — Weekly triage report: stalled DEPs, large PRs without design docs, backlog priorities.

### Writing Guidelines for Agents
- **Be terse** — include only sections needed for the specific proposal
- **Required sections**: Summary, Motivation, Proposal, Alternate Solutions
- **Optional sections**: include only if they add value for this DEP
- **Don't include empty optional sections** — omit them entirely
- **Ground proposals in evidence** — reference actual PRs, issues, code patterns
- **Use RFC-2119 language** in requirements (MUST, SHOULD, MAY)

## Common Commands
Since this is a documentation repository, most work involves:
```bash
# Create new DEP from template
cp NNNN-complete-template.md deps/0000-my-feature.md

# Lint markdown files
pre-commit run --all-files

# Review process via GitHub PRs
git checkout -b feature/my-dep
# Edit and commit changes
git push origin feature/my-dep
# Create PR for review
```

## Development Notes
- All DEPs are written in Markdown following the provided templates
- Use the `pre-commit` framework for linting and formatting validation
- Maintain consistent section ordering as defined in templates
- Include implementation tracking via GitHub issues/PRs when applicable
- DEPs can be retroactively created to document existing decisions