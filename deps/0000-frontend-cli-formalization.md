# DEP: Frontend CLI Formalization

| Status | Draft |
|--------|-------|
| **Authors** | @nnshah1 |
| **Category** | Process |
| **Sponsor** | TBD |
| **Required Reviewers** | TBD |
| **Review Date** | TBD |
| **Related PRs** | #54 |

## Summary

This DEP defines the CLI argument formalization strategy for the Dynamo Frontend component. It addresses validation gaps, environment variable coverage, naming consistency, and argument grouping needed for a stable 1.0 release.

## Motivation

The Frontend CLI has grown to **31 arguments** across 5 categories. While functional, several issues affect production readiness:

1. **Missing Validation**: 15+ arguments lack input validation (port ranges, float bounds)
2. **Incomplete Environment Variables**: 10+ arguments missing `DYN_*` env var support
3. **Inconsistent Patterns**: Mix of boolean styles, naming conventions
4. **Implicit Dependencies**: KV router args accepted in non-KV modes
5. **Poor Discoverability**: No argument grouping, verbose help text

### Current State

| Category | Args | Critical Issues |
|----------|------|-----------------|
| Server Config | 4 | No port validation, missing TLS env vars |
| Router Config | 13 | No range validation, inconsistent booleans |
| Model Config | 3 | Missing mutual exclusivity docs |
| Metrics | 5 | Verbose help, missing validation |
| Backend | 4 | Undocumented dependencies |

## Goals

1. Add comprehensive input validation for all arguments
2. Ensure all arguments have environment variable support
3. Standardize boolean argument patterns
4. Validate implicit feature dependencies at parse time
5. Organize arguments into logical groups with clear help text
6. Document all arguments with stability markers

## Non-Goals

1. Configuration file support (future enhancement)
2. Interactive configuration wizard
3. Breaking changes to existing argument names

---

## Argument Inventory

### Category 1: Server Configuration (4 args)

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--http-host` | str | `0.0.0.0` | `DYN_HTTP_HOST` | None (valid) |
| `--http-port` | int | `8000` | `DYN_HTTP_PORT` | **Add**: 1-65535 range |
| `--tls-cert-path` | Path | None | **Add**: `DYN_TLS_CERT_PATH` | Mutual with key |
| `--tls-key-path` | Path | None | **Add**: `DYN_TLS_KEY_PATH` | Mutual with cert |

**Issues:**
- Port validation missing
- TLS env vars missing
- Mutual requirement not in help text

---

### Category 2: Router Configuration (13 args)

#### Core Router

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--router-mode` | enum | `round-robin` | `DYN_ROUTER_MODE` | Valid (has choices) |

#### KV Router Options (only valid with `--router-mode kv`)

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--kv-cache-block-size` | int | None | `DYN_KV_CACHE_BLOCK_SIZE` | **Add**: >0 |
| `--kv-overlap-score-weight` | float | `1.0` | `DYN_KV_OVERLAP_SCORE_WEIGHT` | **Add**: ≥0 |
| `--kv-events/--no-kv-events` | bool | True | `DYN_KV_EVENTS` | Valid (BooleanOptionalAction) |

#### Router Tuning

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--router-temperature` | float | `0.0` | `DYN_ROUTER_TEMPERATURE` | **Add**: 0-1 range |
| `--router-ttl` | float | `120.0` | `DYN_ROUTER_TTL` | **Add**: >0 |
| `--router-max-tree-size` | int | 2^20 | `DYN_ROUTER_MAX_TREE_SIZE` | **Add**: >0 |
| `--router-prune-target-ratio` | float | `0.8` | `DYN_ROUTER_PRUNE_TARGET_RATIO` | **Add**: 0-1 range |
| `--router-replica-sync` | bool | False | **Add**: `DYN_ROUTER_REPLICA_SYNC` | Mode check |
| `--router-snapshot-threshold` | int | 1000000 | **Add**: `DYN_ROUTER_SNAPSHOT_THRESHOLD` | **Add**: >0 |
| `--router-reset-states` | bool | False | **Add**: `DYN_ROUTER_RESET_STATES` | Mode check |
| `--track-active-blocks/--no-track-active-blocks` | bool | True | **Add**: `DYN_ROUTER_TRACK_ACTIVE_BLOCKS` | Mode check |

#### Worker Load Detection

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--active-decode-blocks-threshold` | float | None | **Add**: `DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD` | **Add**: 0-1 |
| `--active-prefill-tokens-threshold` | int | None | **Add**: `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD` | **Add**: >0 |

**Issues:**
- 13 args lack validation
- 6 args missing env vars
- Inconsistent boolean patterns (5 use `store_true`, 1 uses `BooleanOptionalAction`)
- No mode dependency validation

---

### Category 3: Model Configuration (3 args)

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--model-name` | str | None | None | Has validator (non-empty) |
| `--model-path` | str | None | None | Has validator (valid dir) |
| `--namespace` | str | None | `DYN_NAMESPACE` | **Add**: non-empty if set |

---

### Category 4: Metrics & Observability (5 args)

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--metrics-prefix` | str | None | `DYN_METRICS_PREFIX` | **Add**: alphanumeric |
| `--kserve-grpc-server` | bool | False | **Add**: `DYN_KSERVE_GRPC_SERVER` | None |
| `--grpc-metrics-port` | int | 8788 | **Add**: `DYN_GRPC_METRICS_PORT` | **Add**: 1-65535 |
| `--custom-backend-metrics-endpoint` | str | `nim.backend.runtime_stats` | `CUSTOM_BACKEND_ENDPOINT` | **Add**: format check |
| `--custom-backend-metrics-polling-interval` | float | 0 | `CUSTOM_BACKEND_METRICS_POLLING_INTERVAL` | **Add**: ≥0 |

**Issues:**
- Port validation missing
- Help text too verbose (>200 chars)
- gRPC port only valid with `--kserve-grpc-server`

---

### Category 5: Backend Configuration (4 args)

| Argument | Type | Default | Env Var | Validation Needed |
|----------|------|---------|---------|-------------------|
| `--store-kv` | enum | `etcd` | `DYN_STORE_KV` | Valid (has choices) |
| `--request-plane` | enum | `tcp` | `DYN_REQUEST_PLANE` | Valid (has choices) |
| `--dump-config-to` | str | None | (external) | None |
| `--exp-python-factory` | bool | False | None | Mark experimental |

---

## Formalization Requirements

### 1. Input Validation

Add comprehensive validation for all numeric arguments:

```python
def validate_args(args):
    errors = []

    # Port ranges
    if args.http_port and not (1 <= args.http_port <= 65535):
        errors.append("--http-port must be 1-65535")
    if args.grpc_metrics_port and not (1 <= args.grpc_metrics_port <= 65535):
        errors.append("--grpc-metrics-port must be 1-65535")

    # Float bounds [0, 1]
    for arg, name in [
        (args.router_temperature, "--router-temperature"),
        (args.router_prune_target_ratio, "--router-prune-target-ratio"),
        (args.active_decode_blocks_threshold, "--active-decode-blocks-threshold"),
    ]:
        if arg is not None and not (0.0 <= arg <= 1.0):
            errors.append(f"{name} must be 0-1")

    # Positive numbers
    for arg, name in [
        (args.router_ttl, "--router-ttl"),
        (args.router_max_tree_size, "--router-max-tree-size"),
        (args.router_snapshot_threshold, "--router-snapshot-threshold"),
        (args.kv_cache_block_size, "--kv-cache-block-size"),
        (args.active_prefill_tokens_threshold, "--active-prefill-tokens-threshold"),
    ]:
        if arg is not None and arg <= 0:
            errors.append(f"{name} must be > 0")

    # Non-negative
    if args.kv_overlap_score_weight is not None and args.kv_overlap_score_weight < 0:
        errors.append("--kv-overlap-score-weight must be >= 0")

    if errors:
        raise ValueError("\n".join(errors))
```

### 2. Mode Dependency Validation

Reject KV-specific arguments when not in KV mode:

```python
def validate_mode_dependencies(args):
    if args.router_mode != "kv":
        kv_only_args = [
            ("kv_cache_block_size", "--kv-cache-block-size"),
            ("kv_overlap_score_weight", "--kv-overlap-score-weight"),
            ("router_replica_sync", "--router-replica-sync"),
            ("router_snapshot_threshold", "--router-snapshot-threshold"),
            ("router_reset_states", "--router-reset-states"),
        ]
        for attr, name in kv_only_args:
            val = getattr(args, attr, None)
            if val is not None and val != get_default(attr):
                raise ValueError(f"{name} only valid with --router-mode=kv")

    # gRPC port requires gRPC server
    if args.grpc_metrics_port != 8788 and not args.kserve_grpc_server:
        raise ValueError("--grpc-metrics-port requires --kserve-grpc-server")
```

### 3. Environment Variable Coverage

Add missing environment variables:

| Argument | Add Env Var |
|----------|-------------|
| `--tls-cert-path` | `DYN_TLS_CERT_PATH` |
| `--tls-key-path` | `DYN_TLS_KEY_PATH` |
| `--router-replica-sync` | `DYN_ROUTER_REPLICA_SYNC` |
| `--router-snapshot-threshold` | `DYN_ROUTER_SNAPSHOT_THRESHOLD` |
| `--router-reset-states` | `DYN_ROUTER_RESET_STATES` |
| `--track-active-blocks` | `DYN_ROUTER_TRACK_ACTIVE_BLOCKS` |
| `--active-decode-blocks-threshold` | `DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD` |
| `--active-prefill-tokens-threshold` | `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD` |
| `--kserve-grpc-server` | `DYN_KSERVE_GRPC_SERVER` |
| `--grpc-metrics-port` | `DYN_GRPC_METRICS_PORT` |

### 4. Boolean Argument Standardization

Convert all boolean arguments to `BooleanOptionalAction`:

```python
# Before (inconsistent)
parser.add_argument("--router-replica-sync", action="store_true")

# After (consistent)
parser.add_argument("--router-replica-sync/--no-router-replica-sync",
                    action=argparse.BooleanOptionalAction,
                    default=False)
```

This enables both `--flag` and `--no-flag` forms for all booleans.

### 5. Argument Grouping

Organize arguments into logical groups:

```python
# Server Configuration
server_group = parser.add_argument_group("HTTP Server")
server_group.add_argument("--http-host", ...)
server_group.add_argument("--http-port", ...)
server_group.add_argument("--tls-cert-path", ...)
server_group.add_argument("--tls-key-path", ...)

# Model Configuration
model_group = parser.add_argument_group("Model Configuration")
model_group.add_argument("--model-name", ...)
model_group.add_argument("--model-path", ...)
model_group.add_argument("--namespace", ...)

# Router Configuration
router_group = parser.add_argument_group("Request Routing")
router_group.add_argument("--router-mode", ...)

# KV Router (sub-group)
kv_group = parser.add_argument_group("KV Router Options (--router-mode=kv)")
kv_group.add_argument("--kv-cache-block-size", ...)
kv_group.add_argument("--kv-overlap-score-weight", ...)
kv_group.add_argument("--kv-events", ...)

# Router Tuning
tuning_group = parser.add_argument_group("Router Tuning")
tuning_group.add_argument("--router-temperature", ...)
# ... etc

# Metrics
metrics_group = parser.add_argument_group("Metrics & Observability")
metrics_group.add_argument("--metrics-prefix", ...)
# ... etc

# Backend
backend_group = parser.add_argument_group("Backend Configuration")
backend_group.add_argument("--store-kv", ...)
backend_group.add_argument("--request-plane", ...)
```

---

## Proposed Help Output

```
usage: python -m dynamo.frontend [OPTIONS]

Dynamo Frontend - OpenAI-compatible LLM inference server

HTTP Server:
  --http-host HOST        Listen address [env: DYN_HTTP_HOST] (default: 0.0.0.0)
  --http-port PORT        Listen port [env: DYN_HTTP_PORT] (default: 8000)
  --tls-cert-path FILE    TLS certificate [env: DYN_TLS_CERT_PATH]
  --tls-key-path FILE     TLS private key [env: DYN_TLS_KEY_PATH]
                          Note: Both --tls-cert-path and --tls-key-path required for TLS

Model Configuration:
  --model-name NAME       Model name for discovery (e.g., Llama-3.2-1B)
  --model-path PATH       Local model directory path
  --namespace NS          Namespace for model discovery [env: DYN_NAMESPACE]

Request Routing:
  --router-mode MODE      Routing strategy [env: DYN_ROUTER_MODE] (default: round-robin)
                          Choices: round-robin, random, kv

KV Router Options (requires --router-mode=kv):
  --kv-cache-block-size N      Block size for KV cache [env: DYN_KV_CACHE_BLOCK_SIZE]
  --kv-overlap-score-weight W  Cache reuse weight [env: DYN_KV_OVERLAP_SCORE_WEIGHT] (default: 1.0)
  --kv-events/--no-kv-events   Enable KV event tracking [env: DYN_KV_EVENTS] (default: enabled)

Router Tuning:
  --router-temperature T       Worker selection randomness (0-1) [env: DYN_ROUTER_TEMPERATURE] (default: 0.0)
  --router-ttl SECS            Block TTL when events disabled [env: DYN_ROUTER_TTL] (default: 120)
  --router-max-tree-size N     Max tree size [env: DYN_ROUTER_MAX_TREE_SIZE] (default: 1048576)
  --router-prune-target-ratio R  Prune target (0-1) [env: DYN_ROUTER_PRUNE_TARGET_RATIO] (default: 0.8)
  --router-replica-sync        Enable replica sync [env: DYN_ROUTER_REPLICA_SYNC] (default: disabled)
  --router-snapshot-threshold N  Snapshot interval [env: DYN_ROUTER_SNAPSHOT_THRESHOLD] (default: 1000000)
  --router-reset-states        Reset state on startup [env: DYN_ROUTER_RESET_STATES] (default: disabled)
  --track-active-blocks        Track active blocks [env: DYN_ROUTER_TRACK_ACTIVE_BLOCKS] (default: enabled)

Worker Load Detection:
  --active-decode-blocks-threshold P   Busy threshold (0-1) [env: DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD]
  --active-prefill-tokens-threshold N  Token threshold [env: DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD]

Metrics & Observability:
  --metrics-prefix PREFIX      Metrics name prefix [env: DYN_METRICS_PREFIX]
  --kserve-grpc-server         Enable gRPC metrics server [env: DYN_KSERVE_GRPC_SERVER]
  --grpc-metrics-port PORT     gRPC metrics port [env: DYN_GRPC_METRICS_PORT] (default: 8788)
  --custom-backend-metrics-endpoint EP    Backend metrics endpoint [env: CUSTOM_BACKEND_ENDPOINT]
  --custom-backend-metrics-polling-interval S  Poll interval [env: CUSTOM_BACKEND_METRICS_POLLING_INTERVAL]

Backend Configuration:
  --store-kv BACKEND      KV store backend [env: DYN_STORE_KV] (default: etcd)
                          Choices: etcd, file, mem
  --request-plane PLANE   Request transport [env: DYN_REQUEST_PLANE] (default: tcp)
                          Choices: tcp, http, nats

Experimental:
  --exp-python-factory    Use Python engine factory [EXPERIMENTAL]

General:
  --interactive, -i       Interactive mode
  --version               Show version
  --help                  Show this help
```

---

## Implementation Checklist

### Phase 1: Validation (Critical)

- [ ] Add port range validation (1-65535)
- [ ] Add float bounds validation (0-1, ≥0)
- [ ] Add positive number validation (>0)
- [ ] Add mode dependency validation
- [ ] Add TLS mutual requirement validation

### Phase 2: Environment Variables

- [ ] Add `DYN_TLS_CERT_PATH`, `DYN_TLS_KEY_PATH`
- [ ] Add `DYN_ROUTER_REPLICA_SYNC`, `DYN_ROUTER_SNAPSHOT_THRESHOLD`
- [ ] Add `DYN_ROUTER_RESET_STATES`, `DYN_ROUTER_TRACK_ACTIVE_BLOCKS`
- [ ] Add `DYN_ACTIVE_DECODE_BLOCKS_THRESHOLD`, `DYN_ACTIVE_PREFILL_TOKENS_THRESHOLD`
- [ ] Add `DYN_KSERVE_GRPC_SERVER`, `DYN_GRPC_METRICS_PORT`

### Phase 3: Standardization

- [ ] Convert all booleans to `BooleanOptionalAction`
- [ ] Add argument groups
- [ ] Improve help text (max 100 chars, include env var)
- [ ] Add validation error URLs to documentation

### Phase 4: Documentation

- [ ] Generate CLI reference from argparse
- [ ] Document all environment variables
- [ ] Create argument dependency matrix
- [ ] Add stability markers (stable/experimental)

---

## Alternate Solutions

### Option A: Configuration File Support

Add `--config-file` for complex deployments. **Deferred** to post-1.0:
- Adds complexity to argument precedence
- Current env var support sufficient for K8s
- Can add in 1.1 without breaking changes

### Option B: Strict Mode

Add `--strict` to reject any invalid combinations. **Rejected**:
- Should always validate by default
- Lenient mode not needed for production

---

## Related Proposals

- DEP-0000: Kubernetes API Formalization (related: env vars must match)
- DEP-0000: Dynamo API and Non-Feature Requirements (parent)

---

## References

- [Python argparse documentation](https://docs.python.org/3/library/argparse.html)
- [12-Factor App: Config](https://12factor.net/config)
- [vLLM CLI patterns](https://docs.vllm.ai/)
