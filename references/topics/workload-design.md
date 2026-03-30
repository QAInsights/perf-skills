# Workload Design

Workload design is the most impactful phase of performance testing. A poorly designed workload produces results that are irrelevant to production behavior — no matter how well-scripted the test is.

---

## Step 1: Understand Production Traffic

Before writing a single line of script, gather real data:

| Data Source | What to Extract |
|---|---|
| Web server access logs | Request URLs, methods, frequency, user session patterns |
| APM (Datadog, Dynatrace) | Transaction mix, throughput by endpoint, peak hours |
| CDN logs | Real geographic distribution and peak RPS |
| Analytics (GA, Mixpanel) | User journeys, funnel drop-offs, session duration |
| Database slow query logs | Expensive queries to include in test scenarios |

**Key questions to answer:**
- What is the **peak concurrent user count** (not sessions per day)?
- What is the **throughput target** (RPS or TPS) at peak?
- What is the **transaction mix** (% login, % browse, % checkout)?
- What is the **session length** and **average think time**?
- Are there **batch or background jobs** running during peak?

---

## Step 2: Define the Workload Model

### Concurrency Model Selection

**Closed Model (VU-based)**
- VU count is fixed; next iteration starts only when previous completes.
- Think time is part of the model.
- Best for: web apps where users "hold" sessions.
- Tools: JMeter (Thread Groups), Gatling (`rampUsers`), Locust.

**Open Model (Arrival-rate)**
- New requests arrive at a fixed rate regardless of outstanding requests.
- More realistic for APIs, microservices, and public-facing systems.
- Best for: REST APIs, event-driven systems, queueing scenarios.
- Tools: k6 (`constant-arrival-rate`), Gatling (`constantUsersPerSec`), JMeter (Throughput Shaping Timer).

### When to Use Each

| Scenario | Model |
|---|---|
| E-commerce web app with sessions | Closed |
| REST API serving mobile clients | Open |
| Microservice with queue consumer | Open |
| Banking portal with session timeouts | Closed |
| Public API with rate limiting | Open |

---

## Step 3: Define the Load Profile

### Common Load Profiles

**Ramp-Up + Steady State + Ramp-Down**
```
VUs  │    ▁▂▃▄▅▆▇█████████▇▆▅▄▃▂▁
     │
     └────────────────────────────── Time
     [Ramp 5m][  Hold 10m  ][Down 2m]
```

**Step Load (Staircase)**
```
VUs  │         ████
     │    ████ ████ ████
     │ ████ ████ ████ ████
     └──────────────────────── Time
```
Use for: finding the load level at which behavior degrades.

**Spike Test**
```
VUs  │              ▐█▌
     │              ▐█▌
     │    ██████████▐█▌█████████
     └──────────────────────────── Time
```
Use for: testing autoscaling response, queue behavior under burst.

**Soak/Endurance**
```
VUs  │    ████████████████████████
     └──────────────────────────── Time (2–8 hours)
```
Use for: detecting memory leaks, connection pool exhaustion, log rotation issues.

---

## Step 4: Calculate Concurrency

### Little's Law (the fundamental formula)

```
N = λ × W

N = average concurrency (VUs)
λ = arrival rate (requests/sec = throughput)
W = average response time + think time (seconds)
```

**Example:**
- Target: 1,000 RPS
- Average response time: 200ms = 0.2s
- Average think time: 3s
- W = 0.2 + 3 = 3.2s
- N = 1,000 × 3.2 = **3,200 VUs needed**

This is why VU count alone is meaningless without knowing think time and response time targets.

---

## Step 5: Define Transaction Mix

Break down the workload into realistic business transactions:

```
Example: E-commerce application

Transaction          | % of Traffic | Think Time
---------------------|--------------|------------
Homepage browse      |     40%      |   5–15s
Product search       |     25%      |   3–8s
Product detail view  |     20%      |   5–10s
Add to cart          |     10%      |   2–5s
Checkout + payment   |      5%      |   30–60s
```

Implement in scripts:
- **JMeter**: Use Throughput Controller (% mode) or separate Thread Groups with ratios.
- **k6**: Use scenarios with `weight` or multiple VU groups with proportional counts.
- **Gatling**: Use `Population` with `userWeight`.
- **Locust**: Use `@task(weight)` decorator.

---

## Step 6: Set SLA Targets

Every test must have explicit, measurable SLA targets before execution. Negotiate these with stakeholders:

| Metric | Threshold Example |
|---|---|
| p50 (median) response time | < 200ms |
| p95 response time | < 500ms |
| p99 response time | < 1,500ms |
| Error rate | < 1% |
| Throughput floor | ≥ 500 RPS |
| Max response time | < 5,000ms |

**Why percentiles matter more than averages:**
Average response time hides outliers. p95 tells you what 95% of users experience — averages can look fine while 10% of users timeout.

---

## Think Time Design

Think time simulates the time a real user spends reading a page, filling a form, or deciding what to do next.

| Distribution | When to use |
|---|---|
| Constant (e.g., 3s) | Simple tests, known fixed pacing |
| Uniform random (1–5s) | Basic variability, most common |
| Gaussian (mean=3s, std=1s) | Most realistic for web users |
| Negative exponential | Rare; models Poisson arrival patterns |

**Rule of thumb:** Never set think time to 0 in closed-model tests unless explicitly modeling a batch job or API benchmark with no user interaction.

---

## Pacing vs Think Time

- **Think time**: Time a VU sleeps *within* a transaction (between pages).
- **Pacing**: Time between the *start* of each iteration (controls throughput more directly).

Pacing is used when you want `N` iterations per hour per VU regardless of response time. Example:
- Target: 360 transactions/hour per VU = 1 transaction every 10 seconds
- Pacing = 10s (start next iteration 10s after previous started)

---

## Ramp-Up Strategy

Bad ramp-up causes a "thundering herd" — thousands of VUs hitting the server simultaneously, warming up connection pools, JVM, and caches all at once. This is unrealistic and produces misleading results.

**Good ramp-up:**
- Rule of thumb: Ramp to 100% over at least 5–10 minutes for large tests.
- Allow cache warm-up (exclude ramp period from SLA analysis).
- For step tests, hold each step for at least 2–3× the transaction response time.

---

## Checklist Before Writing Scripts

- [ ] Production traffic baselines collected (RPS, response time, error rate)
- [ ] Concurrency calculated (Little's Law applied)
- [ ] Transaction mix defined (% per transaction type)
- [ ] Think time distribution specified
- [ ] Load profile drawn (ramp, hold, spike, soak)
- [ ] SLA thresholds defined and signed off by stakeholders
- [ ] Environment capacity verified (enough headroom to push load)
- [ ] Test data strategy defined (see test-data.md)
- [ ] Monitoring and APM hooks confirmed (see observability.md)
