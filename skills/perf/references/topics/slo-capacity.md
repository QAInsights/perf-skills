# SLO & Capacity Planning

> Scope: turning load-test results into service level objectives, error budgets, and capacity decisions. Complements `workload-design.md` (designing the load) and `results-analysis.md` (reading the numbers). Read those first if you have not designed or run the test yet.

This topic covers the *decision* layer of performance engineering: how to define SLIs/SLOs that mean something, how to convert a load-test curve into a capacity number and a headroom plan, and how to enforce all of it as an automated CI gate.

> For LLM-specific SLOs (TTFT, goodput), see `llm-inference.md`. For the tool syntax that enforces SLOs (`ensure`, `thresholds`), see the relevant tool file.

---

## SLI / SLO / Error Budget

### Definitions
- **SLI (Service Level Indicator)** - the actual measured signal, e.g. request p95 latency, error rate, availability.
- **SLO (Service Level Objective)** - the target you commit to for an SLI over a window, e.g. "p95 < 500ms for 99% of requests over 28 days."
- **Error Budget** - `1 - SLO`. A 99% SLO means a 1% error budget: you may violate the objective 1% of the time before it's a breach. Budgets make trade-offs explicit (ship features vs. burn reliability).

### Choosing SLIs
Pick SLIs users actually feel, not ones that are easy to measure:

| User-facing concern | SLI | Typical SLO |
|---|---|---|
| Responsiveness | Request p95 latency | < 500ms |
| Worst-case tail | Request p99 latency | < 1,500ms |
| Reliability | Error rate (5xx + timeouts) | < 1% (99% success) |
| Availability | Successfully served requests / total | 99.9% |
| Throughput | Sustained RPS at SLO latency | ≥ peak + headroom |
| Freshness (data) | Staleness of served data | < 60s |

For streaming/LLM, use token-aware SLIs (TTFT, ITL) and **goodput** (fraction of requests meeting all thresholds) instead of raw latency - see `llm-inference.md`.

### Windowing
- Use a **rolling window** (e.g. 28 days) so a single bad day does not trigger an alert; it consumes budget gradually.
- Report SLO attainment as `good_events / total_events` over the window. If attainment < SLO, the budget is exhausted.
- Set **alerts on burn rate**, not on instantaneous SLO pass/fail. Fast burn (e.g. 14× rate) means a major incident; slow burn means a trend to watch.

### Multi-window burn-rate alerting (pattern)
Define fast and slow burn signals so you catch both outages and slow erosion:
```
Fast burn:  burn rate ≥ 14  over 1h   → page
Slow burn:  burn rate ≥ 2   over 6h   → ticket
```
A burn rate of `X` means the budget is being consumed `X` times faster than the 28-day baseline allows.

---

## From Load Test to Capacity

A load test produces a curve of latency/error vs load. Capacity is the point on that curve where SLOs still hold, **plus** the headroom you keep for safety.

### The saturation curve
```
latency
  │              ╱‾‾‾‾‾‾‾   ← cliff: SLO breach
  │ ─────────────
  └──────────────────────── load (RPS/VUs)
   [safe zone]   [capacity point]
```
- **Capacity point** = highest sustained load at which all SLOs (p95, error rate, goodput) still pass.
- **Saturation point** = where throughput flatlines and latency explodes (the "cliff" in `results-analysis.md`).
- **Never plan to capacity.** Plan to capacity minus headroom.

### Headroom rule
- Target **peak production load at ~60-70% of measured capacity** under normal operation.
- Keep **~30-40% headroom** to absorb traffic spikes, node failures, and deployment blips without breaching SLO.
- For autoscaling systems, headroom is what lets new replicas come up before the SLO burns.

### Inverting Little's Law to size capacity
Given a target arrival rate and response time, concurrency is `N = λ × W` (see `workload-design.md`). Use it both ways:
- **To size load**: known λ and W → required VUs (already in workload-design).
- **To size infrastructure**: if the test shows a single replica sustains `C` RPS at SLO, replicas needed = `target_RPS / (C × utilization_target)`. With a 70% utilization target and 1,000 target RPS at 200 RPS/replica → `1000 / (200 × 0.7)` ≈ **8 replicas** (round up, add 1 for failure tolerance).

### Capacity from the goodput cliff
For LLM and streaming services, capacity = load at the goodput cliff (see `llm-inference.md`). Report "sustains 200 concurrent users at 99% goodput; goodput drops to 80% at 350" - that 200 is your planning number, 350 is your hard ceiling.

---

## Worked Example

**Inputs:** Peak prod = 800 RPS. Test shows p95 < 500ms and error < 1% hold up to 1,200 RPS on the current 6-replica cluster; saturation at ~1,500 RPS.

| Decision | Calculation | Result |
|---|---|---|
| Capacity point | measured SLO-holding load | 1,200 RPS |
| Planning target | 60-70% of capacity | 720-840 RPS |
| Headroom | 1,200 - 800 peak | 33% (acceptable) |
| Replicas at growth | if peak grows to 1,500 RPS: `1500 / (200 × 0.7)` | 11 replicas (was 6) |
| Alert threshold | 80% of capacity as early warning | alert at ~960 RPS |

---

## CI Regression Gating

SLOs are useless if they are only in a report. Enforce them automatically:

1. **Gate on thresholds, not on "it ran."** Every tool has an SLO mechanism - k6 `thresholds`, Artillery `ensure`, Gatling Enterprise assertions, JMeter exit codes. A run with no gate always passes CI (see each tool file's Common Mistakes).
2. **Compare against baseline, not just absolute.** A p95 of 480ms passes a 500ms SLO but is a +60% regression vs last week's 300ms. Track percentile deltas build-over-build.
3. **Set a regression threshold.** e.g. "fail the build if p95 rises > 10% vs baseline, or error rate > 0.5%." Avoid zero-tolerance (noise); avoid loose (misses real regressions).
4. **Require statistical significance for soak/large runs.** A 1-run blip should not fail CI; use multiple iterations or a confidence band.
5. **Publish the SLO report in the pipeline.** Surface p50/p95/p99, throughput, error rate, and budget consumption on every run so regressions are visible, not buried.

> For the per-tool gate syntax, see `../tools/k6.md` (`thresholds`), `../tools/artillery.md` (`ensure`), `../tools/gatling.md`, `../tools/jmeter.md`. For running these in pipelines, see `test-execution.md`.

---

## Common Mistakes

- **Averaging for SLOs** - an average of 200ms can hide a p99 of 5s. SLOs must be percentile-based.
- **SLO with no error budget** - a target with no burn policy becomes a meaningless number; you can never "spend" reliability intentionally.
- **Planning to the saturation point** - sizing for max measured throughput leaves zero margin; the first spike breaches SLO.
- **Confusing capacity with peak** - if peak ≈ capacity, you have no headroom. Target 60-70%.
- **One-shot capacity number** - capacity drifts as code, data, and dependencies change. Re-baseline on major releases.
- **Gating CI on run success only** - "test passed" ≠ "SLO met." Without a threshold/ensure gate, regressions ship green.
- **Ignoring tail latency in capacity** - a system can hold p95 at capacity while p99 is 10× worse; gate on the tail users actually feel.
- **No burn-rate alerting** - alerting on instantaneous SLO state misses slow erosion until the budget is already gone.

---

## Reporting SLO & Capacity

Structure the decision section for stakeholders:

```
1. SLO SUMMARY
   - p95 < 500ms: 99.2% attainment (budget: 1% used 0.8%)
   - Error rate < 1%: 99.97% attainment
2. CAPACITY
   - Measured capacity: 1,200 RPS @ SLO
   - Current peak: 800 RPS (33% headroom)
   - Saturation: ~1,500 RPS
3. RECOMMENDATION
   - Scale to 8 replicas before peak season (growth to 1,500 RPS)
   - Alert at 960 RPS (80% of capacity)
4. REGRESSION GATE
   - p95 > +10% vs baseline fails CI
```
