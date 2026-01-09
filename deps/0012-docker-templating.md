# Dockerfile Templating

**Status**: [x] Draft | Under Review | Approved | Replaced | Deferred | Rejected

**Authors**: Dillon Cullinan

**Category**: Process

**Sponsor**:

**Required Reviewers**:

**Review Date**:

**Pull Request**: N/A

**Implementation PR / Tracking Issue**: [Link to Pull Request or Tracking Issue for Implementation]

# Summary

Right now, we use a script called `build.sh` to handle `--build-arg`s to our Dockerfile. However, even with only a few variations in our image, this script has bloated to over 1000 lines, and is currently unsustainable.

Dockerfile templating is a clean solution that can support what we currently do, as well as simplify our expansion in the future. For now, we have less than 10 Dockerfiles to manage. However, I can foresee that we will need to support more variations, whether its CUDA versions, Operating systems, or even different OS versions.

Instead of having a singular large Dockerfile and `build.sh` that handles all these things, we can use a slim implementation of templates to generate what we need on demand.

# Motivation

Dockerfile templating is a solution meant to replace the `container/build.sh` script in the Dynamo repo

- This script has grown to over 1000 lines of code
  - `build.sh` was supposed to simplify things, but has only introduced complexities and hard to understand code
  - When implementing a feature, updating `build.sh` can be even more time consuming than the feature itself
  - `build.sh` is a glorified wrapper script around the `docker build` command, most of what it does is append `--build-arg`s
    - ie `container/build.sh <args>` always results in a `docker build <args>` with no additional value add, its 95% conditional variable declarations
    - Additionally, if we ever want to use more `docker build` features, we have to re-implement them in `build.sh`
  - The order of how things are defined actually matters
    - Adding a new argument too early in the script could break things after it
    - Adding it too early in the script could break things that may have required it earlier.
    - Keeping track of the order is unnecessarily confusing, and massively slows down development
- `build.sh` structure allows developers add things in various, chaotic ways
  - Dockerfile arguments are spread across various layers of abstraction
    - Some args are defined in CI workflows, by passing additional `--build-args` to `build.sh`
    - Some args are defined in `build.sh`, sometimes within deep logical `if` statements
    - Some args just use the default of what is in the `Dockerfile` being referenced
  - Sometimes `build.sh` isn't even updated, so it doesn't function as source of truth anymore (which was its original purpose)
- Sets us up for better expansion in the future
  - Simple way to support slightly different layers based on configuration
    - For example, for more OS support... `RUN apt-get install` for Ubuntu, and `RUN dnf install` for CentOS

# Goals

* Bring structure to chaos

* Declarative > Imperative implementation

* Reduce mental load when implementing new features

* Removes redundancy in current Dockerfiles

* Improve extensibility


# Proposal

## Key Components

- `context.yaml` (~about ~90 lines) [Link to File](https://github.com/ai-dynamo/dynamo/blob/poc-dockerfile-templating/container/context.yaml)
  - Source of truth for all Dockerfile ARGs upon generation
  - Dictionary data structure for sorting, everything in one place
- `render.py` (~60 lines) [Link to File](https://github.com/ai-dynamo/dynamo/blob/poc-dockerfile-templating/container/render.py)
  - Short python script with minimal arguments
    - ie. framework, platform, target, cuda_ver, OS, etc
- `templates/` (lines N/A) [Link to Folder](https://github.com/ai-dynamo/dynamo/tree/poc-dockerfile-templating/container/templates)
  - Folder of templates that represent docker stages (such as our `wheel_builder` stage)
  - Current list
    - wheel_builder
    - {trtllm, vllm, sglang} frameworks
    - {trtllm, vllm, sglang} ruintime
    - dev
- `Dockerfile.template` (30 lines) [Link to File](https://github.com/ai-dynamo/dynamo/blob/poc-dockerfile-templating/container/Dockerfile.template)
  - File that determines the stage templates to include based on args passed to render.py
  - Should never be long... >100 lines

NOTE: Notice, `build.sh` 1000+ lines is being replaced by a total of <200 lines.

## Change Expectations

| File | Frequency | Conditions |
| --- | --- | --- |
| `config.yaml` | Often, expected | Changes to ARGs<br/>Change to Support Matrix |
| `templates/` | Often, expected | Adding layers, changing layers, etc (same as Dockerfile) |
| `Dockerfile.template` | Rare, major structural changes | Changing order of stages<br/>Removal or additional of stages to structure |
| `render.py` | Very rare, only functional changes | QoL improvements for CI (informational)<br/>New support matrix keys (OS, CUDA, etc) |

# Previous Concerns

We talked about this solution already for a while, and there were concerns brought up.

### Q: If we take this route, is there no turning back?

No, we should be able to easily revert to our previous state. The templating solution is merely a tool to generate the Dockerfiles that we are building. The process of turning back would actually be extremely easy. Generate our Dockerfiles using the templating solution, then remove all of the template related files listed above. With that, we should be back to plain old Dockerfiles.

We will not be able to recover `build.sh`. But this file is the problem in the first place.


### Q: Isn't templating also just a wrapper around the Dockerfiles?

I wouldn't consider templating as a wrapper. `build.sh` was a wrapper script around `docker build`, it managed the `<args>` that would be sent to `docker build` using A LOT of if statements. This templating solution doesn't manage how `docker build` is invoked at all. It is a declarative way to generate a `Dockerfile`, then the user invokes `docker build` however they please.

The key difference here is that we are NOT controlling how a user is interacting with another CLI tool, therefore, we are not responsible for making our script support that variety of ways a user could invoke that CLI tool.

Let's say I want to update `Dockerfile.sglang` by adding new `RUN` layers and a new `ARG`:
Wrapper Solution:
- Update `Dockerfile.sglang` with new layers and the new `ARG` (perfectly fine)
  - `RUN` stage lines
  - `ARG` line (but its empty value because its controlled via `build.sh`)
- Additionally, make changes to `build.sh` so that `docker build` is invoked the way we want
  - Add the argument to script options
  - Add the argument to script help output
  - Add if statement flow control to add the value to `--build-args`
  - Also, make sure you add it at the right lines, or everything breaks?

Template Solution:
- Update `sglang` template stage
  - `RUN` stage lines
  - `ARG` line that references the `context.yaml` key
- Single line change in `context.yaml` to add a new variable under the `sglang` key

# Alternate Solutions

## Alt #1 Docker Bake

**Pros:**

- Built in way of supporting different args to different Dockerfiles

**Cons:**

- Does not solve the problem of differing layers based on support matrix (ie. Operating systems have diff `RUN`)

**Notes:**

This solution was being tested by Harrison, however it is facing some caching challenges and may be slower than just templating everything into a single Dockerfile. With a warm cache, speed is not an issue, however our current state of infrastructure uses ECR caches and doesn't support warm local caches.
