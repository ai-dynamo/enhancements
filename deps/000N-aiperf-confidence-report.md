# RFC: AIPerf - Reporting Confidence of Measurement

**Target repo:** `ai-dynamo/aiperf`  
**Last updated:** 2026-01-20

## Summary

AIPerf recently introduced a Warmup Phase that can be executed before the main profile phase to eliminate cold-start effects and stabilize measurements. This RFC proposes adding a new parameter:

```bash
--num-profile-runs N  # default: 1
```

to repeat the profile benchmark N times (under the same configuration), and produce aggregated artifacts that quantify:

- run-to-run variance / noise
- repeatability (e.g., coefficient of variation)
- confidence intervals for key summary metrics

This builds on AIPerf's existing focus on controlled benchmarking: warmup, deterministic RNG via `--random-seed`, and configurable arrival patterns/modes.

## Motivation

Today, AIPerf provides strong single-run reporting (percentiles, throughput, etc.) and supports deterministic dataset generation through `--random-seed`.

However, a single run often isn't enough to establish trust because performance measurements can vary due to:

- System jitter (GPU clocks, background tasks)
- Networking variance
- Server internal scheduling/batching dynamics
- Periodic stalls or transient errors

When optimizing model hosting we frequently ask: **"Is this difference real or noise?"** Adding `--num-profile-runs` lets AIPerf answer that directly and consistently.

## Goals & Non-Goals

### Goals

1. Allow users to run the same profile benchmark multiple times with a single command
2. Produce aggregated results that quantify:
   - variance/standard deviation
   - repeatability (coefficient of variation)
   - confidence intervals for key metrics
3. Preserve all per-run artifacts (so users can debug outliers)
4. Make the feature work across existing benchmarking modes (concurrency, request-rate, request-rate+max-concurrency, trace replay)
5. Keep backward compatibility: default behavior unchanged when `--num-profile-runs=1`

### Non-Goals (initial scope)

- No new benchmark modes
- Not implementing full statistical hypothesis testing (A/B significance tests) in v1
- Not changing existing per-request schema or current "single run" summary format; we will add new aggregate artifacts

## Proposed UX

### CLI

Add:

```bash
--num-profile-runs <int>  # default: 1
```

Optional companion knobs (nice-to-have but not required for first cut):

```bash
--profile-run-cooldown-seconds <float>  # default: 0
# Sleep between runs to reduce correlation / allow steady-state recovery

--confidence-level <float>  # default: 0.95
```

### Warmup interaction

Because AIPerf now has an explicit Warmup Phase to eliminate cold-start effects, the default behavior should be:

- Warm up once, then run the N profile runs back-to-back

### Artifacts & output layout

With `--num-profile-runs N`, create:

```
.../profile_runs/
    run_0001/
      (existing run artifacts)
    run_0002/
      (existing run artifacts)
    ...
    run_000N/
      (existing run artifacts)

.../aggregate/
    profile_export_aiperf_aggregate.json
    profile_export_aiperf_aggregate.csv
    (optional) profile_export_aiperf_aggregate.md
```

### Aggregate JSON schema (proposal)

A single aggregate file should include:

#### Run metadata

- `num_profile_runs`, `num_successful_runs`, `failed_runs[]`
- benchmark config snapshot

#### Per-metric aggregation

- `mean`, `std`, `min`, `max`
- `cv` (coefficient of variation = std/mean) as a "noise score"
- confidence interval: `[ci_low, ci_high]` for mean or median (configurable)

#### For each "headline" metric we already report

Examples:
- throughput (RPS/TPS)
- TTFT p50/p90/p99
- ITL p50/p90/p99
- E2E p50/p90/p99
- error rate
- goodput (if enabled)

### Important detail: what we aggregate

We aggregate **run-level summary statistics**, not all per-request values pooled together.

Example:

- each run produces `ttft_p99_ms`
- we treat `{ttft_p99_ms_run_i}` for i=1..N as samples
- compute mean/std/CV/CI over those N values

This avoids conflating:

- "more requests in a run" with "more statistical certainty"

and provides a clean statement about run-to-run repeatability.

### Confidence interval methodology (simple + defensible)

For each metric, compute:

- sample mean μ
- sample std s
- standard error SE = s / sqrt(N)
- CI: μ ± t_(alpha/2, N-1) * SE

Many latency metrics are heavy-tailed; CI over means should be good enough for N≥3, while CI over p99 will be naturally wide unless runs are long/stable (>=1000 requests). This is good because it surfaces real uncertainty rather than hiding it.

### Failure handling & "trust"

If one run fails:

- do not discard the entire experiment
- write per-run artifacts for completed runs
- aggregate over successful runs
- record failures in aggregate metadata (`failed_runs[]` with error summaries)

This directly improves trust (and debugging) by removing the "all-or-nothing" outcome.

## Implementation sketch (exploratory)

### High-level flow

1. Parse CLI + build BenchmarkConfig as today
2. If warmup enabled, execute warmup phase once (existing behavior)
3. For i in 1..N:
   - run profile phase using the existing execution path
   - write outputs into `profile_runs/run_<i>/...`
   - extract the run summary metrics into an in-memory `RunSummary` object
4. After loop:
   - compute aggregated stats over `RunSummary[]`
   - write aggregate outputs (json/csv)
   - ensure `aiperf plot` can consume either:
     - a directory containing multiple runs
     - the aggregate output

### Possible new components

- `ProfileRunOrchestrator` - loop + artifact routing
- `RunSummaryExtractor` - reads the run's produced summary json/csv OR in-memory results
- `AggregateStatsComputer` - mean/std/CV/CI
- `AggregateArtifactWriter` - json/csv

### Backward compatibility

- Default remains `--num-profile-runs=1` → identical behavior and artifact structure
- New directory layout only appears when N>1
- No breaking changes to existing result schemas

## Example Usage

### Basic multi-run benchmark

```bash
aiperf \
  --num-profile-runs 5 \
  --model llama-3-8b \
  --endpoint-type openai_chat \
  --url http://localhost:8000/v1/chat/completions \
  --concurrency 10 \
  --num-prompts 1000
```

Output:
```
.../artifacts/
  profile_runs/
    run_0001/
      profile_export_aiperf.json
      profile_export_aiperf.csv
    run_0002/
      ...
    run_0005/
      ...
  aggregate/
    profile_export_aiperf_aggregate.json
    profile_export_aiperf_aggregate.csv
```

## Interpretation Guide

### Coefficient of Variation (CV)

The CV is a normalized measure of variability:

- **CV < 0.05** (5%) - Excellent repeatability, low noise
- **CV 0.05-0.10** (5-10%) - Good repeatability, acceptable noise
- **CV 0.10-0.20** (10-20%) - Moderate variability, consider more runs
- **CV > 0.20** (>20%) - High variability, investigate or increase N

### Confidence Intervals

The 95% CI tells you: "If we repeated this experiment many times, 95% of the time the true mean would fall in this range."

- **Narrow CI** - High confidence in the measurement
- **Wide CI** - More uncertainty, consider:
  - Increasing `--num-profile-runs`
  - Investigating sources of variance
  - Using `--profile-run-cooldown-seconds`

### Example interpretation

```
ttft_p99_ms: mean=152.7ms, cv=0.081, ci=[140.3, 165.1]
```

Interpretation:
- Average p99 TTFT is 152.7ms
- Run-to-run variability is 8.1% (good repeatability)
- We're 95% confident the true mean is between 140.3-165.1ms
- The 24.8ms CI width suggests moderate uncertainty with N=5 runs

## Benefits

### For Performance Engineers

1. **Quantify noise** - Know if a 5% improvement is real or within measurement error
2. **Compare configurations** - Determine if differences are statistically meaningful
3. **Debug outliers** - Per-run artifacts preserved for investigation
4. **Build trust** - Confidence intervals provide honest uncertainty estimates

### For CI/CD Integration

1. **Regression detection** - Flag when new code falls outside historical CI
2. **Automated validation** - Fail builds when CV exceeds threshold
3. **Trend analysis** - Track mean and variance over time

### For Research & Publications

1. **Reproducibility** - Report mean ± CI instead of single-run values
2. **Statistical rigor** - Proper uncertainty quantification
3. **Honest reporting** - Surface real measurement uncertainty

## Implementation

I'm happy to implement this feature and submit PR(s).
