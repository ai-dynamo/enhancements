# Dynamo Local Resource Monitor

**Status:** Draft

**Authors:** keivenchang

**Category:** Architecture

**Replaces:** N/A

**Replaced By:** N/A

**Sponsor:** keivenchang

**Required Reviewers:** Jie Hao, Karen Chung

**Review Date:** TBD

**Pull Request:** TBD <!-- this PR -->

**Implementation PR / Tracking Issue:** [Linear DIS-1921](https://linear.app/nvidia/issue/DIS-1921), branch `keivenchang/dynamo-resource-monitor`

# Summary

A host-side Prometheus exporter that captures **per-process** VRAM, GPU
utilization, PCIe TX/RX, CPU, disk, and network I/O for the Dynamo
processes running on a single machine, scraped at **100 ms** (configurable
down to 20 ms) and rendered in a dedicated Grafana dashboard.

This is a **local/host-level** resource monitor — it watches Dynamo
processes from the outside (via `psutil` and `pynvml`), not the
request-lifecycle metrics emitted from inside the workers. It is
complementary and re-uses the existing Grafana + Prometheus stack
that hosts [0004-observability-metrics.md](./0004-observability-metrics.md).

# Motivation

When a Dynamo GPU test fails on engine startup — frontend returns 200,
no OOM in the log, no crash, the engine just sits until the
health-check timeout — there is no way to tell whether the cause was
VRAM exhaustion, allocator fragmentation, a slow torch-compile, or
PCIe contention. Three reasons no existing tool in the stack
distinguishes those:

**1. Sample rate.** DCGM scrapes at 1 s, Dynamo's request-lifecycle
metrics at 1–15 s. The events live on a 10–100 ms scale (cudaMalloc
defragment passes, PCIe weight bursts, torch-compile stalls). At 1 s
sampling they smear into a flat average. **We need 10–20 ms
resolution** to see them at all.

**2. Dynamo-framework aware.** The engine PID is buried under
`pytest → python → bash → python`. Generic per-PID exporters see
four unrelated rows; DCGM aggregates per device, not per process.
What's missing is a collector that **understands the Dynamo
framework** — recognizes how Dynamo workers are spawned, where the
engine actually lives in that chain, and the relationship between a
test invocation, the launch script, and the worker — and groups
everything into one logical worker on the dashboard. This is not a
generic process tree walker; it encodes Dynamo-specific lineage
patterns.

**3. Reuse existing infrastructure.** The Grafana + Prometheus stack
under `deploy/observability/` already carries DCGM, request-lifecycle
metrics, Loki, and Tempo. This work plugs in as one more Prometheus
instance, datasource, and dashboard provisioned through the same
paths — no parallel stack, one operational story.

Concrete gap: GH Actions job
[74054957724](https://github.com/ai-dynamo/dynamo/actions/runs/25245688688/job/74054957724)
— both `*_diffusion-2` cases timed out at 300 s, zero per-process
data captured.

## Goals

* Provide a continuous, sub-second timeline of per-Dynamo-process
  resource state suitable for forensic analysis of CI engine-startup
  failures.
* Group multiple PIDs that belong to the same logical Dynamo worker
  into a single dashboard row.
* Run alongside the existing observability stack without conflicting
  with it (separate Prometheus instance, separate datasource).
* Deployable in CI on the same hosts that run the GPU tests, with
  bounded retention so it does not fill disk.

### Non Goals

* Not a replacement for DCGM-exporter, the inference-pipeline
  Prometheus metrics, or any cluster-wide observability tool. This is
  a local diagnostic.
* Not intended for long-term storage or production SLO dashboards —
  retention is intentionally short (15 m default).
* No alerting or paging integration in the initial cut.
* Does not depend on or extend the request-lifecycle metrics covered
  by [0004-observability-metrics.md](./0004-observability-metrics.md);
  the two layers do not interact.

## Requirements

### REQ 1 Sample Resolution

The exporter **MUST** support a configurable scrape interval down to
**20 ms** and **MUST** default to **100 ms**. Slower defaults defeat
the purpose; events on the 10–100 ms scale **MUST** be observable in
the resulting timeseries.

### REQ 2 Per-Process Metrics

The exporter **MUST** emit, per Dynamo process: VRAM used and total,
GPU utilization, GPU temperature, PCIe TX/RX throughput, CPU
percentage, disk read/write, and network sent/received. Metric names
**MUST** follow the `dynamo_*` prefix convention.

### REQ 3 Subprocess Lineage Grouping

The exporter **MUST** walk the parent/child process tree, recognize
the Dynamo `pytest → python → bash → python` lineage, and emit
metrics labeled such that a single Grafana panel can show all PIDs
that belong to one logical worker as one timeline.

### REQ 4 Independence

The exporter **MUST** run as a separate process from any Dynamo
worker and **MUST NOT** be scraped by the same Prometheus instance
that scrapes request-lifecycle metrics. Scrape contention at sub-100
ms intervals against the workers themselves would introduce more
noise than signal.

### REQ 5 Dashboard

A Grafana dashboard **MUST** ship with the exporter, provisioned via
the existing `docker-observability.yml` profile, with quick-pick time
ranges down to at least 1 minute and refresh intervals down to at
least 500 ms.

# Proposal

## Components

1. **`dynamo_local_resource_monitor.py`** — a host-side Python
   exporter using `psutil` + `pynvml` + `prometheus_client`. Default
   port `8051`. Walks the process tree, recognizes Dynamo lineage,
   emits `dynamo_*` Prometheus metrics with `process_name` and
   `worker_group` labels.

2. **Dedicated Prometheus instance** — `prom/prometheus:v3.4.1`
   container on `:9091`, scrape interval `100ms`, retention `15m`.
   Configured via `dynamo_local_resource_monitor.yml`. Separate from
   the standard observability Prometheus to avoid conflating the two
   sample rates.

3. **Grafana datasource** —
   `grafana-datasource-local-resource-monitor.yml`, points at
   `http://dynamo-resource-monitor:9091`, declared with a
   `timeInterval: 100ms`.

4. **Grafana dashboard** —
   `grafana_dashboards/dynamo_local_resource_monitor.json`, title
   "Dynamo Local Resource Monitor", uid `dynamo-resource-monitor`,
   tagged `dynamo, gpu, vram, cpu, pcie, disk, network, monitoring`.
   Panels show VRAM, GPU util, PCIe TX/RX, CPU%, disk I/O, network
   I/O grouped by worker.

5. **Documentation** — `docs/observability/local-resource-monitor.md`
   with run instructions and a description of the lineage-detection
   heuristics.

## Lineage detection

The walker starts from each running Python process whose ancestry
includes a `pytest` invocation (or, in production runs, a
`dynamo serve` invocation), then walks descendants through any
intermediate `bash` launchers (`launch/agg_*.sh`) down to the final
worker process. All PIDs in the chain receive the same
`worker_group` label, derived from the test name (in CI) or the
service name (in production).

## Deployment

In CI: started as a sidecar process on the test host before the
pytest invocation, torn down after. In a developer environment:
brought up via `docker compose -f deploy/docker-observability.yml up
-d dynamo-resource-monitor grafana` plus the exporter on the host.
The exporter is a single Python file with no Dynamo runtime
dependency, so it can run inside or outside the devcontainer.

## Out-of-band on purpose

This monitor deliberately runs **outside** the Dynamo worker
processes it observes. That keeps it cheap (no instrumentation
overhead in the hot path), independent (a worker crash does not stop
data collection), and decoupled from the worker release cycle. It is
explicitly the wrong tool for measuring request latency or queue
depth — those belong to the request-lifecycle metrics layer covered
by [0004-observability-metrics.md](./0004-observability-metrics.md).

# Alternate Solutions

## Alt 1 Tune DCGM-exporter to sub-second sampling

**Pros:**

* No new tool; reuses an existing component already deployed in the
  observability profile.

**Cons:**

* DCGM-exporter is a *device* exporter, not a process exporter — it
  cannot attribute VRAM to a specific PID, which is the central
  requirement of this DEP.
* Sub-second DCGM scraping has measurable GPU overhead because the
  underlying NVML calls themselves are not free at high frequency.

**Reason Rejected:** does not satisfy REQ 2 (per-process) or REQ 3
(lineage grouping) regardless of how its sample rate is tuned.

## Alt 2 Add the same metrics to the existing Dynamo Prometheus exporter

**Pros:**

* One fewer Prometheus instance to operate.
* Co-located with request-lifecycle metrics for joint queries.

**Cons:**

* The existing exporter scrapes every 1–15 s by design. Adding
  100 ms metrics there would either drag the rest of the pipeline
  down to that rate (expensive) or require a separate scrape config
  inside the same Prometheus instance (which is what we are doing
  anyway, just with extra coupling).
* The existing exporter runs *inside* the worker. A worker that has
  failed to come up (the exact case we want to debug) is also a
  worker that is not exporting. Out-of-band collection is required.

**Reason Rejected:** the failure mode this DEP targets is precisely
when the worker is not yet emitting its own metrics. An in-process
exporter cannot observe its own startup failure.

## Alt 3 Generic `psutil` exporter without lineage detection

**Pros:**

* Simpler implementation; reuses any of several open-source
  per-process exporters.

**Cons:**

* Produces 4–6 unlabeled rows per test (pytest, python, bash, python
  again) with no way to tell which row corresponds to which logical
  worker. Forensic value is near zero when multiple tests run in
  parallel on the same host.

**Reason Rejected:** fails REQ 3.

# References

* Linear: [DIS-1921](https://linear.app/nvidia/issue/DIS-1921)
* Implementation branch: `keivenchang/dynamo-resource-monitor`
* Predecessor exploration: PR
  [#7968](https://github.com/ai-dynamo/dynamo/pull/7968)
  (`add-realtime-gpu-monitor-observability-tool`, closed without
  merge); this DEP is the cleaner architectural successor.
* Related Linear tickets:
  [DIS-1528](https://linear.app/nvidia/issue/DIS-1528) (static VRAM
  markers per test),
  [DIS-1700](https://linear.app/nvidia/issue/DIS-1700) (engine VRAM
  caps),
  [DIS-1616](https://linear.app/nvidia/issue/DIS-1616) (parallel
  GPU bin-packing),
  [DYN-2616](https://linear.app/nvidia/issue/DYN-2616)
  (request-lifecycle observability audit),
  [DIS-1745](https://linear.app/nvidia/issue/DIS-1745) (KV-cache
  transfer observability gaps).
* Companion DEP at a different layer:
  [0004-observability-metrics.md](./0004-observability-metrics.md)
  covers request-lifecycle metrics inside the workers; this DEP
  covers host-level resource visibility from outside the workers.
