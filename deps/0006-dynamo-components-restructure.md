# Dynamo Components Directory Restructure

**Status**: Draft

**Authors**: @anants

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: @nnshah1, @grahamking

**Required Reviewers**: @nnshah1, @grahamking, @athreesh, @nealvaidya, @ishandhanani, @rmccorm4, @biswapanda, @alec-flowers ,@ai-dynamo/DevOps

**Review Date**: TBD

**Pull Request**: TBD

**Implementation PR / Tracking Issue**: TBD

# Summary

Restructure the `components` directory in Dynamo repository to use a single `src/dynamo/` package structure and consolidate all non-source files (deploy/, launch/, docs/, configs/, etc.) into a logical, maintainable organization. This change will simplify packaging, improve editable installs, enhance developer experience, and create a more maintainable codebase while preserving all existing functionality.

# Motivation

The current Dynamo repository structure has several organizational issues that impact developer experience, maintainability, and operational efficiency:

1. **Deep nesting complexity**: Each component follows a `components/*/src/dynamo/component_name/` pattern, creating unnecessary directory depth
2. **Fragmented packaging**: The `pyproject.toml` requires listing 7 separate package paths, making it complex to manage
3. **Poor editable installß experience**: Developers must navigate through multiple nested directories to find source code
4. **Scattered non-source files**: Deployment configs, launch scripts, documentation, and benchmarks are scattered across component directories
5. **Inconsistent organization**: Similar files (deploy/, launch/, docs/) are duplicated across components with no central organization
6. **Maintenance overhead**: Adding new components requires updating multiple configuration files and duplicating directory structures
7. **Poor discoverability**: Related files are hard to find because they're buried in component-specific directories
8. **Operational complexity**: DevOps teams must navigate multiple directories to find deployment configurations

## Goals

* Simplify the package structure to use a single `src/dynamo/` directory
* Enable easier editable installs with `pip install -e .`
* Reduce configuration complexity in `pyproject.toml`
* Keep all backend requirements defined in `pyproject.toml`
* Consolidate all non-source files into logical, centralized directories
* Create a more maintainable and navigable repository structure
* Maintain all existing functionality and API compatibility
* Preserve the current wheel packaging behavior
* Reduce duplication of deployment configs, scripts, and documentation

### Non Goals

* Changing the external API or import structure for end users
* Modifying the wheel packaging output or distribution
* Changing the component architecture or functionality

## Requirements

### REQ 1 Single Package Structure

The project **MUST** use a single `src/dynamo/` directory containing all component source code instead of the current fragmented structure.

### REQ 2 Simplified pyproject.toml

The `pyproject.toml` **MUST** reference only a single package path: `"src/dynamo"` instead of the current 7 separate package paths. 

All the python dependencies **MUST** be defined in optional install under `pyproject.toml`

### REQ 3 Preserved Import Structure

All existing import statements **MUST** continue to work without modification. The restructuring **MUST NOT** break any existing code that imports from dynamo components.

### REQ 4 Editable Install Support

The new structure **MUST** support `pip install -e .` for development workflows without requiring additional configuration.

### REQ 5 Backward Compatibility

The wheel packaging output **MUST** remain identical to the current structure to ensure no breaking changes for existing users.

# Proposal

Comprehensively restructure the entire Dynamo repository to use a single `src/dynamo/` package structure and consolidate all non-source files into a logical, maintainable organization. This includes source code, deployment configurations, launch scripts, documentation, benchmarks, and all other repository files.

### Key Changes Summary

#### Source Code Organization
- **Current**: Scattered across `components/*/src/dynamo/*`
- **Proposed**: Consolidated in `components/src/dynamo/*`

#### Configuration and Script files
- **Current**: Scattered in `components/*/deploy/`, `components/*/launch/`
- **Proposed**: Consolidated in `configs/deployments/`, `scripts/launch/`

#### Documentation
- **Current**: Mixed with source code in component directories
- **Proposed**: Organized in `docs/` by component type

#### Tests
- **Current**: `components/planner/test/` + existing `tests/`
- **Proposed**: All tests in `tests/` with `planner/unit/` subdirectory

## Current Structure

```
dynamo/
├── components/
│   ├── frontend/
│   │   ├── src/dynamo/frontend/
│   │   └── README.md
│   ├── planner/
│   │   ├── src/dynamo/planner/
│   │   ├── test/
│   │   └── README.md
│   ├── metrics/
│   │   ├── src/ (Rust code)
│   │   └── README.md
│   └── backends/
│       ├── vllm/
│       │   ├── src/dynamo/vllm/
│       │   ├── deploy/
│       │   ├── launch/
│       │   └── README.md
│       ├── sglang/
│       │   ├── src/dynamo/sglang/
│       │   ├── deploy/
│       │   ├── launch/
│       │   ├── docs/
│       │   ├── benchmarks/
│       │   ├── configs/
│       │   ├── slurm_jobs/
│       │   └── README.md
│       ├── trtllm/
│       │   ├── src/dynamo/trtllm/
│       │   ├── deploy/
│       │   ├── launch/
│       │   ├── engine_configs/
│       │   ├── multinode/
│       │   ├── performance_sweeps/
│       │   └── README.md
│       ├── llama_cpp/
│       │   ├── src/dynamo/llama_cpp/
│       │   └── README.md
│       └── mocker/
│           ├── src/dynamo/mocker/
│           └── README.md
├── pyproject.toml
└── ... (other root files)
```

## Proposed Structure

```
dynamo/
├── components/
│   ├── src/                      # All Python source code
│   │   └── dynamo/
│   │       ├── __init__.py
│   │       ├── frontend/
│   │       │   ├── __init__.py
│   │       │   ├── __main__.py
│   │       │   ├── main.py
│   │       │   └── README.md
│   │       ├── planner/
│   │       │   ├── __init__.py
│   │       │   ├── ..
│   │       │   ├── README.md
│   │       ├── vllm/             # vLLM backend
│   │       │   ├── __init__.py
│   │       │   ├── ...
│   │       │   └── README.md
│   │       ├── sglang/           # SGLang backend
│   │       │   ├── __init__.py
│   │       │   ├── ..
│   │       │   ├── README.md
│   │       ├── trtllm/           # TensorRT-LLM backend
│   │       │   ├── __init__.py
│   │       │   ├── ..
│   │       │   ├── README.md
│   │       ├── llama_cpp/        # llama.cpp backend
│   │       │   ├── __init__.py
│   │       │   ├── ..
│   │       │   └── README.md
│   │       └── mocker/           # Mock backend
│   │           ├── __init__.py
│   │           ├── ..
│   │           └── README.md
│   └── metrics/                  # Metrics component (Rust)
│       ├── Cargo.toml
│       ├── src/
│       └── images/
├── configs/                      # All configuration files
│   ├── engines/                  # Engine-specific configs
│   │   ├── trtllm/
│   │   ├── sglang/
│   │   └── vllm/
│   └── deployments/              # Deployment configs
│       ├── vllm/
│       ├── sglang/
│       └── trtllm/
├── scripts/                      # All launch and utility scripts
│   ├── launch/                   # Launch scripts
│   │   ├── vllm/
│   │   ├── sglang/
│   │   └── trtllm/
│   ├── slurm/                    # SLURM job scripts
│   │   ├── sglang/
│   │   └── trtllm/
│   └── utils/                    # Utility scripts
│       ├── clear_namespace.py
│       └── gen_env_vars.sh
├── docs/                         # Component-specific documentation
│   ├── backends/
│   │   ├── vllm/
│   │   │   ├── deepseek-r1.md
│   │   │   ├── LMCache_Integration.md
│   │   │   └── multi-node.md
│   │   ├── sglang/
│   │   │   ├── dsr1-wideep-gb200.md
│   │   │   ├── dsr1-wideep-h100.md
│   │   │   ├── expert-distribution-eplb.md
│   │   │   ├── multinode-examples.md
│   │   │   └── sgl-hicache-example.md
│   │   └── trtllm/
│   │       ├── gemma3_sliding_window_attention.md
│   │       ├── gpt-oss.md
│   │       ├── kv-cache-transfer.md
│   │       ├── llama4_plus_eagle.md
│   │       ├── multimodal_epd.md
│   │       ├── multimodal_support.md
│   │       └── multinode-examples.md
├── benchmarks/                   # Performance benchmarks
│   ├── sglang/
│   │   ├── bench.sh
│   │   └── generate_bench_data.py
│   └── trtllm/
│       ├── benchmark_agg.slurm
│       ├── benchmark_disagg.slurm
│       ├── plot_performance_comparison.py
│       ├── post_process.py
│       └── scripts/
├── tests/     
│   ├── planner/unit/             # Move from components/planner/test
├── pyproject.toml
└── ... (other root files)
```

# Implementation Details

## Migration Steps

1. **Create new structure**: Create all new directories (`components/src/`, `configs/`, `scripts/`, `docs/`, `benchmarks/`)
2. **Move source code**: Copy all Python source files from `components/*/src/dynamo/*` to `components/src/dynamo/`
3. **Keep metrics**: Preserve `components/metrics/` in its current location
4. **Consolidate configs**: Move all `deploy/` directories to `configs/deployments/` organized by deployment type
5. **Consolidate scripts**: Move all `launch/` directories to `scripts/launch/` organized by deployment type
6. **Consolidate docs**: Move component-specific documentation to `docs/` organized by component
7. **Consolidate benchmarks**: Move all benchmark files to `benchmarks/` organized by component
8. **Consolidate tests**: Move `components/planner/test/` content to `tests/planner/unit/` (main tests directory already exists)
9. **Update pyproject.toml**: Change packages configuration to use single `"components/src/dynamo"` path
10. **Update build system**: Ensure hatch build system works with new structure
11. **Test packaging**: Verify wheel output remains identical
12. **Update documentation**: Update all references to old structure
13. **Update CI/CD**: Update any CI/CD scripts that reference old paths

## Package Structure Details

The new structure will maintain the same import paths:
- `from dynamo.frontend import ...` (unchanged)
- `from dynamo.planner import ...` (unchanged)  
- `from dynamo.vllm import ...` (unchanged)
- `from dynamo.sglang import ...` (unchanged)
- `from dynamo.trtllm import ...` (unchanged)
- `from dynamo.llama_cpp import ...` (unchanged)
- `from dynamo.mocker import ...` (unchanged)

## Build System Changes

The `pyproject.toml` will be simplified from:
```toml
packages = [
    "components/frontend/src/dynamo",
    "components/planner/src/dynamo",
    "components/backends/llama_cpp/src/dynamo",
    "components/backends/mocker/src/dynamo", 
    "components/backends/trtllm/src/dynamo",
    "components/backends/sglang/src/dynamo",
    "components/backends/vllm/src/dynamo"
]
```

To:
```toml
packages = ["components/src/dynamo"]
```

## Benefits of Comprehensive Restructuring

### Developer Experience
- **Single source directory**: All Python code in `components/src/dynamo/` for easy navigation
- **Preserved metrics**: Rust metrics component stays in `components/metrics/` for consistency
- **Centralized configs**: All deployment configurations in `configs/` for easy discovery
- **Organized scripts**: All launch scripts in `scripts/` organized by deployment type
- **Clear separation**: Source code, configs, scripts, docs, and tests are clearly separated

### Operational Efficiency
- **Faster deployment setup**: DevOps teams can find all deployment configs in one place
- **Easier maintenance**: Similar files are grouped together instead of scattered
- **Better discoverability**: New team members can understand the structure at a glance
- **Reduced duplication**: Common patterns are consolidated instead of repeated

### Maintainability
- **Simplified packaging**: Single package path in `pyproject.toml`
- **Consistent structure**: All components follow the same organization pattern
- **Better documentation**: Component docs organized in `docs/`
- **Cleaner repository**: No more deep nesting or mixed concerns

## Deferred to Implementation

* Specific migration implementation details
* CI/CD pipeline updates for new structure
* Documentation updates for development setup
* IDE configuration updates for new structure
* Path updates in all configuration files and scripts

# Implementation Phases
TBD

# Related Proposals

* N/A

# Alternate Solutions

## Alt 1: Keep Current Structure

**Pros:**
* No migration effort required
* No risk of breaking changes
* Familiar to current developers

**Cons:**
* Maintains current complexity and poor developer experience
* Difficult editable installs
* Complex pyproject.toml configuration
* Hard to add new components

**Reason Rejected:**
* Does not address the core problems of developer experience and maintainability
* Misses opportunity to improve the project structure

## Alt 2: Flatten to Root Level

**Pros:**
* Even simpler structure
* No src/ directory needed

**Cons:**
* Mixes source code with project root files
* Not following Python packaging best practices
* Could cause conflicts with project files

**Reason Rejected:**
* Violates Python packaging conventions
* Creates potential file conflicts
* Not recommended by Python packaging guidelines

## Alt 3: Separate Packages

**Pros:**
* Each component is independently versioned
* Clear separation of concerns

**Cons:**
* Much more complex packaging and dependency management
* Difficult to maintain consistent versions
* Overkill for tightly coupled components
* Breaks the unified dynamo package concept

**Reason Rejected:**
* Adds unnecessary complexity for a tightly integrated system
* Would require major changes to import structure

