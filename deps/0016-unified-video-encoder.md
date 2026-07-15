# Unified Video Encoder for Dynamo Diffusion Backends

**Status**: Draft

**Authors**: Sergey Plotnikov

**Category**: Architecture

**Replaces**: N/A

**Replaced By**: N/A

**Sponsor**: [Name of code owner or maintainer to shepard process]

**Required Reviewers**: [Names of technical leads that are required for acceptance]

**Review Date**: [Date for review]


**Implementation PR / Tracking Issue**: https://github.com/ai-dynamo/dynamo/pull/11121

# Summary

Dynamo currently serves video generation through three independent backends —
**vLLM** (via vLLM-Omni), **TensorRT-LLM**, and **SGLang**. Each backend
re-implements the final "raw frames → encoded video" step in its own way, with
its own codec, hardware assumptions, and code path. This document proposes a
single, shared video-encoding layer in `dynamo.common` that all three backends
call. The goal is to **unify encoding** so that codec, container, and hardware
support are implemented once, are consistent across backends, and are trivial to
extend (new codecs, new accelerators) without touching backend-specific code.

# Motivation

The three video backends share the same frontend, request schema, and storage
layer, but each ships its own frames→bytes encoder with divergent codecs,
hardware assumptions, and duplicated logic. This makes adding a codec, container,
or hardware accelerator (e.g. Intel XPU / VA-API) an N-times change and produces
inconsistent output across backends. Consolidating the single divergent step
into one shared encoder removes the duplication and makes hardware/codec support
a one-place change.

## Goals

* Unify the frames→bytes encode step for all three backends behind one shared
  entry point in `dynamo.common`.
* Preserve existing NVIDIA (NVENC) behavior unchanged.
* Make new hardware (XPU) and new codecs (notably HEVC) a single-place addition.
* Expose encoding controls (codec / container / HW accel / device) via
  environment variables.

### Non Goals

* Changing the frontend, request schema, or the storage/persistence layer.
* Per-backend CLI flags (may be layered on later).
* Removing the legacy helper functions in this change.

# Proposal

## Current Architecture

The video request travels through five stages. Stages 1 and 5 are **already
shared** by all three backends; stages 2–4 are where they diverge.

### Table 1 — Request handling (frontend → backend handler)

| Stage | vLLM | TensorRT-LLM | SGLang |
|---|---|---|---|
| HTTP / prompt processing | Dynamo Frontend (`dynamo.frontend`, OpenAI-compatible) | *same* | *same* |
| Shared request protocol | `NvCreateVideoRequest` / `VideoNvExt` (`dynamo.common.protocols`) | *same* | *same* |
| Backend entry point | `OmniHandler.generate()` | `VideoGenerationHandler.generate()` | `VideoGenerationWorkerHandler.generate()` |

> All three share the **same frontend and request schema**. The frontend routes
> `/v1/videos` to the appropriate worker endpoint; only the handler class differs.

### Table 2 — Generation (who produces the frames)

| | vLLM | TensorRT-LLM | SGLang |
|---|---|---|---|
| Generation call | `AsyncOmni.generate()` | `DiffusionEngine.generate()` | `DiffGenerator.generate()` |
| Owning package | `vllm_omni` | `dynamo.trtllm` (wraps `tensorrt_llm._torch.visual_gen`) | `sglang.multimodal_gen` |

> The call site for all three is **inside Dynamo handler code** — raw pixels are
> returned across the package boundary into Dynamo, which is what makes a shared
> Dynamo-side encoder possible without any upward dependency.

### Table 3 — Raw pixel output format

| | vLLM | TensorRT-LLM | SGLang |
|---|---|---|---|
| Type | `stage_output.images` — `list` (may hold one 5-D array) | `torch.Tensor` `(1, T, H, W, C)` `uint8` | `list[PIL.Image \| np.ndarray]` |
| Normalization helper | `normalize_video_frames()` | `video[0].cpu().numpy()` | per-frame `np.array(...)` |

> Three different in-memory shapes/types are produced. Each backend's own
> `to_canonical()` converter absorbs *its* shape into the canonical format, so
> only the canonical `np.ndarray (T, H, W, 3) uint8` ever reaches the shared
> encoder.

### Table 4 — Encoding (frames → bytes)

| | vLLM | TensorRT-LLM | SGLang |
|---|---|---|---|
| Encode function | `DiffusionFormatter._encode_video()` | `encode_to_video_bytes()` (`dynamo.common.utils.video_utils`) | `_frames_to_video()` (inline) |
| Encoder library | diffusers `export_to_video` → **ffmpeg** | `imageio.v3.imwrite(buffer, frames, extension=".mp4", codec="h264_nvenc")` → **ffmpeg** | `imageio.get_writer(buffer, format="mp4", codec="h264_nvenc")` → **ffmpeg** |
| Codec / HW | H.264 (libx264, **software**) | H.264 (**NVENC**), VP9 (`libvpx-vp9`) for webm | H.264 (**NVENC**) |
| Container(s) | mp4 | mp4 (webm code-capable) | mp4 |

> All three ultimately call **ffmpeg** (through imageio or diffusers). Only
> TensorRT-LLM uses the shared helper; SGLang duplicates equivalent logic inline,
> and vLLM has its own path. Containers other than mp4 are rejected by the
> handlers today even where the encoder could produce them.

### Table 5 — Persisting the encoded bitstream

| | vLLM | TensorRT-LLM | SGLang |
|---|---|---|---|
| Disk / object write | `upload_to_fs()` → `fs.pipe()` (`dynamo.common.storage`) | *same* | *same* |
| Backend (fsspec) | `DirFileSystem`; `file://` → local disk, `s3://`/`gs://` → object store | *same* | *same* |
| Response wrapping | `VideoData(url=… \| b64_json=…)` | *same* | *same* |

> Persistence is **already unified**. The encoders only produce bytes; the
> storage layer decides where the bytes land. (The unused `encode_to_mp4()`
> file-path variant bypasses this and is dead code.)

### Current flow chart

```mermaid
graph LR
    P[Prompt /v1/videos] --> FE[Dynamo Frontend<br/>shared]

    FE --> V[vLLM<br/>AsyncOmni.generate]
    FE --> T[TRT-LLM<br/>DiffusionEngine.generate]
    FE --> S[SGLang<br/>DiffGenerator.generate]

    V --> Ev[export_to_video<br/>ffmpeg / software H.264]
    T --> Et[imageio + ffmpeg<br/>NVENC H.264]
    S --> Es[imageio + ffmpeg<br/>NVENC H.264]

    Ev --> W[upload_to_fs / fs.pipe<br/>shared]
    Et --> W
    Es --> W
    W --> D[(file:// / s3:// …)]
```

The fork in the middle (three encoders) is the only real divergence — and the
target of this work.

## Proposed Architecture

Insert a single shared encoder between the backends and the existing file
writer. The shared encoder accepts **only a canonical frame format** —
`np.ndarray (T, H, W, 3) uint8` RGB. Each backend owns a small `to_canonical()`
converter that maps *its own* native output into that format before calling
`encode_video()`. Conversion logic that operates purely in the canonical domain
(float→uint8 scaling, alpha drop, PIL→array stacking, contiguity) lives once in
`dynamo.common` as small primitives; each backend converter composes those
primitives with its own shape mapping.

This keeps backend-specific shape knowledge inside the backend that produces it,
and gives the shared encoder a single, narrow, well-typed input contract — no
runtime type-sniffing, and adding a new backend never touches `dynamo.common`.

> **Design note — why not one shared converter?** An earlier draft placed a
> single `to_canonical_frames()` in `dynamo.common` that sniffed all three
> backend shapes. That reintroduces the very "N-times change" coupling this DEP
> removes (a new backend means editing shared code) and inverts the dependency
> direction (shared infra knowing backend internals). Pushing conversion to the
> producing edge fixes both.

```mermaid
graph LR
    V[vLLM frames] --> Cv[vLLM<br/>to_canonical]
    T[TRT-LLM frames] --> Ct[TRT-LLM<br/>to_canonical]
    S[SGLang frames] --> Cs[SGLang<br/>to_canonical]

    P[dynamo.common<br/>canonical primitives] -.shared by.-> Cv
    P -.-> Ct
    P -.-> Cs

    Cv --> U{encode_video<br/>canonical only}
    Ct --> U
    Cs --> U

    U -->|NVIDIA| N[imageio → ffmpeg<br/>NVENC]
    U -->|new HW / codecs| F[ffmpeg CLI<br/>raw pipe]

    N --> W[upload_to_fs / fs.pipe<br/>unchanged]
    F --> W
    W --> D[(file:// / s3:// …)]
```

### Canonical frame format (encoder ABI)

`encode_video()` accepts exactly one input shape and validates it on entry
(raising `ValueError` on wrong ndim, channel count, or dtype):

| Aspect | Contract |
|---|---|
| Type | `np.ndarray` |
| Shape | `(T, H, W, 3)` |
| dtype | `uint8` |
| Range | `0–255` |
| Channel order | RGB |

**Per-backend converters** (each lives in its own package, next to the handler
that produces the frames):

| Backend | Native output | Converter |
|---|---|---|
| vLLM | `stage_output.images` — list holding one 5-D array | `dynamo.vllm` `to_canonical()` |
| TensorRT-LLM | `torch.Tensor (1, T, H, W, C)` | `dynamo.trtllm` `to_canonical()` |
| SGLang | `list[PIL.Image \| np.ndarray]` | `dynamo.sglang` `to_canonical()` |

**Shared canonical-domain primitives** in `dynamo.common` (no backend
knowledge), composed by the converters — e.g. `ensure_uint8_rgb(arr)`,
`pil_frames_to_array(list)`, `drop_alpha(arr)`.

### Two encode paths

| Path | When | Why this mechanism |
|---|---|---|
| **imageio → ffmpeg (NVENC)** | NVIDIA platforms (current behavior) | Keeps the proven, working path; minimal risk; no behavior change for existing users. |
| **ffmpeg CLI (raw pipe)** | New hardware / codecs | imageio's ffmpeg plugin does **not** expose the options needed for hardware acceleration on non-NVIDIA encoders (device selection, `hwupload`, hardware filter chains). Driving `ffmpeg` directly via the command line (piping raw frames to stdin) is the only way to reach those encoders and to add codecs imageio doesn't surface. |

The key principles: **(a)** the existing NVENC path is preserved unchanged;
**(b)** the file-writing stage (`upload_to_fs`) is untouched; **(c)** codec /
hardware divergence collapses into one dispatch point; **(d)** the encoder's
input is the canonical format only — all backend-shape divergence is resolved
*before* the shared layer, inside each backend's `to_canonical()`.

## Codecs and Hardware Support

### Table 3a — Current hardware support

| Backend | CPU/software | NVIDIA (NVENC) |
|---|---|---|
| vLLM | ✅ (libx264) | — |
| TensorRT-LLM | (fallback) | ✅ |
| SGLang | (fallback) | ✅ |

### Table 3b — Current codecs / containers

| Codec | Container | Available in |
|---|---|---|
| H.264 / AVC | mp4 | all three |
| VP9 | webm | TRT-LLM (encoder-capable; handler-gated) |

### Table 3c — Unified encoder support (target)

| Capability | Current | Unified target |
|---|---|---|
| Software (CPU) | ✅ | ✅ |
| NVIDIA NVENC | ✅ | ✅ |
| **New accelerator (XPU)** | — | ✅ (new ffmpeg-CLI path) |
| H.264 / AVC | ✅ | ✅ |
| VP9 | partial | ✅ |
| **HEVC / H.265** | — | ✅ |
| mp4 container | ✅ | ✅ |
| webm container | partial | ✅ |

> Net effect: the unified encoder supports **everything supported today, plus**
> a new hardware accelerator and additional codecs (notably HEVC), without
> changing any backend's generation code.

## Encoding Controls

### Table 4a — Controls available today

| Control | Mechanism | Notes |
|---|---|---|
| Response format (`url` / `b64_json`) | request field `response_format` | per-request |
| Container (`output_format`) | request field | effectively mp4-only (handlers reject others) |
| FPS | request field `fps` / `nvext.fps` | per-request |
| Codec | — | hardcoded (`h264_nvenc` / libx264) |
| Hardware accelerator | — | hardcoded per backend |
| Hardware device | — | not selectable (except `DYNAMO_VAAPI_DEVICE`, local POC) |

### Table 4b — Controls to add

| Control | Values | Default behavior | Override |
|---|---|---|---|
| Codec selection | H.264 / HEVC / VP9 | container-appropriate default | explicit |
| Container | mp4 / webm | mp4 | explicit |
| HW acceleration | NVENC / XPU / CPU | **auto-detect** from platform | force a specific encoder (incl. CPU) |
| HW device selection | NVIDIA device index; XPU DRM render node | first available | explicit per accelerator |

### Configuration mechanism

The codebase already uses a **`flag_name` + `env_var`** pattern (e.g.
`--http-host` / `DYN_HTTP_HOST`), and a device-selection env var precedent
already exists (`DYNAMO_VAAPI_DEVICE`). Both CLI flags and env vars are
therefore feasible.

**Recommendation:** because the encoder lives in shared infra
(`dynamo.common`) and must serve three separate backend arg parsers, use
**environment variables as the universal baseline** (one set of `DYN_VIDEO_*`
knobs read by the shared encoder), with **optional per-backend CLI flags**
layered on top later following the existing `flag_name`/`env_var` convention.
This keeps the shared layer self-contained while still allowing CLI overrides
where a backend wants them.

| Proposed knob | Example env var | Example CLI (optional) |
|---|---|---|
| Codec | `DYN_VIDEO_CODEC` | `--video-codec` |
| Container | `DYN_VIDEO_CONTAINER` | `--video-container` |
| HW accelerator | `DYN_VIDEO_HW_ACCEL` (`auto`/`nvenc`/`xpu`/`cpu`) | `--video-hw-accel` |
| HW device | `DYN_VIDEO_DEVICE` (index or render node) | `--video-device` |

## Testing

## Principle

The encoder is now one shared component with a narrow canonical ABI, so the
tests mirror that split:

* **Encoder behavior is tested once, in `dynamo.common`**, against canonical
  arrays — the shared suite has *no* backend knowledge.
* **Each backend tests only its own `to_canonical()` converter and its handler
  adapter.**

No encoder logic is duplicated per backend, and "this backend emits shape X"
lives with the backend, never in `dynamo.common`.

## Layer A — Shared encoder suite (`dynamo.common`)

Location: `components/src/dynamo/common/tests/test_video_utils.py`.

| Area | What it checks |
|---|---|
| Canonical primitives (`ensure_uint8_rgb`, `pil_frames_to_array`, `drop_alpha`) | float→uint8 scaling, alpha drop, PIL→array stacking, contiguity |
| `encode_video()` ABI validation | rejects non-canonical input (wrong ndim / channel count / dtype) with `ValueError` |
| `encode_video()` dispatch | control-resolution order (arg > `DYN_VIDEO_*` env > auto-detect) and `auto` → `nvenc`/`xpu`; both encode paths mocked |
| ffmpeg-CLI path | mock `shutil.which` / `subprocess.run`; assert the VA-API command line |
| Real round-trip (`skipif` no ffmpeg) | encode N canonical frames → demux/decode → assert frame count, `W×H`, container magic — the one test that touches a real bitstream |

## Layer B — Backend adapter suites (per backend)

Location: each backend's own `tests/`.

| Test | What it checks |
|---|---|
| `to_canonical()` round-trip | build a known-truth canonical array with **distinctive per-pixel values** → synthesize the backend's real native shape from it → `to_canonical()` → assert **bit-exact equal** to the truth (distinctive values catch axis / channel-order bugs) |
| Handler adapter | run the handler with `encode_video` patched; assert it passes canonical frames and forwards `fps` / `container` |
| Response wrapping | `url` → storage upload; `b64_json` → base64 of the returned bytes |

## Migration of existing tests

| Test file | Change |
|---|---|
| `…/trtllm/tests/test_trtllm_video_diffusion.py` | Repatch the ≈9 `encode_to_video_bytes` sites to `encode_video`; drop the `output_format=` kwarg assertions (now `container=`, `fps` positional); move frame-shape handling into the new `to_canonical()` round-trip test |
| `…/common/tests/test_video_utils.py` | Keep the `encode_to_video_bytes` coverage (function retained); add the Layer-A cases above. The removed `to_canonical_frames` god-function has no direct test — it is replaced by the primitives (Layer A) plus per-backend converters (Layer B) |
| SGLang / vLLM-Omni video output | Add Layer-B suites (converter round-trip + handler adapter); no old inline encoder is mocked today, so nothing breaks |

## CI placement — what we add, where it lands

CI selects tests by **marker expression** (`pytest -m "…"`) over a repo-wide
collection — no test file is enumerated in any workflow, so a correctly placed
and marked file is picked up automatically. Every test carries one marker from
each of Lifecycle / Test Type / Hardware (enforced by the `pytest-marker-report`
pre-commit hook), and all markers are pre-registered (`--strict-markers`).

| File | Markers | CI job (marker expr) |
|---|---|---|
| `components/src/dynamo/common/tests/test_video_utils.py` | `unit`, `pre_merge`, `gpu_0` (no backend marker) | CPU job: `pre_merge and not (vllm or sglang or trtllm) and gpu_0` |
| `components/src/dynamo/trtllm/tests/test_trtllm_video_diffusion.py` | `unit`, `trtllm`, `pre_merge`, `gpu_0` | TRT-LLM job: `pre_merge and trtllm and gpu_0` |
| `components/src/dynamo/vllm/tests/…` (Layer-B) | `unit`, `vllm`, `pre_merge`, `gpu_0` | vLLM job: `pre_merge and vllm and gpu_0` |
| `components/src/dynamo/sglang/tests/…` (Layer-B) | `unit`, `sglang`, `pre_merge`, `gpu_0` | SGLang job: `pre_merge and sglang and gpu_0` |

> Placement note: SGLang tests go under `sglang/tests/`, **not**
> `sglang/request_handlers/` — the latter is excluded from pytest collection by
> an `--ignore-glob` in `pyproject.toml`. All suites are fully mocked; the real
> round-trip is CPU / libx264 and `skipif`-guarded, so no job needs a GPU.

# Alternate Solutions

N/A — the only considered alternative was leaving each backend's encoder in
place (status quo), which keeps the N-times duplication and was rejected for the
reasons given in Motivation.
