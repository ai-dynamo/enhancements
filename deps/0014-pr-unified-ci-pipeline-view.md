# Enhancement Proposal: Unified PR Pipeline View

## Summary
Introduce a unified Pull Request (PR) pipeline view in GitHub that aggregates all workflows triggered by a PR into a single visual pipeline, rather than listing them as separate rows in the Checks section.

This will improve clarity, reduce cognitive load, and make CI results easier to interpret — especially for large teams and complex repositories.

---

## Motivation
Currently, GitHub Actions workflows triggered by a PR appear as independent rows under the **Checks** tab. There is no way to visualize a consolidated pipeline or stage progression across workflows. This hinders developer productivity and increases overhead when diagnosing CI failures.

### Problems with the current experience:
- Multiple unrelated rows under **Checks** for each workflow  
- No stage grouping (e.g., Lint → Test → Build → Security)  
- No way to visualize dependencies across workflows  
- Harder to manage required checks  
- Clutter increases with scale  

---

## Goals
- Provide a **unified view** of PR CI as a pipeline  
- Show workflow **dependencies** in a DAG / stage progression  
- Group related jobs and workflows into **logical stages**  
- Enable improved **debugging** and pipeline clarity  

---

## Current State (Before)

**PR Checks view (fragmented):**
```
Pull Request #482
Checks
────────────────────────────
✔ lint.yml
✔ unit-tests.yml
✖ integration-tests.yml
✔ build.yml
✔ security-scan.yml
✔ docker-build.yml
✔ docs-check.yml
```

**Limitations:**
- Separate rows per workflow  
- No visual pipeline  
- No indication of blocked stages  

---

## Proposed UX (After)

### Unified PR Pipeline View
**Pull Request #482 • CI Pipeline Status: ❌ Failed**

**Pipeline (Grouped by Stage)**
```
[Stage: Code Quality]
   ✅ Lint
   ✅ Docs Check

[Stage: Testing]
   ✅ Unit Tests
   ❌ Integration Tests

[Stage: Build]
   ⏸ Docker Build  (blocked)

[Stage: Security]
   ⏸ Security Scan (blocked)
```


**Visual Pipeline (ASCII DAG)**
```
       ✅ Lint ──────────────┐
       ✅ Docs Check ────────┤
                            │
                     ✅ Unit Tests
                            │
                     ❌ Integration Tests
                            │
                ┌───────────┴───────────┐
                │                       │
           ⏸ Docker Build          ⏸ Security Scan
             (blocked)                (blocked)
```
### Highlights
- **Clear logical progression:** Code Quality → Testing → Build → Security  
- **Blocked stages** are visually indicated  
- **Single CI/Checks entry** reduces clutter  

---

### Benefits

| Area             | Before                         | After                                |
|-----------------|--------------------------------|--------------------------------------|
| PR UI clarity    | Many unrelated rows            | One unified pipeline view            |
| Debugging       | Open each workflow separately  | See blocked/failing stages upfront   |
| Required checks | Per-workflow                   | Per pipeline with stage grouping     |
| Team scale      | UI clutter and overload        | Clean, maintainable at scale         |
| Cognitive load  | Higher                         | Lower                                |

---

### Non-Goals
- Change existing workflow execution semantics  
- Remove GitHub Actions functionality  
- Replace existing dependency definitions (**this is a UI enhancement**)  

---

### Implementation Notes
- Current workaround: using a single orchestrator workflow with `workflow_call`, but this requires restructuring CI.  
- A native unified pipeline view would eliminate that need.

