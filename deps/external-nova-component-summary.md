# Nova Component Summary (nvidia-lpu/Groq)

## Overview

Nova is a conductor service responsible for spawning agents and assigning relevant work for them on Inference Engines (IEs). It manages the lifecycle of inference workers on Groq's LPU (Language Processing Unit) hardware, handling model setup, agent coordination, and configuration distribution.

## Location

- **Repository**: `https://github.com/nvidia-lpu/nova` (private)
- **Primary Source**: `src/` (Rust)
- **Python Components**: `mock_agent_python/`
- **Configuration**: `configs/`
- **Cap'n Proto Schemas**: `capnp/`

## Internal Dependencies

### Languages & LOC

| Language | Lines of Code | Purpose |
|----------|---------------|---------|
| Python | 2,958,998 | Agent implementations, testing |
| Rust | 864,530 | Core conductor logic |
| Go | 29,417 | Auxiliary tooling |
| Cap'n Proto | 27,945 | IE communication protocol |
| Shell | 18,640 | Build/deploy scripts |
| HCL | 3,006 | Terraform infrastructure |

### Key Dependencies

| Category | Dependencies |
|----------|--------------|
| **Async Runtime** | tokio |
| **Protocol** | Cap'n Proto |
| **Serialization** | serde, toml |
| **Build** | protoc, capnp compiler |

## Module Structure

### Key Directories

| Directory | Purpose |
|-----------|---------|
| `src/` | Rust conductor source code |
| `mock_agent_python/` | Python mock agent implementations |
| `configs/` | Model and engine configurations |
| `capnp/` | Cap'n Proto schema definitions |
| `tests/` | Test configurations and fixtures |
| `terraform/` | Infrastructure as code |
| `benches/` | Performance benchmarks |

### Configuration Files

| File | Purpose |
|------|---------|
| `app_config.toml` | Application configuration |
| `datacenter.toml` | Datacenter topology |
| `*_model_config.toml` | Model-specific configurations |
| `engine-config.toml` | Inference engine settings |

## Public Interface

### CLI Interface

```bash
nova \
  -c ./app_config.toml \
  -d ./datacenter.toml \
  -m ./model_config.toml \
  --allocation c1r1 \
  --instance-model-name <name> \
  [--mock-agents]
```

### Key CLI Arguments

| Argument | Description |
|----------|-------------|
| `-c, --config-filepath` | Application configuration file |
| `-d` | Datacenter configuration |
| `-m, --model-config-filepath` | Model configuration |
| `--allocation` | Node allocation (e.g., `c1r1:c2r2,c2r3`) |
| `--instance-model-name` | Model instance identifier |
| `--mock-agents` | Use mock agents for testing |

### Supported Configurations

- Single model deployments
- Speculative decoding (`spec_decode_model_config.toml`)
- Vision models (`vision_model_config.toml`)
- Static LoRA adapters (`single_model_static_lora_config.toml`)
- Dynamic LoRA adapters (`single_model_dyn_lora_config.toml`)
- GRun configurations (`grun_config.toml`)

## User/Developer Interaction

### 1. Local Development

```bash
# Install Rust toolchain
source ./.buildkite/scripts/install-rust-toolchain.sh

# Activate Hermit environment
source ./bin/activate-hermit

# Build
cargo build

# Run with mock agents
./target/debug/nova \
  -c ./tests/app_config.toml \
  -d ./tests/datacenter.toml \
  -m ./tests/single_model_config.toml \
  --allocation c1r1 \
  --instance-model-name test \
  --mock-agents
```

### 2. Integration with Neutrino

Nova works in conjunction with Neutrino (inference proxy):

```bash
# Terminal 1: Run Nova
./target/debug/nova -m ./tests/single_model_config.toml --mock-agents

# Terminal 2: Run Neutrino
cd ~/workspace/neutrino
export NEUTRINO_CFG=./neutrino_cli_config.txt
cargo run --bin neutrino

# Terminal 3: Test client
./target/debug/test_client -n 1 -c 1 --show-output --stream-response
```

### 3. Vision Model Example

```bash
# Start vision agents
cargo build && ./target/debug/nova \
  -m ./tests/vision_model_config.toml \
  --allocation c1r1:c2r2 \
  --instance-model-name test2 \
  --mock-agents

# Test with image
curl -X POST "http://localhost:51055/openai/v1/chat/completions" \
  -H "Content-Type: application/json" \
  -d '{"model": "llama3-70b-8192", "messages": [{"role": "user", "content": [{"type": "text", "text": "What is in this image?"}, {"type": "image_url", "image_url": {"url": "http://example.com/image.png"}}]}]}'
```

## Packaging & Containers

### Build Process

```bash
# Debug build
cargo build

# Release build
cargo build --release
```

### Deployment

- **CI/CD**: Buildkite pipeline
- **Release**: `./deploy_release.sh <release tag>`
- **Production Config**: `/etc/groq-nova.config`

### Container

- Dockerfile available in repository
- Multi-stage build
- Requires protoc and Cap'n Proto compiler

## Service Interface & I/O Contract

### Inputs

| Input | Source | Format |
|-------|--------|--------|
| Application config | File | TOML |
| Model config | File | TOML |
| Datacenter config | File | TOML |
| Allocation spec | CLI | String pattern |

### Outputs

| Output | Destination | Format |
|--------|-------------|--------|
| Agent assignments | Inference Engines | Cap'n Proto |
| Setup operations | IEs | Cap'n Proto |
| Logs | stdout/file | Structured |

### Communication Protocol

- **To IEs**: Cap'n Proto over UNIX sockets
- **With Neutrino**: Coordinated via shared IE connections

## Observability

### Logging

- Structured logging via Rust tracing
- Configurable log levels
- Integration with Buildkite for CI logs

### Metrics

- Performance benchmarks in `benches/`
- Integration metrics via Neutrino

## 1.0 Standardization Checklist

| Area | Current State | Notes |
|------|---------------|-------|
| Configuration | TOML files | Well-structured, multiple config types |
| CLI interface | argparse-style | Documented arguments |
| Protocol | Cap'n Proto | Versioned schemas |
| Error handling | Rust Result types | Strong typing |
| Documentation | README + Notion | External docs in Notion |

## Customization & Extension

### Extension Points

1. **Model Configurations** - Add new model TOML files
2. **Mock Agents** - Python mock implementations
3. **Allocation Patterns** - Custom node allocation strings
4. **LoRA Adapters** - Static and dynamic adapter support

### Configuration Options

| Option | Description |
|--------|-------------|
| Model type | Chat, vision, speculative decode |
| Allocation | Node/rack allocation pattern |
| LoRA mode | Static, dynamic, none |
| Agent type | Real or mock |

## Dynamo Equivalent

**Closest Match**: **Planner + Backend Orchestration**

| Aspect | Nova | Dynamo |
|--------|------|--------|
| **Purpose** | Spawn agents, assign IE work | Scale workers, coordinate backends |
| **Deployment** | Conductor service | Planner component |
| **Resource Management** | Node/rack allocation | K8s replica scaling |
| **Configuration** | TOML files | CLI args + K8s CRDs |

**Key Differences**:
- Nova is tightly coupled to Groq's LPU hardware
- Dynamo's Planner is K8s-native with CRD-based scaling
- Nova handles lower-level agent lifecycle; Dynamo abstracts this to backends

## Related Documentation

- [Neutrino Documentation](https://www.notion.so/groq/Neutrino-Developer-Documentation)
- Buildkite Pipeline: `https://buildkite.com/groq/nova`

## Tests

- **Unit Tests**: `tests/` directory
- **Mock Agents**: `mock_agent_python/`
- **Config Fixtures**: Various `*_config.toml` files
- **Integration**: Via Neutrino test client
