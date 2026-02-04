# DEP-0000: Adaptive Compute Scheduling for Async Inference Pipelines

- **Status:** Draft
- **Authors:** Ryan Olson (rolson@nvidia.com)
- **Category:** Architecture
- **Created:** 2026-02-04

## The Problem: CPU Compute on Async I/O Threads

Async runtimes like tokio use cooperative scheduling: tasks are expected to yield quickly so the executor can service timers, I/O completions, and other concurrent work. CPU-bound operations violate this assumption. A 50ms `tokenizer.encode()` call blocks the worker thread for its entire duration, starving every other task scheduled on that thread — timers fire late, I/O completions queue up, and concurrent requests see inflated Time To First Token (TTFT).

[PR #5865](https://github.com/pytorch/serve/pull/5865) identified `tokenizer.encode()` as a concrete source of event loop blocking in dynamo. But tokenization is one example of a general pattern. The same problem hits any CPU-bound work in an async context: JSON parsing, compression, tensor preprocessing, data transformation, server-side agentic tools, and more.

The question is not *whether* to address this, but *how*.

### Proposed Approach: Learn the Cost, Schedule Smartly

This DEP proposes adaptive scheduling using a **Multi-Armed Bandit (MAB)** algorithm. A multi-armed bandit is a decision-making framework borrowed from statistics: imagine a row of slot machines ("arms"), each with an unknown payout. You pull arms, observe results, and over time concentrate on the best one. Here, the two "arms" are:

- **Inline** — run the work on the current I/O thread (zero overhead, but blocks the thread while it runs)
- **Offload** — move the work to a dedicated compute thread pool (adds ~100-500ns overhead, but frees the I/O thread immediately)

The scheduler measures the actual execution time of each function and learns which choice is better. Fast functions (< 50us) get inlined — the overhead of moving them to another thread costs more than the work itself. Slow functions (> 250us) get offloaded automatically — blocking the I/O thread for that long risks starving other work. Functions in between adapt based on observed cost and current system pressure.

The key insight: **the right scheduling decision depends on the function, the input, and the current load — not a one-size-fits-all rule.** A `tokenizer.encode()` call might take 5us for a short prompt and 50ms for a long one. The MAB learns this per-function, adapting as workload characteristics change. Hard safety guardrails prevent dangerous decisions regardless of what the algorithm has learned — work over 250us is always offloaded, no exceptions.

## Evidence: Measured Impact

We built an evaluation suite that simulates real-world compute work and measures how it affects I/O responsiveness. Before showing results, here is what the experiment measures and what the columns mean.

### What we're testing

Each test simulates `tokenizer.encode()` — a CPU-bound function that takes ~50ms per call. While these compute tasks run, a background **probe task** repeatedly asks tokio to sleep for 100 microseconds and then measures how long the sleep *actually* took. Under no load, the sleep overshoots by about 1ms (tokio's normal timer resolution). Under heavy compute load, the sleep can overshoot by much more — because the thread is busy with compute and can't wake the probe on time.

### How to read the results

- **P95 / P99** — The 95th and 99th percentile sleep overshoot of the probe task, in microseconds. Lower is better. These tell you: "for 95% (or 99%) of measurements, the I/O task was delayed by no more than this amount." P95 captures typical behavior; P99 captures worst-case spikes.
- **Interference** — The ratio of P95 under load to P95 with no load. **1.0x means no degradation** — compute work has no visible effect on I/O responsiveness. **2.0x means I/O tasks take twice as long to wake up** as they would with no compute running. This is the key metric: it tells you whether compute work is starving the I/O event loop.
- **Throughput** — How many compute tasks completed per second. Higher is better. A low number means compute work is bottlenecked (e.g., only one thread is available to process it).

### The strategies compared

- **TokioOnly (8T)** — Standard tokio runtime with 8 worker threads, no separate compute pool. All work (I/O and compute) shares the same 8 threads. This is what most applications do today.
- **AlwaysInline (1T+7R)** — Pool partition: 1 tokio thread for I/O, 7 rayon threads for compute. But compute is *not offloaded* — it still runs on the single tokio thread, blocking it entirely. This is what happens if you partition threads without a scheduling strategy.
- **AlwaysOffload (1T+7R)** — Same pool partition, but all compute is moved to rayon. The tokio thread stays free for I/O.
- **Adaptive MAB (1T+7R)** — Same pool partition, but the MAB scheduler decides per-task whether to inline or offload based on learned execution times and safety guardrails.

### Traffic pattern

The test uses burst traffic to model production reality: traffic arrives in spikes, not at constant rates. Each test runs **10 burst cycles** — a 500ms burst at 300 tasks/s (150 tasks per burst), followed by a 500ms quiet period. The average rate is 150 tasks/s, but the instantaneous burst rate of 300/s temporarily oversubscribes all 8 threads. During each burst, 8 threads can complete ~80 of the 150 launched tasks; the remaining ~70 queue up and drain during the quiet period.

### Results

| Strategy | P95 | P99 | Interference | Throughput |
|----------|-----|-----|--------------|------------|
| TokioOnly (8T, no pool partition) | 4321us | 17722us | **2.2x** | 80/s |
| AlwaysInline (1T+7R, pool partition) | 1959us | **49907us** | 1.0x | **10/s** |
| AlwaysOffload (1T+7R, pool partition) | 1957us | 1967us | 1.0x | 114/s |
| Adaptive MAB (1T+7R, pool partition) | 1957us | 1963us | **1.0x** | 114/s |

### What this shows

**TokioOnly (8T)** — During bursts, all 8 threads saturate with 50ms compute tasks. The probe's P95 jumps from ~2ms to ~4.3ms (**2.2x interference**) and P99 hits 17.7ms. I/O tasks are visibly delayed during traffic spikes because every thread is busy with compute. Throughput is 80/s — lower than expected because threads are contending.

**AlwaysInline (1T+7R)** — The worst outcome. The P95 interference looks fine (1.0x) because quiet periods between bursts let the probe recover — but P99 hits **50ms**, the full blocking duration of a single compute task. Throughput collapses to **10/s** because only the single tokio thread processes compute, and each task monopolizes it for 50ms. This is what happens when you partition threads but don't offload the work.

**AlwaysOffload and Adaptive MAB (1T+7R)** — Both show **1.0x interference** at all percentiles and **114/s** throughput. Compute tasks queue on rayon's 7-thread pool during bursts — that's fine, rayon is designed for it. The tokio thread stays free for I/O throughout, including during peak burst load. The MAB achieves this automatically through its safety guardrails: 50ms work exceeds the 250us hard ceiling, so it's offloaded from the first call.

**The takeaway:** Queuing compute work on a dedicated pool during traffic spikes is fine. Starving the I/O event loop is not.

## The Obvious Fix: spawn_blocking

[PR #5865](https://github.com/pytorch/serve/pull/5865) wraps `tokenizer.encode()` in `tokio::task::spawn_blocking`:

```rust
let tokenizer = self.tokenizer.clone();  // clone the tokenizer
let prompt = prompt.clone();              // clone the input
let encoding = tokio::task::spawn_blocking(move || {
    tokenizer.encode(&prompt)
}).await??;
```

This works, but has three costs:

1. **Cloning overhead** — the tokenizer and prompt must be cloned on every call because `spawn_blocking` requires `'static` captures.
2. **Wrong pool model** — `spawn_blocking`'s thread pool is designed for I/O blocking (file system, DNS), not compute. It uses an unbounded thread pool that grows under load, with no work-stealing or CPU affinity.
3. **No learning** — small prompts (5us encode) pay the same offload overhead as large prompts (50ms encode). Our evaluation shows always-offload is +1480% slower than adaptive for fast work (~10us).

Teams reach for `spawn_blocking` because it's available and it stops the immediate starvation. But it doesn't solve the architectural question of how compute should be scheduled.

## Strategic Options

There are three broad strategies for handling CPU compute in an async inference pipeline:

### Option A: All-tokio, scale cores

Keep everything on tokio. Add enough threads that compute doesn't starve I/O.

**Advantage:** No architectural change. Simple mental model — everything runs on the same executor.

**Cost:** You must always provision enough threads to handle **peak burst load** without starving I/O — not average load, but the worst-case spike. A 50ms encode blocks whichever thread it lands on; the only defense is having enough other threads free to handle I/O while some are blocked. Our evaluation shows that even 8 threads show 2.2x interference under burst traffic (300/s during spikes) — the system has no headroom when traffic arrives in spikes rather than at constant rates. Provisioning for peak means over-provisioning CPU for the common case. There is also no work-stealing optimization for CPU-parallel compute (tokio's work-stealing is designed for async task scheduling, not data-parallel computation).

### Option B: Strict pool partition

Separate tokio (I/O) and rayon (compute) pools on disjoint CPU sets. Compute gets its own work-stealing pool with CPU affinity — purpose-built for CPU-parallel work.

**Advantage:** Clean separation. Compute work gets a pool optimized for it (rayon: work-stealing, bounded concurrency, CPU affinity). I/O threads are never blocked by compute.

**Cost:** Concentrates async work onto fewer threads. If you then *inline* compute on those threads, starvation is worse than before — the eval shows P99 hitting 50ms and throughput collapsing to 10/s with 1T+7R. Requires a clear strategy for deciding what runs where.

### Option C: Oversubscribed pools with reserved I/O cores

Both tokio and rayon pools exist, potentially with more total threads than physical cores. Guarantee that at least some tokio threads are pinned to cores *without* rayon threads competing. Tokio's work-stealing scheduler pulls I/O work to these reserved cores when other tokio threads are blocked.

**Advantage:** More forgiving than strict partition. Tokio's work-stealing provides a safety net — if some tokio threads are occupied, others can pick up I/O work.

**Cost:** The reserved tokio threads must remain I/O-focused. If compute is inlined on them, the safety net disappears. Still requires discipline about what runs where.

### Recommendation

Options B and C both require a **scheduling strategy** — a way to decide, for each unit of CPU work, whether to run it on the current (tokio) thread or offload it to the compute pool. Option A avoids this decision but trades CPU efficiency for simplicity.

For GPU-bound inference servers where CPU is provisioned tightly (not scaled for headroom), Options B or C provide better resource efficiency. The rest of this DEP addresses the scheduling problem common to both.

## The Scheduling Problem

If we partition threads (Options B or C), how do we decide what runs on which pool?

### Static annotation (manual)

Developer marks each function as "fast" (inline) or "slow" (offload). Simple and explicit.

**Problem:** Wrong for variable workloads. `tokenizer.encode()` is 5us for short prompts, 50ms for long ones. A static annotation must be conservative (always offload), paying overhead on every call regardless of actual cost.

### Static threshold

Fixed time cutoff (e.g., 100us). Anything expected to exceed the threshold gets offloaded.

**Problem:** One threshold for all functions. No pressure awareness — same threshold under high and low load. Requires per-deployment tuning. Doesn't account for functions whose cost varies by input.

### Adaptive (Multi-Armed Bandit)

Runtime measures each function's execution time and learns the optimal strategy per-function. Thompson Sampling explores both strategies (inline vs. offload) and converges to the better choice. Adapts to changing workloads and system pressure.

The inline-vs-offload tradeoff is fundamental:

| | Fast work (<50us) | Slow work (>250us) |
|---|---|---|
| **Inline** | Wins (zero overhead) | Loses (blocks event loop) |
| **Offload** | Loses (overhead > work) | Wins (protects event loop) |

For mixed workloads — which is reality, since small prompts and large prompts arrive on the same endpoint — neither static choice is optimal. Hard safety guardrails prevent the runtime from ever inlining work that exceeds 250us, regardless of what the algorithm thinks. The MAB doesn't need to "learn" that 50ms work should be offloaded; the guardrails block it from the first call.

## Evaluation Results

All numbers from a single machine run on a 16-core x86_64 Linux system (30 physical cores available). The evaluation suite measures each claim independently. Run: `cargo run --example mab_evaluation --release`

All latency tests use the same probe methodology described in the Evidence section above: a background task measures how long a 100us sleep *actually* takes, and the interference ratio tells you how much compute work is delaying I/O. **1.0x = no impact on I/O, higher = I/O is being starved.**

### Tokenizer Test (50ms, burst traffic @ 300/s)

This is the headline result from the Evidence section above (repeated here for completeness in the detailed results). See that section for full setup, column definitions, and analysis.

| Strategy | P95 | P99 | Interference | Throughput |
|----------|-----|-----|--------------|------------|
| TokioOnly (8T) | 4321us | 17722us | **2.2x** | 80/s |
| AlwaysInline (1T+7R) | 1959us | **49907us** | 1.0x | **10/s** |
| AlwaysOffload (1T+7R) | 1957us | 1967us | 1.0x | 114/s |
| Adaptive MAB (1T+7R) | 1957us | 1963us | **1.0x** | 114/s |

### Latency Under Moderate Load (3ms @ 1500/s)

The tokenizer test uses 50ms work — an extreme case. What about lighter compute work that's still non-trivial?

**What we're testing:** 3ms of CPU-bound work per task (think JSON parsing a medium-sized payload, or a small tensor preprocessing step), spawned at 1500 tasks/s. This is well below the 50ms tokenizer case, but the high spawn rate means the single tokio thread in a 1T+7R configuration is frequently busy. We compare the same four strategies, using the same probe methodology: a background task measures I/O responsiveness while compute work runs alongside it.

| Strategy | P95 | Interference | Throughput |
|----------|-----|--------------|------------|
| TokioOnly (8T) | 1190us | 0.6x | 601/s |
| AlwaysInline (1T+7R) | 2925us | 1.5x | 331/s |
| AlwaysOffload (1T+7R) | 1684us | 0.8x | 642/s |
| Adaptive MAB (1T+7R) | 1665us | 0.8x | 671/s |

**Reading the results:** TokioOnly with 8 threads handles this fine (0.6x — I/O is unaffected). But in a partitioned 1T+7R setup, inlining 3ms work on the single tokio thread causes **1.5x interference** — I/O tasks take 50% longer to wake up. That's the single thread getting blocked by compute 1500 times per second. Offloading (either always or via MAB) brings interference down to **0.8x**, keeping I/O responsive. The MAB achieves this while retaining the flexibility to inline truly fast work when appropriate.

### Throughput: Does Adaptive Scheduling Leave Performance on the Table?

If we always offload *everything* to rayon, do we lose performance on fast work? This test answers that question.

**What we're testing:** Streams of 500 items are processed two ways: `compute_map()` (always offloads every item to rayon) and `adaptive_map()` (the MAB decides per-item whether to inline or offload). We test four workload sizes — fast (~10us, like a small string operation), medium (~100us), slow (~500us), and a realistic mix (60% fast, 30% medium, 10% slow). Each workload runs 3 times and results are averaged. **Throughput** is how many items are processed per second — higher is better. **Speedup** shows how much faster the adaptive approach is compared to always-offload.

| Workload | compute_map (always offload) | adaptive_map | Speedup |
|----------|------------------------------|--------------|---------|
| Fast (~10us) | 87,995/s | 1,390,598/s | +1480% |
| Medium (~100us) | 62,313/s | 209,871/s | +237% |
| Slow (~500us) | 28,585/s | 44,319/s | +55% |
| Mixed (60/30/10) | 65,098/s | 243,493/s | +274% |

**Reading the results:** For fast work (~10us), always-offload is catastrophically wasteful — moving 10us of work to another thread costs more than doing the work itself. The MAB learns to inline fast work, achieving **15x higher throughput**. For slow work, both approaches offload, so the gap is smaller (+55%). On realistic mixed workloads, the MAB delivers **+274%** throughput by inlining the fast items and offloading the slow ones — the best of both worlds.

### Learning: How Quickly Does the MAB Converge?

A scheduler that takes thousands of observations to learn the right decision is useless in practice. This test measures how many function calls the MAB needs before it consistently picks the right strategy.

**What we're testing:** The MAB scheduler is given either fast work (~20us, should be inlined) or slow work (~500us, should be offloaded). We run up to 100 observations and record when the MAB reaches "stability" — 5 consecutive correct decisions in a row. **Observations to Stable** is the number of function calls before the MAB locks in. **Final Decision** is what the MAB chose after learning.

| Work Type | Observations to Stable | Final Decision | Correct? |
|-----------|------------------------|----------------|----------|
| Fast (~20us) | 5 | Inline | Yes |
| Slow (~500us) | 6 | Offload | Yes |

**Reading the results:** The MAB converges in **5-6 observations** — essentially the minimum possible (it needs a few samples of each strategy to compare them). After training, fast work is inlined 100% of the time and slow work is offloaded 100% of the time.

### Adaptation: What Happens When Workloads Change?

Production workloads aren't static. A batch of short chat messages might be followed by a batch of long documents. Can the MAB adapt when the characteristics of a function change?

**What we're testing:** The same MAB instance first sees 200 fast observations (~20us each), then 200 slow observations (~500us each) — simulating a shift from short prompts to long documents. Decisions are tracked in windows of 50 to show the trajectory. **Inline %** and **Offload %** show what the MAB chose in each window.

| Observation Range | Inline % | Offload % |
|-------------------|----------|-----------|
| Fast 1-50 | 100% | 0% |
| Fast 151-200 | 100% | 0% |
| Slow 1-50 | 14% | 86% |
| Slow 51-200 | 0% | 100% |

**Reading the results:** During the fast phase, the MAB inlines everything (correct). When the workload shifts to slow, it detects the change and switches to offloading within **~10 observations**. The brief 14% inline rate in the first slow window is the MAB exploring — by the second window it's fully committed to offload. Exponential decay (half-life ~2000 observations) ensures old "this function is fast" data doesn't anchor decisions when the reality has changed.

### Guardrails: Safety Under Pressure

The MAB learns over time, but what about the first few calls, or unusual load spikes? Safety guardrails provide hard limits that override the MAB's learned decisions when the system is under pressure.

**What we're testing:** 150us work (moderate — above the pressure-sensitive threshold of 100us, below the hard ceiling of 250us) is run at increasing load levels on a 4-thread tokio runtime. The system computes a **pressure index** from how many tasks are in-flight and how fast new ones arrive. At each load level, 100 scheduling decisions are made. **GR Activations** counts how many times a guardrail overrode the MAB to force offloading. **Inline %** shows what fraction of work was actually inlined.

| Load Level | Pressure | GR Activations | Inline % |
|------------|----------|----------------|----------|
| Low (p=0.7) | 0.7 | 0 | 99% |
| Medium (p=2.2) | 2.2 | 0 | 99% |
| High (p=3.8) | 3.8 | 99 | 1% |
| Very High (p=7.6) | 7.6 | 99 | 1% |
| Extreme (p=10.0) | 10.0 | 99 | 1% |

**Reading the results:** At low and medium load, the 150us work is fast enough to inline — the MAB freely does so (99% inline, no guardrail activations). At high load (pressure > 3.0), guardrail GR2 activates: "the system is under enough pressure that even moderate compute should be offloaded." The result is **297 guardrail activations** across the three high-load tiers, forcing offload to protect the event loop. This is the safety net: even if the MAB's learned model says "inline this," the guardrails override when conditions are dangerous.

## How MAB Scheduling Works

On each call, `choose()` checks guardrails in order — GR0 (single-worker protection), GR1 (hard 250us ceiling), GR2 (pressure-sensitive threshold), GR3 (strike suppression after repeated slow inlines). If no guardrail fires, Thompson Sampling draws from each arm's posterior distribution and picks the lower-cost option. After execution, `finish()` updates the per-function statistics with exponential decay (half-life ~2000 observations).

Guardrails provide immediate safety regardless of learned state:

- **GR0**: Single-worker protection — very conservative when there's only 1 tokio thread (any block is total starvation)
- **GR1**: Hard ceiling — never inline if EMA > 250us
- **GR2**: Pressure-sensitive — tighter threshold (100us) under high load (pressure > 3.0)
- **GR3**: Strike suppression — ban inline after repeated slow executions (>1ms)

> **Full algorithm details:** [`docs/mab.md`](../docs/mab.md) — Thompson Sampling, cost model, decay, configuration knobs
>
> **Implementation:** [`src/mab/scheduler.rs`](../src/mab/scheduler.rs)

## Implementation Example: loom-rs

loom-rs is one implementation of the pool-partition + adaptive scheduling approach (Options B/C above). It combines a tokio runtime with a rayon compute pool, partitioned by CPU affinity, and provides the MAB scheduler as the bridge between them.

The API provides three tools for different situations — the MAB is one tool in the toolbox, not the only one:

### `spawn_async()` — I/O-bound async work

For work that is obviously lightweight or I/O-bound. Runs on tokio threads with task tracking for graceful shutdown. ~10ns overhead (token allocation only). Use this for network calls, database queries, timer-driven work — anything that yields quickly.

```rust
// Lightweight async work stays on tokio
let response = runtime.spawn_async(async {
    client.get(url).await
}).await?;
```

### `spawn_compute()` — known-heavy CPU work

For work that is obviously expensive. Always offloads to rayon's work-stealing pool — no MAB decision, no learning overhead. ~100-500ns overhead (Arc state + cross-thread handoff). Use this when you *know* the work is heavy: large matrix multiplies, image processing, compression of large buffers.

```rust
// Known-heavy work always goes to rayon
let result = runtime.spawn_compute(|| {
    tokenizer.encode(&large_prompt)  // known 50ms+ operation
}).await;
```

### `spawn_adaptive()` — variable or unknown work

For work where the cost depends on input or is not known ahead of time. The MAB scheduler measures execution time and learns per-function whether to inline (zero overhead) or offload (protect event loop). Use this when the same function can be fast or slow depending on input — like `tokenizer.encode()` which is 5us for short prompts and 50ms for long ones.

```rust
// Variable work — MAB learns the right strategy
let result = runtime.spawn_adaptive(|| {
    tokenizer.encode(&prompt)  // 5us or 50ms depending on prompt length
}).await;
```

### Stream mode

`adaptive_map()` extends futures streams with per-item MAB scheduling:

```rust
use loom_rs::ComputeStreamExt;
use futures::stream::{self, StreamExt};

let results: Vec<_> = stream::iter(prompts)
    .adaptive_map(|p| tokenizer.encode(&p))
    .collect()
    .await;
```

### Choosing the right tool

| Situation | Tool | Why |
|-----------|------|-----|
| HTTP request, DB query, timer | `spawn_async()` | I/O-bound, yields quickly, no compute to offload |
| Large prompt tokenization, image resize, zstd compress | `spawn_compute()` | Known-heavy, always offload, skip the MAB |
| `tokenizer.encode()` on variable-length input | `spawn_adaptive()` | Cost varies by input, MAB learns the breakpoint |
| JSON parse (usually fast, occasionally large) | `spawn_adaptive()` | Mostly inline (fast), offload when large |
| Preprocessing pipeline with mixed ops | `adaptive_map()` | Stream of items with variable compute cost |

### Integration with dynamo

[PR #5916](https://github.com/pytorch/serve/pull/5916) shows the primary change is `tokio::test` to `loom_rs::test`. The runtime provides `spawn_compute` (always offload), `spawn_async` (I/O-bound), and `spawn_adaptive` (MAB decides) as drop-in alternatives to `spawn_blocking`.

Compare to the `spawn_blocking` pattern from PR #5865 — no cloning needed (`spawn_adaptive` captures by move like any closure), no unbounded thread pool, and the MAB learns per-function whether to inline or offload.

## Why This Matters for Dynamo

1. **The problem exists today.** [PR #5865](https://github.com/pytorch/serve/pull/5865) shows `tokenizer.encode()` blocking tokio. Similar patterns exist throughout the codebase — JSON parsing, tensor operations, preprocessing transforms. Every CPU-bound call on a tokio worker is a potential starvation source.

2. **Scale-out (Option A) Provision more CPU cores.** Provision more CPU cores to handle the increased load. There are a lot of CPU cores free in a LLM inference system. Only a few cores are needed to drive the GPU compute, the remainder can be used for CPU-bound tasks. Agressive scale out the problem and there will be no issues.

3. **Naive pool partitioning is dangerous without scheduling.** Partitioning threads without a strategy for what runs where concentrates async onto fewer threads. Inlining 50ms compute on 1 tokio thread collapses throughput to 10/s (vs 114/s offloaded) with P99 hitting the full 50ms blocking duration. The MAB + guardrails eliminate this entirely (1.0x interference, 114/s throughput).

4. **Adaptive scheduling provides better resource efficiency.** The MAB inlines fast work (zero overhead) and offloads slow work (protects event loop), achieving +274% throughput on mixed workloads compared to always-offload. `spawn_blocking`'s unbounded thread pool doesn't provide this optimization. Offloading to a compute queue provides a feedback signal for when to scale out.

> *tokenizer.encode() is one example, but the pattern applies everywhere async code does CPU work.*

## Alternate Scheduling Approaches

Given the scheduling problem (when to inline vs. offload), here are alternative approaches to the MAB:

### Alt 1: Static Threshold

**Approach**: Use a fixed execution time threshold (e.g., 100us) to decide inline vs offload.

**Pros**: Simple to implement, zero learning overhead, deterministic.

**Cons**: No per-function learning (one threshold for all functions), no pressure awareness (same threshold under high and low load), requires manual tuning per deployment.

**Rejected**: Doesn't adapt to different function profiles, changing hardware, or varying system load. A threshold tuned for one workload is wrong for another.

### Alt 2: UCB1 (Upper Confidence Bound)

**Approach**: Use UCB1 instead of Thompson Sampling for arm selection.

**Pros**: Deterministic (no sampling), well-studied theoretical guarantees, simpler implementation.

**Cons**: Less adaptive to non-stationary environments (UCB1 assumes stationary rewards), no natural posterior distribution for cost modeling, confidence bounds grow slowly.

**Rejected**: Thompson Sampling handles changing workloads better due to decay-weighted posteriors. The stochastic nature of Thompson Sampling also provides more natural exploration in the face of uncertainty.

### Alt 3: Manual Annotation Only

**Approach**: Require developers to annotate each function as "fast" (inline) or "slow" (offload) using `spawn_compute()` or direct execution.

**Pros**: Zero runtime overhead, explicit control, no surprises.

**Cons**: Requires developer knowledge of execution times, wrong for variable workloads (same function can be fast or slow depending on input), doesn't adapt to hardware changes, maintenance burden.

**Rejected**: Manual annotation is still appropriate for known-slow functions (`spawn_compute()`), but the MAB handles the variable and unknown cases.

### Alt 4: Work-Stealing Between Pools

**Approach**: Instead of separate Tokio and Rayon pools, use a unified work-stealing scheduler that can move tasks between async and compute contexts.

**Pros**: Automatic load balancing, no explicit decision needed, can handle bursty workloads.

**Cons**: Fundamentally different architecture from Tokio + Rayon, complex implementation, doesn't solve per-function optimization (work-stealing is about load balancing, not about whether to block the event loop), would require forking or replacing Tokio's scheduler.

**Rejected**: Wrong abstraction level. The problem isn't load balancing between pools; it's deciding whether a specific function should run on the event loop or be moved off it. Work-stealing doesn't prevent starvation when a stolen task blocks the stealer.

## References

- Thompson, W.R. (1933). "On the Likelihood that One Unknown Probability Exceeds Another in View of the Evidence of Two Samples." *Biometrika*, 25(3/4), 285-294.
- Chapelle, O. & Li, L. (2011). "An Empirical Evaluation of Thompson Sampling." *Advances in Neural Information Processing Systems*, 24.
- Russo, D.J., Van Roy, B., Kazerouni, A., Osband, I., & Wen, Z. (2018). "A Tutorial on Thompson Sampling." *Foundations and Trends in Machine Learning*, 11(1), 1-96.
- Welford, B.P. (1962). "Note on a Method for Calculating Corrected Sums of Squares and Products." *Technometrics*, 4(3), 419-420.
- [`docs/mab.md`](../docs/mab.md) — Full algorithm details, guardrails, configuration knobs
- [`src/mab/scheduler.rs`](../src/mab/scheduler.rs) — Implementation
- [`examples/mab_evaluation.rs`](../examples/mab_evaluation.rs) — Evaluation suite
- [Dynamo PR #5865](https://github.com/pytorch/serve/pull/5865) — tokenizer.encode() blocking tokio
- [Dynamo PR #5916](https://github.com/pytorch/serve/pull/5916) — loom-rs integration
