# Build and Dependency Management

**Status**: Approved

**Authors**: [@mc-nv](https://github.com/mc-nv)

**Category**: CI/CD

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: [@saturley-hall](https://github.com/saturley-hall), [@dmitry-tokarev-nv](https://github.com/dmitry-tokarev-nv)

**Review Date**: TBD



### Observation:
Dynamo repository host Dockerfiles which are used as a build environment for different cases.

### Goal:
- Make build orchestration more flexible.
- Spped up the release process.

### Proposal:
1. Start building each project individually and keep results as an artifacts.
2. Stay within lifecycle of the build tool for the given codebase (pip, cargo, etc.).
3. Use artifacts to assemble final products.

### Benefits:
- Removes full rebuilds of the same project.
- Normalizes build process, and git history.
- Speed up testing and deployment.
- Speed up development and build process.
- Minimize usage of the computer resources.
- Release time for innovations.
- Creates more opportunities for the team for collaboration.

> [!NOTE]
> Implementation is different to different projects.
> There is no straight forward way to implement this, this process is tend to evolve over time.


High level schema is shown below.

<img src="./0006/workflow.0.svg">

