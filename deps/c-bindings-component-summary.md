# C Bindings Component Summary

## Overview

The C Bindings (`libdynamo_llm`) provide a C-compatible API for Dynamo's LLM functionality, enabling integration with C/C++ applications and other languages via FFI. The bindings wrap `dynamo-llm` and `dynamo-runtime` crates into a shared library with a stable C ABI.

## Location

- **Source**: `lib/bindings/c/`
- **Header Generation**: `cbindgen`
- **Library Name**: `libdynamo_llm_capi`
- **Output**: `libdynamo_llm_capi.so` / `libdynamo_llm_capi.a`

## Internal Dependencies

### Rust Crates (Wrapped)

| Crate | Location | Provides |
|-------|----------|----------|
| `dynamo-llm` | `lib/llm/` | LLM functionality |
| `dynamo-runtime` | `lib/runtime/` | Distributed runtime |

### Build Dependencies

| Tool | Purpose |
|------|---------|
| cbindgen | C header generation |
| Cargo | Rust build system |

## Public Interface

### Library Types

```toml
[lib]
name = "dynamo_llm_capi"
crate-type = ["cdylib", "staticlib"]
```

- `cdylib` - Dynamic shared library (`.so`, `.dylib`, `.dll`)
- `staticlib` - Static library (`.a`, `.lib`)

### Generated Header

Location: `lib/bindings/c/include/dynamo_llm.h`

Generated via `cbindgen.toml` configuration:
```toml
language = "C"
include_guard = "DYNAMO_LLM_H"
```

### Core Functions (Expected)

Based on typical C FFI patterns:
```c
// Runtime management
dynamo_runtime_t* dynamo_runtime_create(const char* config);
void dynamo_runtime_destroy(dynamo_runtime_t* runtime);

// Request handling
dynamo_request_t* dynamo_request_create(...);
int dynamo_request_send(dynamo_runtime_t* rt, dynamo_request_t* req);
dynamo_response_t* dynamo_request_recv(dynamo_request_t* req);

// Memory management
void dynamo_free(void* ptr);
```

## User/Developer Interaction

### 1. Building the Library

```bash
cd lib/bindings/c
cargo build --release
```

Output:
- `target/release/libdynamo_llm_capi.so`
- `target/release/libdynamo_llm_capi.a`

### 2. Using from C/C++

```c
#include "dynamo_llm.h"

int main() {
    // Initialize runtime
    dynamo_runtime_t* runtime = dynamo_runtime_create("{}");

    // Use API...

    // Cleanup
    dynamo_runtime_destroy(runtime);
    return 0;
}
```

### 3. Linking

```bash
gcc -o myapp myapp.c -L/path/to -ldynamo_llm_capi
```

## Packaging & Containers

### Build Artifacts

| Artifact | Platform | Notes |
|----------|----------|-------|
| `libdynamo_llm_capi.so` | Linux | Dynamic library |
| `libdynamo_llm_capi.a` | Linux | Static library |
| `dynamo_llm.h` | All | C header |

### Container Integration

- Not typically pre-built in containers
- Build from source when needed

## Service Interface & I/O Contract

### C ABI Stability

- Uses `extern "C"` for stable ABI
- Opaque pointer types for internal structures
- Explicit memory management (caller or callee ownership)

### Error Handling

Typical C error patterns:
- Return codes (0 = success, negative = error)
- Error message retrieval functions
- Nullable pointers for optional returns

## 1.0 Standardization Checklist

| Area | Current State | 1.0 Requirements |
|------|---------------|------------------|
| Header generation | cbindgen | Document API |
| ABI stability | Implicit | Version header, soname |
| Error codes | TBD | Define error enum |
| Memory ownership | TBD | Document conventions |
| Thread safety | TBD | Document guarantees |

## Customization & Extension

### Adding New Functions

1. Add Rust function with `#[no_mangle]` and `extern "C"`
2. Regenerate header with cbindgen
3. Update documentation

### Example Rust FFI

```rust
#[no_mangle]
pub extern "C" fn dynamo_example_function(
    input: *const c_char,
) -> *mut c_char {
    // Implementation
}
```

## Related Documentation

- [cbindgen Documentation](https://github.com/eqrion/cbindgen)
- Rust FFI best practices

## Tests

- Rust unit tests in `src/lib.rs`
- Integration tests would require C test harness

## Current Status

The C bindings appear to be in development. Key areas to define for 1.0:
1. Complete API surface
2. Memory management conventions
3. Error handling patterns
4. Thread safety guarantees
5. Versioning scheme
