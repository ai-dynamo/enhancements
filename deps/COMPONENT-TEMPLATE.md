# {Component Name} - 1.0 Component Specification

> **Template Version**: 1.0
> **Last Updated**: 2026-02-05
> **Instructions**: Replace all `{placeholders}` and `...` with component-specific information. Delete sections that don't apply (e.g., Public API for non-Library components).

---

## Overview

Brief description of the component's purpose and role in Dynamo.

**Support Level**: {Service/Command | Library | Schemas | Experimental | Internal}

**PIC**: {Name} ({email})

---

## Quick Start

### Installation

```bash
# If standalone binary
pip install dynamo-{component}

# If part of dynamo package
pip install dynamo
```

### Basic Usage

```bash
# Command line example
dynamo-{component} --help

# Or programmatic example
from dynamo import {Component}
```

### Minimal Working Example

```python
# A complete, copy-paste-able example that demonstrates the core functionality
...
```

---

## Design

### Architecture

```
                    ┌─────────────────┐
                    │   {Component}   │
                    └────────┬────────┘
                             │
            ┌────────────────┼────────────────┐
            ▼                ▼                ▼
    ┌───────────┐    ┌───────────┐    ┌───────────┐
    │  Input A  │    │  Input B  │    │  Output   │
    └───────────┘    └───────────┘    └───────────┘
```

*Replace with actual architecture diagram*

### Role in Dynamo Ecosystem

Describe how this component fits into the overall Dynamo architecture and what problem it solves.

### Relationship to Other Components

| Component | Relationship | Interface |
|-----------|-------------|-----------|
| Frontend | {Consumes from / Provides to / Coordinates with} | {NATS / gRPC / REST / Direct call} |
| Router | ... | ... |
| Backend | ... | ... |
| ... | ... | ... |

### Data Flow

**Input Contract**:
- Format: {JSON / Protobuf / Binary / ...}
- Schema: Link or inline definition
- Example:
```json
{
  "example": "input"
}
```

**Output Contract**:
- Format: {JSON / Protobuf / Binary / ...}
- Schema: Link or inline definition
- Example:
```json
{
  "example": "output"
}
```

---

## Configuration

### Command Line Flags

| Flag | Type | Default | Env Var | Description |
|------|------|---------|---------|-------------|
| `--host` | string | `0.0.0.0` | `DYNAMO_{COMPONENT}_HOST` | Host address to bind |
| `--port` | int | `8000` | `DYNAMO_{COMPONENT}_PORT` | Port to listen on |
| `--log-level` | string | `info` | `DYNAMO_LOG_LEVEL` | Logging level (debug, info, warn, error) |
| ... | ... | ... | ... | ... |

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `DYNAMO_{COMPONENT}_*` | - | - | Component-specific variables |
| `DYNAMO_LOG_LEVEL` | string | `info` | Global log level |
| `DYNAMO_NATS_URL` | string | `nats://localhost:4222` | NATS connection URL |
| `DYNAMO_ETCD_URL` | string | `http://localhost:2379` | etcd connection URL |
| ... | ... | ... | ... |

### Configuration Precedence

1. Command line flags (highest priority)
2. Environment variables
3. Configuration file (if supported)
4. Default values (lowest priority)

### Configuration Files (if applicable)

**Schema**: `{component}-config.yaml`

```yaml
# Example configuration file
{component}:
  host: "0.0.0.0"
  port: 8000
  options:
    key: value
```

---

## Public API (for Library components)

### Stable API (1.0)

These APIs have backwards compatibility guarantees and follow semantic versioning.

| Class/Function | Purpose | Module |
|----------------|---------|--------|
| `{ComponentClass}` | Main entry point | `dynamo.{component}` |
| `{function_name}()` | Description | `dynamo.{component}` |
| ... | ... | ... |

**Example Usage**:
```python
from dynamo.{component} import {ComponentClass}

# Initialize
component = {ComponentClass}(config)

# Use
result = component.process(input_data)
```

### Experimental API

These APIs may change without notice. Import from `dynamo.experimental.{component}`.

| Class/Function | Purpose | Module |
|----------------|---------|--------|
| ... | ... | `dynamo.experimental.{component}` |

### Internal API (not for external use)

Internal APIs are in `dynamo._internal.{component}` and have no stability guarantees.

---

## Metrics & Observability

### Metrics

All metrics use the prefix `dynamo_{component}_`.

| Metric Name | Type | Labels | Description |
|-------------|------|--------|-------------|
| `dynamo_{component}_requests_total` | Counter | `status`, `method` | Total requests processed |
| `dynamo_{component}_request_duration_seconds` | Histogram | `method` | Request latency |
| `dynamo_{component}_errors_total` | Counter | `error_type` | Total errors |
| ... | ... | ... | ... |

### Logging

| Log Level | When Used |
|-----------|-----------|
| `debug` | Detailed debugging information |
| `info` | General operational information |
| `warn` | Warning conditions that should be reviewed |
| `error` | Error conditions that require attention |

**Structured Logging Fields**:
- `component`: Always set to `{component}`
- `request_id`: Unique request identifier (if applicable)
- `...`: Other component-specific fields

### Health Checks

| Endpoint/Mechanism | Purpose | Expected Response |
|--------------------|---------|-------------------|
| `GET /health` | Liveness check | `200 OK` |
| `GET /ready` | Readiness check | `200 OK` when ready |
| ... | ... | ... |

### Tracing

- **Span Name**: `dynamo.{component}.{operation}`
- **Attributes**: List key span attributes
- **Baggage**: List any propagated context

---

## Customization & Extension Points

### Extension Points

| Extension Point | How to Customize | Example |
|-----------------|------------------|---------|
| Custom {X} | Implement `{Interface}` and register | See `examples/{component}/custom_{x}.py` |
| Plugin system | Add to `{plugin_dir}` | ... |
| ... | ... | ... |

### Configuration-Based Customization

Describe what can be customized through configuration without code changes.

### Limitations

- **Cannot customize**: List things that cannot be changed
- **Planned extensions**: Future extension points under consideration

---

## Testing

### Test Locations

| Test Type | Location | Coverage Target |
|-----------|----------|-----------------|
| Unit tests | `tests/unit/{component}/` | 80%+ line coverage |
| Integration tests | `tests/integration/{component}/` | All public APIs |
| E2E tests | `tests/e2e/{component}/` | Critical user workflows |

### Running Tests

```bash
# Unit tests
pytest tests/unit/{component}/ -v

# Integration tests
pytest tests/integration/{component}/ -v

# E2E tests (requires running services)
pytest tests/e2e/{component}/ -v

# With coverage
pytest tests/unit/{component}/ --cov=dynamo.{component} --cov-report=html
```

### Test Categories

- **API Contract Tests**: Verify public API signatures are stable
- **Configuration Tests**: Test all CLI flags and environment variables
- **Error Handling Tests**: Test error codes and messages
- **Metrics Tests**: Verify metrics are emitted correctly
- **Backwards Compatibility Tests**: Test N-1 version compatibility

---

## Error Handling

### Error Codes

| Code | Name | Description | User Action |
|------|------|-------------|-------------|
| `{COMPONENT}_001` | `InvalidConfig` | Configuration validation failed | Check configuration values |
| `{COMPONENT}_002` | `ConnectionFailed` | Could not connect to dependency | Verify dependency is running |
| ... | ... | ... | ... |

### Exception Hierarchy

```
DynamoError
└── {Component}Error
    ├── {Component}ConfigError
    ├── {Component}ConnectionError
    └── ...
```

---

## 1.0 Standardization Status

| Requirement | Current Status | 1.0 Target | Owner |
|-------------|---------------|------------|-------|
| CLI flags documented | {Complete / In Progress / Not Started} | Complete | {Name} |
| Env vars documented | ... | Complete | ... |
| Metrics standardized | ... | Complete | ... |
| Python API typed | ... | Complete | ... |
| Test coverage | {X}% | 80%+ | ... |
| Error codes defined | ... | Complete | ... |
| Health checks implemented | ... | Complete | ... |

---

## Migration Guide (if applicable)

### Breaking Changes from 0.x

| Change | Old Behavior | New Behavior | Migration |
|--------|-------------|--------------|-----------|
| ... | ... | ... | ... |

### Deprecations

| Deprecated | Replacement | Removal Version |
|------------|-------------|-----------------|
| ... | ... | 2.0 |

---

## Related Documentation

- [Component Summary (Deep Dive)](link-to-component-summary.md)
- [1.0 API DEP](https://github.com/ai-dynamo/enhancements/blob/dynamo_api/deps/0000-dynamo-api-and-non-feature-requirements.md)
- [Design Document](link-if-applicable)
- [External Documentation](link-if-applicable)

---

## Changelog

### 1.0.0 (Target: 2026-03-11)
- Initial stable release
- ...

### 0.x.x
- Pre-release versions
