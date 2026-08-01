# Layered YAML Configuration Support

**Status**: Draft

**Authors**: @jihao

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Add layered YAML configuration support to Dynamo using OmegaConf for config composition. Config files support single-parent inheritance, interpolation across the merged config tree, a flat root-level flag namespace, and a strict precedence model (CLI > env > config file > code defaults). All values flow through the existing argparse and backend parser paths for final type conversion and validation, keeping one source of truth for parsing semantics.

# Motivation

Dynamo's current configuration model relies on CLI flags, environment variables, and code defaults. This works for small launches but scales poorly for model families and deployment variants that share most settings and differ only in a few overrides.

Operators managing multiple model configurations today must duplicate long CLI invocations or maintain wrapper scripts that are difficult to review, version, and reuse. A file-based layered config system addresses this by enabling config files that are easy to check in, review, and reuse; layered inheritance so a child config can override a parent; and interpolation so one value can be derived from another without duplication.

## Goals

- Enable declarative, version-controlled configuration for Dynamo deployments.
- Support layered inheritance so common settings are defined once and overridden per variant.
- Support interpolation across the merged config tree to eliminate duplication.
- Preserve the existing argparse and backend parser flow as the single source of truth for type conversion, validation, and parsing semantics.
- Maintain full backward compatibility when no config file is specified.

### Non Goals

- Multiple parent inheritance.
- Hydra integration or config groups.
- Custom OmegaConf resolvers or environment-backed interpolation (`${oc.env:...}`).
- Rust-side config loading.
- YAML emission as a public interface.

## Requirements

### REQ 1 Config File Selection

The system **MUST** support selecting a config file via `--dyn-config` CLI flag or the `DYN_CONFIG` environment variable. CLI **MUST** take precedence over the environment variable for file selection.

### REQ 2 Single-Parent Inheritance

Each config file **MUST** support at most one parent via a top-level `parent` key. Parent paths **MUST** be resolved relative to the child file. The parent chain **MUST** be acyclic.

### REQ 3 Interpolation

The system **MUST** support OmegaConf node interpolation across the fully merged config tree. Interpolation **MUST** happen after the precedence merge so that higher-precedence overrides participate in resolved values.

### REQ 4 Precedence Order

Final precedence **MUST** be:

```text
CLI flags > environment variables > config file > code defaults
```

### REQ 5 Parser-First Validation

Config-sourced values **MUST** flow through the same argparse or backend parser path as CLI values. The system **MUST NOT** introduce a separate hand-written coercion layer.

### REQ 6 Flat Root-Level Namespace

Real config flags **MUST** live at the top level of the YAML file. Organizational sections such as `shared`, `runtime`, or `vllm` **MUST NOT** be part of the config format for real flags.

### REQ 7 Backward Compatibility

Existing behavior **MUST** remain unchanged when both `--dyn-config` and `DYN_CONFIG` are absent. No OmegaConf merge path runs in that case.

# Proposal

## Overview

Use OmegaConf only for config composition: loading YAML, resolving the single-parent chain, merging layered config trees, and resolving interpolation.

Keep argparse and backend-native parsers as the source of truth for type conversion, boolean action semantics, `choices` validation, `nargs` handling, and backend-specific parsing quirks.

The key idea is a six-step pipeline:

1. Bootstrap `--dyn-config` and load the YAML parent chain with OmegaConf.
2. Run a permissive collector parse to learn which CLI flags were explicitly set.
3. Build one unresolved canonical config tree from code defaults, YAML, env vars, and explicit CLI flags using recorded source provenance.
4. Resolve interpolation on that merged tree with OmegaConf.
5. Compile the final resolved tree back into synthetic CLI arguments.
6. Feed those synthetic arguments through the existing Dynamo and backend parser flow.

This keeps one strict parsing and validation path while adding config files and interpolation, without implementing separate parser passes for each precedence layer.

## Why OmegaConf

OmegaConf provides layered merge and interpolation across the merged config tree directly. The default merge behavior is well suited:

- Dicts deep-merge.
- Lists replace.
- Last source wins.

This matches the intended "parent base config + child override" model.

The following OmegaConf features are explicitly excluded:

- `oc.env` and custom resolvers.
- Hydra.
- OmegaConf as a schema or parser replacement.

This keeps config files self-contained and preserves Dynamo's existing parser ownership.

## File Format

### Reserved Metadata Keys

Only two top-level keys are reserved by the config system:

- `version` (required, must be a supported schema version)
- `parent` (optional, string path to parent config)

Every other top-level key must be a real registered flag for the selected entrypoint.

Config files do not support:

- organizational sections such as `shared`, `runtime`, or `vllm`
- interpolation-only helper keys

Real flags always live at the root level of the file.

### Single Parent

Each file may specify at most one parent:

```yaml
parent: ./base.yaml
```

Parent paths are resolved relative to the child file. The parent chain must be acyclic. Merge order is parent first, child second.

### Dash-Only Keys

Top-level Dynamo config keys must use dash form:

```yaml
discovery-backend: etcd
```

### Flat Root-Level Flag Namespace

All real flags share one flat root-level namespace in the config file.

That means:

- a real flag such as `namespace` appears as `namespace: ...` at the top level
- interpolation references real flags by their top-level names
- there is no flattening step from user-defined sections into flags
- unknown top-level keys are rejected

Example:

```yaml
namespace: llama-prod
discovery-backend: etcd
served-model-name: ${namespace}-70b
```

This emits the flags: `namespace`, `discovery-backend`, `served-model-name`.

### Structured Flag Payloads

Most flags are scalars, booleans, or lists.

Some flags accept structured payloads. In YAML, those flags are still root-level keys, but their values may be mappings or lists.

Example:

```yaml
override-engine-args:
  kv_cache_config:
    free_gpu_memory_fraction: 0.7
    enable_block_reuse: false
  tensor_parallel_size: 4
```

Important rules:

- the root key is still the real flag name, for example `override-engine-args`
- nested payload keys are not treated as separate Dynamo flags
- nested payload keys use the native format expected by that flag's consumer
- during argv compilation, structured payloads are serialized into the flag's native CLI form
- if a higher-precedence source overrides the same root-level flag, it replaces the entire payload

### Interpolation Semantics

Interpolation is standard OmegaConf node interpolation over the fully merged config tree. Both dot and bracket notation are supported. Bracket notation can be used when a path segment contains a dash, including top-level flag names:

```yaml
namespace: llama-prod
served-model-name: ${namespace}-70b
```

Supported: absolute and relative interpolation.

Not supported: environment resolvers (`${oc.env:HOME}`), custom resolvers. The config file resolves from values within the merged config tree only.

## Precedence and Resolution

```text
CLI flags > environment variables > config file > code defaults
```

Interpolation happens after those four sources are merged. This means interpolation sees the final winning value, not just the value from YAML.

Example:

```yaml
version: 1
namespace: base
served-model-name: ${namespace}-70b
```

With `DYN_NAMESPACE=test` and `--namespace prod`, the final resolved value of `served-model-name` is `prod-70b`.

## Canonical Internal Model

From the OmegaConf tree, the loader derives a recognized-flag index mapping each recognized flag name to its top-level OmegaConf key. Interpolation uses top-level flag names and any nested paths inside structured payload values. Parser compilation uses top-level recognized flag names only.

# Implementation Details

## 1. Argument Metadata Registry

A registry of argument specs for Dynamo-owned args and each backend adapter. Each spec includes: canonical flag name, argparse dest, env var name, code default, action kind, type callable, `choices`, and `nargs`.

This registry is additive and does not change current parser behavior. `add_argument()` and `add_negatable_bool_argument()` capture most of this. The registry supports building the parser in collector mode and final mode.

## 2. Bootstrap `--dyn-config`

Before the main parser runs, a minimal bootstrap parse finds `--dyn-config` or falls back to `DYN_CONFIG`. If neither is present, the entire config path is skipped and today's behavior is preserved.

## 3. Load Parent Chain with OmegaConf

Load the selected file and walk the single-parent chain:

- Validate top-level keys.
- Detect parent cycles.

Merge parent first, child last with `OmegaConf.merge()`. Lists follow OmegaConf default behavior (source list replaces target list).

## 4. Permissive Collector Parse for CLI Provenance

Build the same Dynamo and backend parsers in a special "collector" mode:

- No env lookup and no code defaults.
- `default=argparse.SUPPRESS`.
- `required=False` for the collector pass.
- Only explicitly provided CLI values appear in the parsed output.

This answers: which flags did the user explicitly set on the command line? Required arguments are enforced only in the final strict parse.

## 5. Build Merged Unresolved Config Tree

Build one unresolved merged tree by projecting each source into the recognized-flag index in precedence order:

1. **Code defaults**: Project from the argument metadata registry into the root-level config tree.
2. **Config file**: The unresolved OmegaConf tree from the YAML parent chain.
3. **Environment variables**: Read only env vars that are actually set for registered arguments, using existing env decoding rules and projecting them to root-level flag keys.
4. **Explicit CLI flags**: Values from the collector parse, projected to root-level flag keys.

## 6. Resolve Interpolation

```python
resolved = OmegaConf.to_container(merged_cfg, resolve=True)
```

Precedence merge happens first; interpolation resolution happens second.

## 7. Compile Resolved Tree to Synthetic argv

Compile recognized root-level flags into synthetic CLI arguments:

- Scalar: `--flag value`
- Bool: `--flag` or `--no-flag`
- List: repeated CLI values according to the target action
- Structured payload: flag-specific serialization such as JSON when required by the target CLI

The compiler emits canonical long-form flags only.

## 8. Parse Synthetic argv Through Existing Parser Flow

The final synthetic argv is parsed by the same Dynamo and backend parsers that handle normal CLI input. This ensures config values get the same type conversion, `choices` validation, boolean action semantics, and backend-specific parsing as CLI values.

Parser ownership is determined by the existing parse flow, not by section names in the config file.

## Backend Integration

### vLLM

Compile resolved root-level flags into CLI arguments. Dynamo parses known flags first, then vLLM parses remaining flags. For options whose CLI form expects a serialized payload (e.g., `kv-transfer-config`), the compiler emits the correct string form.

### SGLang

Compile resolved root-level flags into CLI arguments. Dynamo parses known flags first, then SGLang parses remaining flags. The design must not introduce a second competing `--config` flag in the SGLang parser path.

### TRT-LLM

Compile resolved root-level flags into CLI arguments. Dynamo parses known flags first, then TRT-LLM parser and dynamic flag handling parse remaining flags.

`--trtllm.*` dotted CLI flags are treated as CLI-only sugar for building the nested `override-engine-args` payload.

In YAML, the canonical representation is the root-level `override-engine-args` flag with a nested mapping value:

```yaml
override-engine-args:
  kv_cache_config:
    free_gpu_memory_fraction: 0.7
```

During argv compilation, that mapping is serialized to the JSON string expected by `--override-engine-args`.

## Validation Rules

These checks are enabled in the initial implementation:

**File-level validation:**

- `version` is required and must be supported.
- `parent` must be a string when present.
- Parent cycles are errors.
- Non-metadata top-level keys must be recognized flags for the selected entrypoint.
- Nested keys inside structured payload values are validated according to the payload format of the owning flag, not by the generic dash-style rule.

**Interpolation validation:**

- Unresolved interpolation is an error.
- Interpolation cycles are an error.

**Parser validation:**

- `type=...`, `choices=...`, boolean action semantics, `nargs`, backend-specific parse and validation logic, unknown flag rejection, and enforcement of required arguments all run on the final synthetic argv.

**Collector parse behavior:**

- `default=argparse.SUPPRESS` so only explicitly provided CLI values appear.
- `required=True` is temporarily treated as `False`.
- Requiredness is enforced only in the final strict parse.

## Deferred to Implementation

- Exact API surface of the argument metadata registry.
- Serialization format details for backend-specific compound flags.

## Example

Base config (`llama-base.yaml`):

```yaml
version: 1
namespace: llama-prod
discovery-backend: etcd
model: meta-llama/Llama-3.3-70B-Instruct
tensor-parallel-size: 4
served-model-name: ${namespace}-70b
```

Child config (`llama-prefill.yaml`):

```yaml
version: 1
parent: ./llama-base.yaml

disaggregation-mode: prefill
max-num-batched-tokens: 65536
```

Launch:

```bash
DYN_NAMESPACE=test python -m dynamo.vllm \
  --dyn-config ./llama-prefill.yaml \
  --namespace prod
```

Final outcome:

- `namespace = prod` (CLI wins over env and file)
- `served-model-name = prod-70b` (interpolation sees CLI value)
- `model = meta-llama/Llama-3.3-70B-Instruct`
- `disaggregation-mode = prefill`

# Alternate Solutions

## Alt 1 Hydra-Based Configuration

**Pros:**

- Mature config composition framework with config groups, overrides, and sweep support.

**Cons:**

- Hydra takes over the application entry point and imposes its own CLI override syntax.
- Heavy dependency that conflicts with Dynamo's existing argparse-based parser flow.
- Config groups and sweep are not needed for the initial use case.

**Reason Rejected:**

- Dynamo already has a well-established parser flow. Hydra would require either replacing it or running two parallel config systems. OmegaConf (which Hydra uses internally) provides the needed merge and interpolation primitives without the entry-point takeover.

## Alt 2 Standalone YAML Parsing Without OmegaConf

**Pros:**

- No external dependency for config loading.

**Cons:**

- Requires implementing layered merge and interpolation from scratch.
- Interpolation across a merged tree is non-trivial to implement correctly (cycle detection, resolution ordering, bracket notation for dashed keys).

**Reason Rejected:**

- OmegaConf already solves the two hardest problems (layered merge and interpolation) and is a lightweight, well-tested library. Building this from scratch adds implementation risk without meaningful benefit.

## Alt 3 Separate Parser Passes Per Precedence Layer

**Pros:**

- Each precedence layer (code defaults, file, env, CLI) gets its own independent parse, which is conceptually simple.

**Cons:**

- Requires standalone parser implementations for the env and file layers that duplicate argparse semantics (type conversion, boolean actions, nargs).
- Maintaining parity across multiple parser implementations is error-prone.

**Reason Rejected:**

- The parser-first approach achieves the same precedence model by projecting all sources into a single tree and parsing once through the existing flow. This avoids duplicating parser semantics across layers while preserving source provenance via the collector parse.

# Background

## References

- [OmegaConf documentation](https://omegaconf.readthedocs.io/)

