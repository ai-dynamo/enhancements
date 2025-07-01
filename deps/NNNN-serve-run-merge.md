# Dynamo run and serve merged

**Status**: Draft

**Authors**: grahamk

**Category**: Architecture

**Sponsor**: Literally everybody. Multiple times.

**Required Reviewers**: Your name here

**Review Date**: [Date for review]

**Pull Request**: [Link to Pull Request of the Proposal itself]

**Implementation PR / Tracking Issue**:  https://github.com/ai-dynamo/dynamo/issues/1647 and others

# Summary

We propose to merge `dynamo run` and `dynamo serve` to make the Dynamo CLI more intuitive to use.

There are two parts:
* the CLI (e.g. `dynamo serve`)
* the worker (`await register_llm(..)`) e.g. https://github.com/ai-dynamo/dynamo/blob/main/docs/guides/dynamo_run.md#writing-your-own-engine-in-python

I am focused on the CLI side right now.

First port `dynamo-run` from Rust to Python. `dynamo-run` becomes a Rust library with Python bindings. `dynamo run` (with a space!) will call those Python bindings instead of shelling to `dynamo-run`. The `dynamo-run` binary moves into the examples folder as a Rust CLI example, but is essentially gone.

Next we allow `dynamo serve` to take the same command line arguments as `dynamo run`: `dynamo serve in=http out=llamacpp <gguf>`. The old syntax remains, `dynamo serve -f <python things>` still works.

If we can get to here, we remove the `run` verb. Great success!

Next the real work starts, but because it's Python we can all work on it together. We need to:
* merge the yaml file's settings (which came from dynamo serve) with the command line params (which came from dynamo-run),
* continue simplifying `serve`,
* remove all the deployment bento-style pieces from `serve` (e.g. the `resources` key in decorator) to have a clean `deploy` / `serve` layering,
* and so on.

