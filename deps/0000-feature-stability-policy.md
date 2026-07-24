# DEP: Feature Stability Policy

**Status**: Draft

**Authors**: @nnshah1

**Category**: Process

**Sponsor**: TBD

**Required Reviewers**: TBD

**Review Date**: TBD

**Related PRs**: N/A

---

## Summary

This DEP establishes a formal feature stability policy for Dynamo, defining how features are classified as Experimental, Beta, or Stable. It specifies marking mechanisms for each component type (Python, Rust, CLI, Kubernetes CRDs, environment variables), graduation criteria, and deprecation timelines. The policy is based on best practices from PyTorch, Kubernetes, Rust, Google APIs, and Python PEP 702.

---

## Motivation

As Dynamo approaches 1.0 GA, users and integrators need clear signals about which features are production-ready versus still evolving. Currently:

1. **No formal stability markers** - Users cannot distinguish experimental features from stable ones
2. **Inconsistent deprecation** - Some features are removed without notice; others linger indefinitely
3. **Unclear API contracts** - CLI arguments, environment variables, and Python APIs lack stability guarantees
4. **Enterprise adoption blocked** - Organizations require stability guarantees before production deployment

### Current State

| Component | Stability Marking | Deprecation Policy |
|-----------|-------------------|-------------------|
| Python APIs | None | Ad-hoc |
| Rust APIs | Some `#[deprecated]` | Inconsistent |
| CLI arguments | None | None |
| K8s CRDs | `v1alpha1` version | Follows K8s conventions |
| Environment variables | None | None |

---

## Goals

- Define three stability levels with clear semantics: Experimental, Beta, Stable
- Establish marking mechanisms appropriate for each component type
- Create graduation criteria for feature promotion
- Define deprecation timelines and removal policies
- Enable tooling to extract and validate stability metadata

### Non Goals

- Retroactively classifying all existing features (separate effort)
- Implementing stability-aware documentation generation
- Creating stability dashboards or tracking systems

---

## Requirements

### REQ 1: Stability Levels

All public APIs **MUST** be classified into one of three stability levels:

| Level | Symbol | Commitment | Breaking Changes |
|-------|--------|------------|------------------|
| **Experimental** | `experimental` | May change or be removed without notice | Allowed any time |
| **Beta** | `beta` | Will graduate to Stable; changes with deprecation notice | 1 release notice |
| **Stable** | `stable` | Production-ready; BC guaranteed within major version | 2 release notice |

### REQ 2: Marking Mechanisms

Each component type **MUST** have a defined marking mechanism:

- Python: Decorators (`@experimental`, `@beta`, `@deprecated`)
- Rust: Feature flags and `#[deprecated]` attributes
- CLI: Help text suffixes (`[EXPERIMENTAL]`, `[BETA]`)
- K8s CRDs: API version suffixes (`v1alpha1`, `v1beta1`, `v1`)
- Environment variables: Naming prefixes (`DYN_EXPERIMENTAL_*`)

### REQ 3: Graduation Criteria

Features **MUST** meet defined criteria before promotion between stability levels.

### REQ 4: Deprecation Timeline

Deprecated features **MUST** follow minimum notice periods before removal:
- Experimental: No notice required
- Beta: 1 release (minimum 2 months)
- Stable: 2 releases (minimum 4 months)

---

## Industry Research

### PyTorch: Three-Tier Classification

PyTorch uses a three-tier system that evolved from a confusing "Experimental" label to clearer terminology:

| Level | Definition | BC Guarantee | Examples |
|-------|------------|--------------|----------|
| **Stable** | Proven value, stable API, performant, documented | Yes (1 release notice) | `torch.nn`, `torch.optim` |
| **Beta** | Value proven, API may change based on feedback | No | `torch.compile`, `torch.export` |
| **Prototype** | Early-stage, gathering feedback, may be dropped | No | Named tensors (early) |

**Key Insight**: PyTorch renamed "Experimental" to "Beta" because users incorrectly assumed experimental features were unstable prototypes rather than nearly-complete features needing polish.

**Dynamo Application**: The `--enable-chunked-prefill` and `--enable-prefix-caching` CLI flags represent Beta features - they work but performance characteristics may change.

Source: [PyTorch Feature Classification Changes](https://pytorch.org/blog/pytorch-feature-classification-changes/)

---

### Kubernetes: Version-Based API Stability

Kubernetes embeds stability in API versions with strict graduation and deprecation rules:

| Level | Version | Lifetime | Deprecation |
|-------|---------|----------|-------------|
| **Alpha** | `v1alpha1` | 1 release | May remove without notice |
| **Beta** | `v1beta1` | 9 months or 3 releases | Required before removal |
| **GA** | `v1` | Major version | Required, long notice |

**Key Rules**:
1. Beta starts a "countdown" - 9 months to graduate or deprecate
2. GA APIs can only be deprecated when a newer GA version exists
3. Deprecated functionality **must not** graduate (no "pre-deprecated" features)

**Dynamo Application**: Dynamo's CRDs (`DynamoGraphDeployment`, `DynamoComponentDeployment`) are currently `v1alpha1`. Before 1.0 GA:
- Lock API fields and graduate to `v1beta1`
- Provide 9-month window before `v1` GA

Source: [Kubernetes Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)

---

### Rust: Compiler-Enforced Stability

Rust uses compiler-enforced attributes that prevent accidental use of unstable features:

```rust
// Unstable - requires #![feature(foo)] on nightly
#[unstable(feature = "foo", issue = "1234")]
pub fn experimental_function() { ... }

// Stable - available on all channels
#[stable(feature = "bar", since = "1.50.0")]
pub fn stable_function() { ... }

// Deprecated - emits warning at compile time
#[deprecated(since = "1.60.0", note = "Use new_function instead")]
pub fn old_function() { ... }
```

**Key Insight**: Rust's `#[unstable]` attribute blocks usage at compile time unless explicitly opted-in via feature flags. This prevents accidental dependencies on unstable APIs.

**Dynamo Application**: For Rust crates like `dynamo-runtime` and `dynamo-llm`:
- Use `#[cfg(feature = "experimental_kv_disagg")]` to gate experimental code
- Use `#[deprecated]` with `since` and `note` for removal warnings
- Document stability in rustdoc comments for Beta features

Source: [Rust Stability Attributes](https://rustc-dev-guide.rust-lang.org/stability.html)

---

### Google APIs: Channel-Based Versioning

Google uses channel-based versioning with strict superset rules:

| Channel | Version | Deprecation | Removal |
|---------|---------|-------------|---------|
| **Alpha** | `v1alpha` | Optional | Any time |
| **Beta** | `v1beta` | Required | 180 days minimum |
| **Stable** | `v1` | Required | Major version only |

**Key Rules**:
1. Beta **must be superset** of Stable functionality
2. Alpha **must be superset** of Beta functionality
3. Deprecated functionality **must not** graduate between channels

**Dynamo Application**: The OpenAI-compatible API endpoints should follow this pattern:
- `/v1/chat/completions` - Stable (OpenAI parity)
- `/v1beta/chat/completions` - Beta extensions (Dynamo-specific features)
- `/v1alpha/internal/*` - Alpha internal APIs

Source: [Google AIP-185: API Versioning](https://google.aip.dev/185)

---

### Python PEP 702: @deprecated Decorator

Python 3.13+ includes a standard `@deprecated` decorator with type checker integration:

```python
from warnings import deprecated

@deprecated("Use new_function() instead")
def old_function(): ...

# Type checkers (Pyright, mypy) emit warnings at call sites
# Runtime emits DeprecationWarning
```

**Key Features**:
- Static analysis support (IDE warnings before running code)
- Runtime warnings with configurable severity
- Available for older Python via `typing_extensions`

**Dynamo Application**: Create `dynamo.utils.stability` module with decorators that:
- Emit runtime warnings
- Add `__stability__` metadata for tooling
- Support static type checkers

Source: [PEP 702 – Marking deprecations using the type system](https://peps.python.org/pep-0702/)

---

### vLLM: Production-First Stability

vLLM's 2025 vision emphasizes production stability:

> "As LLMs become the backbone of modern applications, vLLM envisions powering thousands of production clusters running 24/7. These aren't experimental deployments—they're mission-critical systems."

**Key Approach**:
- Features like quantization, prefix caching, speculative decoding becoming **default** (not optional)
- Creating **stable interfaces** for cluster-level solutions
- Explicitly marking experimental features in documentation (e.g., "This feature is still experimental")

**Dynamo Application**: Features like KV cache disaggregation and multi-node inference should have explicit stability markers in both code and documentation.

Source: [vLLM 2024 Retrospective and 2025 Vision](https://blog.vllm.ai/2025/01/10/vllm-2024-wrapped-2025-vision.html)

---

## Proposal

### 1. Stability Level Definitions

#### Experimental
- **Definition**: Early-stage feature for evaluation and feedback
- **Commitment**: None - may be removed or changed without notice
- **Documentation**: Purpose and known limitations only
- **Marking**: Explicit `[EXPERIMENTAL]` labels, runtime warnings on first use

**Dynamo Examples**:
- KV cache disaggregation (`--enable-kv-disagg`)
- Multi-node inference with MPI
- Mooncake connector integration

#### Beta
- **Definition**: Feature-complete, API may change based on feedback
- **Commitment**: Will either graduate to Stable or be deprecated with notice
- **Documentation**: Full API docs, migration notes, known limitations
- **Marking**: `[BETA]` labels, no runtime warnings

**Dynamo Examples**:
- Chunked prefill (`--enable-chunked-prefill`)
- Prefix caching (`--enable-prefix-caching`)
- LoRA hot-loading
- DGDSA autoscaling adapter

#### Stable
- **Definition**: Production-ready with backwards compatibility guarantee
- **Commitment**: Breaking changes only with 2-release deprecation notice
- **Documentation**: Full docs, examples, tutorials, performance characteristics
- **Marking**: No special marking (absence of label implies Stable)

**Dynamo Examples**:
- OpenAI-compatible `/v1/chat/completions` endpoint
- Core CLI arguments (`--model`, `--http-port`, `--tensor-parallel-size`)
- `DynamoGraphDeployment` CRD (after graduation to `v1`)

---

### 2. Python Marking Implementation

Create `dynamo/utils/stability.py`:

```python
"""Feature stability decorators for Dynamo Python APIs."""

import functools
import warnings
from typing import Callable, Optional, TypeVar

F = TypeVar('F', bound=Callable)

# Track first-use warnings to avoid spam
_warned_experimental: set[str] = set()


def experimental(reason: str = "") -> Callable[[F], F]:
    """Mark a function/class as experimental.

    Experimental features may change or be removed without notice.
    Emits FutureWarning on first use.

    Example:
        @experimental("KV disaggregation API is under active development")
        def enable_kv_disaggregation():
            ...
    """
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            key = f"{func.__module__}.{func.__qualname__}"
            if key not in _warned_experimental:
                msg = f"{func.__qualname__} is EXPERIMENTAL and may change without notice."
                if reason:
                    msg += f" {reason}"
                warnings.warn(msg, category=FutureWarning, stacklevel=2)
                _warned_experimental.add(key)
            return func(*args, **kwargs)

        wrapper.__stability__ = "experimental"
        wrapper.__stability_reason__ = reason
        return wrapper
    return decorator


def beta(reason: str = "") -> Callable[[F], F]:
    """Mark a function/class as beta.

    Beta features are feature-complete but API may change with notice.
    No runtime warning - stability indicated in docs only.

    Example:
        @beta("API may change based on user feedback")
        def configure_prefix_caching():
            ...
    """
    def decorator(func: F) -> F:
        func.__stability__ = "beta"
        func.__stability_reason__ = reason
        return func
    return decorator


def deprecated(
    reason: str,
    removal: Optional[str] = None,
    alternative: Optional[str] = None
) -> Callable[[F], F]:
    """Mark a function/class as deprecated.

    Emits DeprecationWarning on every use.

    Args:
        reason: Why this is deprecated
        removal: Version when this will be removed (e.g., "1.0")
        alternative: What to use instead

    Example:
        @deprecated(
            "Direct etcd access is deprecated",
            removal="1.0",
            alternative="Use DistributedRuntime.kv_store() instead"
        )
        def get_etcd_client():
            ...
    """
    def decorator(func: F) -> F:
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            msg = f"{func.__qualname__} is DEPRECATED. {reason}"
            if alternative:
                msg += f" Use {alternative} instead."
            if removal:
                msg += f" Will be removed in version {removal}."
            warnings.warn(msg, category=DeprecationWarning, stacklevel=2)
            return func(*args, **kwargs)

        wrapper.__stability__ = "deprecated"
        wrapper.__deprecated_reason__ = reason
        wrapper.__removal_version__ = removal
        wrapper.__alternative__ = alternative
        return wrapper
    return decorator
```

**Usage in Dynamo Frontend**:

```python
from dynamo.utils.stability import experimental, beta, deprecated

@experimental("Multi-node inference is under active development")
def configure_multinode(node_count: int):
    """Configure multi-node inference with MPI."""
    ...

@beta("Prefix caching API may change based on performance feedback")
def enable_prefix_caching(max_cache_size: int):
    """Enable KV prefix caching for repeated prompts."""
    ...

@deprecated(
    "Static endpoint mode is deprecated",
    removal="1.0",
    alternative="Use dynamic service discovery"
)
def set_static_endpoint(endpoint: str):
    """Set a static backend endpoint."""
    ...
```

---

### 3. CLI Argument Marking

Update argparse help text with stability suffixes:

```python
# Before
parser.add_argument("--enable-kv-disagg", action="store_true",
                    help="Enable KV cache disaggregation")

# After
parser.add_argument("--enable-kv-disagg", action="store_true",
                    help="Enable KV cache disaggregation [EXPERIMENTAL]")

parser.add_argument("--enable-prefix-caching", action="store_true",
                    help="Enable prefix caching for repeated prompts [BETA]")

parser.add_argument("--model", type=str, required=True,
                    help="Model name or path")  # No suffix = Stable
```

**Rendered Help Output**:
```
Options:
  --model MODEL           Model name or path
  --enable-prefix-caching Enable prefix caching for repeated prompts [BETA]
  --enable-kv-disagg      Enable KV cache disaggregation [EXPERIMENTAL]
```

---

### 4. Rust Feature Gating

Use Cargo feature flags for experimental features:

```toml
# Cargo.toml
[features]
default = []
experimental_kv_disagg = []
experimental_multinode = []
```

```rust
// lib.rs
/// Enable KV cache disaggregation.
///
/// # Stability
///
/// **EXPERIMENTAL**: This API is under active development and may change
/// without notice. Enable with `--features experimental_kv_disagg`.
#[cfg(feature = "experimental_kv_disagg")]
pub mod kv_disagg {
    pub fn enable() { ... }
}

/// **DEPRECATED since 0.9.0**: Use `new_router()` instead.
#[deprecated(since = "0.9.0", note = "Use new_router() instead")]
pub fn old_router() -> Router { ... }
```

---

### 5. Environment Variable Naming

| Prefix | Stability | Example |
|--------|-----------|---------|
| `DYN_EXPERIMENTAL_*` | Experimental | `DYN_EXPERIMENTAL_KV_DISAGG=true` |
| `DYN_BETA_*` | Beta | `DYN_BETA_CHUNKED_PREFILL=true` |
| `DYN_*` | Stable | `DYN_NAMESPACE=default` |

**Migration Example**:
```bash
# v0.8 (Experimental)
DYN_EXPERIMENTAL_PREFIX_CACHING=true

# v0.9 (Beta) - old name deprecated, new name works
DYN_BETA_PREFIX_CACHING=true
# Warning: DYN_EXPERIMENTAL_PREFIX_CACHING is deprecated

# v1.0 (Stable) - promoted to stable name
DYN_PREFIX_CACHING=true
```

---

### 6. Kubernetes CRD Versioning

Follow Kubernetes conventions already in use:

| Stability | API Version | Example |
|-----------|-------------|---------|
| Experimental | `v1alpha1`, `v1alpha2` | Current state |
| Beta | `v1beta1` | After field stabilization |
| Stable | `v1` | After 9-month beta period |

**Current CRD Status**:

| CRD | Current | Target for 1.0 |
|-----|---------|----------------|
| `DynamoGraphDeployment` | `v1alpha1` | `v1beta1` or `v1` |
| `DynamoComponentDeployment` | `v1alpha1` | `v1beta1` or `v1` |
| `DynamoModel` | `v1alpha1` | `v1beta1` or `v1` |
| `DynamoGraphDeploymentRequest` | `v1alpha1` | `v1alpha1` (experimental) |
| `DynamoGraphDeploymentScalingAdapter` | `v1alpha1` | `v1beta1` |

---

### 7. Graduation Criteria

#### Experimental → Beta

- [ ] Core functionality works end-to-end in CI
- [ ] Basic documentation exists (purpose, usage, limitations)
- [ ] At least one production partner has tested
- [ ] No known critical bugs blocking usage
- [ ] Performance within 2x of expected target

#### Beta → Stable

- [ ] API frozen for 2+ releases (no breaking changes)
- [ ] Full documentation with examples and tutorials
- [ ] Performance meets published SLAs
- [ ] No breaking changes for 3+ releases
- [ ] Migration guide from deprecated APIs (if applicable)
- [ ] Feature enabled by default (for opt-in features)

---

### 8. Deprecation Policy

| Current Level | Notice Period | Removal Allowed | Notes |
|---------------|---------------|-----------------|-------|
| Experimental | None required | Any release | May disappear without warning |
| Beta | 1 release (2 months) | After notice period | Must document alternative |
| Stable | 2 releases (4 months) | After notice period | Requires migration guide |

**Deprecation Checklist**:
1. Add `@deprecated` decorator / `#[deprecated]` attribute
2. Document in CHANGELOG.md
3. Add runtime warning with alternative
4. Update documentation with migration path
5. Remove after notice period expires

---

## Implementation Phases

### Phase 1: Infrastructure

**Effort**: 1 engineer, 1 week

- [ ] Create `dynamo/utils/stability.py` module
- [ ] Define Rust feature flag conventions in contributing guide
- [ ] Update CLI argument parser helper for stability suffixes
- [ ] Document environment variable naming convention

### Phase 2: Audit and Classification

**Effort**: 2 engineers, 2 weeks

- [ ] Audit all CLI arguments and classify by stability
- [ ] Audit Python public APIs and classify
- [ ] Audit Rust public APIs and classify
- [ ] Create initial stability registry document

### Phase 3: Apply Markers

**Effort**: 2 engineers, 2 weeks

- [ ] Add decorators to Python functions/classes
- [ ] Add `#[deprecated]` to Rust APIs
- [ ] Update CLI help text with stability markers
- [ ] Add stability badges to documentation

### Phase 4: CI Integration

**Effort**: 1 engineer, 1 week

- [ ] CI check: new experimental features require marker
- [ ] CI check: deprecated features have removal version
- [ ] Auto-generate stability section in release notes

---

## Related Proposals

- [DEP-0000: Kubernetes API Formalization](0000-kubernetes-api-formalization.md)
- [DEP-0000: Frontend CLI Formalization](0000-frontend-cli-formalization.md)
- [DEP-0000: Backend-Frontend Contract](0000-backend-frontend-contract.md)

---

## Alternate Solutions

### Alt 1: Documentation-Only Marking

Mark stability only in documentation without code-level enforcement.

**Pros**:
- Simple to implement
- No runtime overhead

**Cons**:
- Easy to miss during development
- No tooling support
- Users may not read docs

**Reason Rejected**: Insufficient for enterprise users who need programmatic stability detection.

---

### Alt 2: Semantic Versioning Only

Rely solely on semver (major version = breaking changes).

**Pros**:
- Industry standard
- Simple to understand

**Cons**:
- Doesn't distinguish experimental from stable within a version
- All of 0.x is implicitly "unstable"
- Doesn't help users choose features

**Reason Rejected**: Semver applies to releases, not individual features. Users need feature-level stability signals.

---

### Alt 3: Feature Flags for Everything

Gate all non-stable features behind explicit opt-in flags.

**Pros**:
- Maximum protection for users
- Clear opt-in semantics

**Cons**:
- Friction for trying new features
- Complex flag management
- Rust feature flags have compile-time overhead

**Reason Rejected**: Too heavy-handed for Beta features that are nearly production-ready. Reserve for truly experimental features only.

---

## Background

### References

- [PyTorch Feature Classification Changes](https://pytorch.org/blog/pytorch-feature-classification-changes/) - Three-tier system
- [ExecuTorch API Life Cycle](https://docs.pytorch.org/executorch/stable/api-life-cycle.html) - Deprecation policy
- [Kubernetes Deprecation Policy](https://kubernetes.io/docs/reference/using-api/deprecation-policy/) - Version-based stability
- [Rust Stability Attributes](https://rustc-dev-guide.rust-lang.org/stability.html) - Compiler-enforced stability
- [Google AIP-185: API Versioning](https://google.aip.dev/185) - Channel-based versioning
- [PEP 702: @deprecated Decorator](https://peps.python.org/pep-0702/) - Python deprecation standard
- [vLLM 2024-2025 Vision](https://blog.vllm.ai/2025/01/10/vllm-2024-wrapped-2025-vision.html) - Production stability focus

### Terminology & Definitions

| Term | Definition |
|------|------------|
| **Backwards Compatibility (BC)** | Existing code continues to work without modification after an update |
| **Breaking Change** | A change that requires users to modify their code |
| **Deprecation** | Marking a feature as obsolete with intent to remove |
| **Feature Gate** | Compile-time or runtime flag to enable/disable functionality |
| **Graduation** | Promotion of a feature from one stability level to the next |

### Acronyms

- **BC**: Backwards Compatibility
- **GA**: General Availability (stable release)
- **API**: Application Programming Interface
- **CRD**: Custom Resource Definition (Kubernetes)
- **PEP**: Python Enhancement Proposal
