# Results Analysis

Interpreting performance test results is a skill in itself. Raw numbers are meaningless without context — this reference covers how to read, analyze, and act on performance data.

---

## Core Metrics

| Metric | Description | Target |
|---|---|---|
| **Throughput** | Requests per second (RPS / TPS) | ≥ defined baseline |
| **Response Time (p50)** | Median — half of requests faster than this | Per SLA |
| **Response Time (p95)** | 95% of requests faster than this — main SLA metric | Per SLA |
| **Response Time (p99)** | Tail latency — reveals worst-case behavior | Per SLA |
| **Error Rate** | % of failed requests | < 1% (or as defined) |
| **Active VUs / Concurrency** | Number of simulated users at any point | Matches load profile |
| **Network I/O** | Bytes/sec in and out — detects bandwidth bottlenecks | Headroom vs NIC capacity |

---

## Response Time Percentiles

Never rely on averages alone. They mask the tail:

```
Example: 100 requests
  98 requests at 100ms  ← average pulled down
   2 requests at 5000ms ← 2% of users wait 5 seconds

  Average: 198ms   ← looks fine
  p95:     100ms   ← looks fine
  p99:     5000ms  ← alerts you to the problem
  Max:     5100ms  ← confirms the problem
```

**Which percentile to use for SLAs:**
- p50 = typical user experience
- p95 = most users' worst case (standard SLA metric)
- p99 = tail latency (critical for high-volume services)
- p99.9 = extreme tail (needed for financial/medical systems)

---

## Reading JMeter Reports

### Summary Report columns
| Column | Meaning |
|---|---|
| # Samples | Total requests sent |
| Average | Mean response time (ms) — use sparingly |
| Min / Max | Absolute floor and ceiling |
| Std. Dev. | Variance — high deviation = inconsistent behavior |
| Error % | Percentage of failed requests |
| Throughput | Requests/second |
| KB/sec | Network throughput |
| p90 / p95 / p99 | Percentile response times |

### HTML Dashboard
Generated with `-e -o results/dashboard` flag. Key graphs:
- **Response Time Over Time** — spot degradation trends
- **Transactions Per Second** — confirm throughput matches expectation
- **Response Time Percentiles** — see distribution
- **Active Threads Over Time** — correlate with response time
- **Errors Over Time** — when and what errors spike

### JTL Analysis (programmatic)
```python
import pandas as pd

df = pd.read_csv('results.jtl')
df['elapsed'] = df['elapsed'].astype(int)
df['success'] = df['success'].astype(bool)

# Filter to successful requests only for latency analysis
success = df[df['success'] == True]

print(f"p50:  {success['elapsed'].quantile(0.50):.0f}ms")
print(f"p95:  {success['elapsed'].quantile(0.95):.0f}ms")
print(f"p99:  {success['elapsed'].quantile(0.99):.0f}ms")
print(f"Max:  {success['elapsed'].max():.0f}ms")
print(f"RPS:  {len(df) / (df['timeStamp'].max() - df['timeStamp'].min()) * 1000:.1f}")
print(f"Error rate: {(~df['success']).mean()*100:.2f}%")
```

---

## Reading k6 Reports

### Terminal summary output
```
✓ http_req_duration............: avg=182ms min=45ms med=156ms max=2.1s p(90)=312ms p(95)=489ms
✓ http_req_failed..............: 0.52% ✓ 52 ✗ 9948
✓ login_errors.................: count=2
  http_reqs.....................: 10000  83.33/s
  iterations....................: 2000   16.67/s
  vus...........................: 100
  vus_max.......................: 100
```

**Thresholds status:** `✓` = passed, `✗` = failed (non-zero exit code).

### k6 JSON output analysis
```python
import json

with open('results.json') as f:
    for line in f:
        point = json.loads(line)
        if point['type'] == 'Point' and point['metric'] == 'http_req_duration':
            # Process individual data points
            pass
```

---

## Bottleneck Identification Framework

When you see high response times or errors, use this framework:

```
High Response Time / Errors
        │
        ├─ Check: Is the error rate high?
        │    ├─ YES → What status codes? (503 = server overloaded, 504 = timeout, 429 = rate limit)
        │    └─ NO → Latency issue, not availability
        │
        ├─ Check: Is throughput (RPS) flat-lining below target?
        │    ├─ YES → Server saturated; find the saturated resource
        │    └─ NO → Throughput ok; issue is latency distribution
        │
        ├─ Check: Server CPU > 80%?
        │    ├─ YES → CPU bottleneck → profile code, optimize algorithms, scale horizontally
        │    └─ NO → CPU is not the limit
        │
        ├─ Check: Database slow query log showing queries > 100ms?
        │    ├─ YES → DB bottleneck → add indexes, optimize queries, connection pool tuning
        │    └─ NO → Look elsewhere
        │
        ├─ Check: JVM GC pauses (Java services)?
        │    ├─ YES → Memory tuning, GC algorithm selection, heap sizing
        │    └─ NO → Not GC
        │
        ├─ Check: Connection pool exhausted? (connection refused, pool timeout errors)
        │    ├─ YES → Tune pool size, connection timeout, check for connection leaks
        │    └─ NO → Not pool
        │
        └─ Check: Network I/O saturated?
             ├─ YES → Response payload too large? Bandwidth ceiling? CDN needed?
             └─ NO → Dig deeper with APM traces
```

---

## Analyzing Latency Patterns

### Response Time Over Time Shapes

**Stable (good):**
```
ms │ ~~~~~~~~~~~~~~~~~~~~
   └──────────────────────── Time
```
Consistent response time = healthy, stable system.

**Degrading (memory leak / connection exhaustion):**
```
ms │                    ╱‾‾‾
   │          ╱‾‾‾‾‾‾‾‾
   │ ╱‾‾‾‾‾‾‾
   └──────────────────────── Time
```
Gradual increase = something is accumulating over time.

**Sawtooth (GC or periodic batch job):**
```
ms │ ▁▂▃▄▅▆▇█▁▂▃▄▅▆▇█▁▂▃▄▅
   └──────────────────────── Time
```
Periodic spikes = GC pauses, scheduled jobs, connection pool refresh.

**Cliff (saturation point):**
```
ms │              ╱‾‾‾‾‾‾‾
   │ ─────────────
   └──────────────────────── Time
   [normal load]  [saturation]
```
Sudden jump at a load threshold = system has hit a constraint (thread limit, DB connection pool, queue depth).

---

## Comparing Runs (Trend Analysis)

Always compare the current run against a baseline — never evaluate a run in isolation.

| Metric | Run 1 (Baseline) | Run 2 | Delta | Status |
|---|---|---|---|---|
| p95 response time | 320ms | 380ms | +18.7% | ⚠️ Regression |
| Error rate | 0.1% | 0.08% | -20% | ✅ Improved |
| Throughput | 450 RPS | 480 RPS | +6.7% | ✅ Improved |
| p99 | 1200ms | 2100ms | +75% | ❌ Major regression |

Tools for trend analysis:
- **Grafana + InfluxDB**: Overlay runs as time-series overlays.
- **Gatling Enterprise**: Built-in run comparison.
- **JMeter + Jenkins Performance Plugin**: Trend charts per build.
- **Custom scripting**: Python/Pandas on JTL files.

---

## Reporting to Stakeholders

Structure your report:

```
1. EXECUTIVE SUMMARY
   - Test objective and load profile (1 paragraph)
   - Overall result: PASS / FAIL against SLA
   - Top finding (e.g., "p95 exceeded SLA by 40% above 500 VUs")

2. TEST PARAMETERS
   - Tool, VU count, duration, environment, data set

3. KEY METRICS TABLE
   - Throughput, p50/p95/p99, error rate vs target

4. GRAPHS
   - Response time over time
   - Throughput over time
   - Error rate over time
   - Server metrics (CPU, memory) if available

5. BOTTLENECK ANALYSIS
   - What degraded, at what load level, probable cause

6. RECOMMENDATIONS
   - Specific, actionable items (e.g., "Add index on orders.customer_id")
   - Priority and estimated impact

7. APPENDIX
   - Full metrics breakdown per endpoint
   - Error log samples
   - Environment configuration
```
