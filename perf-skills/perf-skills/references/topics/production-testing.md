# Production & Staging Performance Testing

Running tests in production is not inherently dangerous — it's a discipline. Many organizations run production performance testing routinely. The key is controlled blast radius, observability, and rollback readiness.

---

## Staging Environment Testing

### What Makes Staging Valid for Performance Testing

Staging is only a reliable proxy for production if:

| Factor | Requirement |
|---|---|
| **Hardware** | Same instance type, CPU, memory as production |
| **Architecture** | Same number of replicas, same DB tier, same cache config |
| **Data volume** | DB has similar row counts to production (schema-level) |
| **Network** | Similar topology (no shortcutting the load balancer) |
| **Dependencies** | Real downstream services (not mocks for performance tests) |

Using mocks for load tests is only appropriate for component-level isolation tests — full user journey tests must hit real (or realistic stub) dependencies.

### Staging Limitations
- Cache hit rates will differ if data set is too small.
- Third-party integrations may behave differently (rate limits are usually lower in test).
- Infrastructure auto-scaling may not be configured identically.
- Cold-start performance distorts early-stage results — pre-warm before measuring.

### Pre-Warm Strategy
```bash
# Run a smoke test (1 VU) for 5 minutes before starting the load test
# This warms:
# - JVM JIT compilation
# - Connection pools
# - Application-level caches
# - CDN edge caches (if applicable)

k6 run --vus 1 --duration 5m --tag phase=warmup warmup.js
k6 run --vus 500 --duration 20m --tag phase=load main.js
```

---

## Production Testing Strategies

### Strategy 1: Canary Deployment + Load Test

Route a small % of production traffic to the new release and observe:

```
Production Traffic
    │
    ├─ 95% → Stable instances (v1.0)
    └─ 5%  → Canary instances (v1.1)  ← Monitor closely
```

Gradually increase canary traffic percentage while monitoring:
- Error rate (should not increase)
- p95 response time (should not increase)
- Business metrics (conversion rate, etc.)

**Tools:** AWS ALB weighted target groups, Kubernetes Argo Rollouts, Istio traffic splitting, NGINX split_clients.

### Strategy 2: Shadow Testing (Traffic Mirroring)

Mirror production traffic to a shadow cluster — it receives all production requests but its responses are discarded:

```
Real User Request
    │
    ├─ Live → Production (v1) → Response to user
    └─ Mirror → Shadow (v2)  → Response discarded (no user impact)
```

Shadow cluster processes real production workload — perfect for validating performance of a new version without any user impact.

**Tools:** AWS ALB request mirroring, Istio `mirror`, NGINX `mirror` directive.

```yaml
# Istio traffic mirroring
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
spec:
  http:
  - route:
    - destination:
        host: my-service-v1
    mirror:
      host: my-service-v2
    mirrorPercentage:
      value: 100.0
```

### Strategy 3: Synthetic Load Injection in Production

Inject artificial load at low volumes (typically 5–10% of peak) during off-peak hours:

- Use a dedicated load generator with production-class credentials.
- Tag synthetic requests to exclude from business metrics dashboards.
- Use a synthetic user pool (not real users).
- Monitor production SLAs during injection.

**When to use:** Validate autoscaling behavior, warm new nodes before peak, run regression checks post-deployment.

### Strategy 4: Chaos Engineering (Resilience Testing)

Intentionally inject failures to test how the system degrades:

| Experiment | Tool | Tests |
|---|---|---|
| Kill a pod/instance | Chaos Monkey, LitmusChaos | Failover speed, error rate |
| Throttle CPU/memory | Stress-ng, LitmusChaos | Degradation under resource pressure |
| Introduce network latency | tc netem, Chaos Mesh | Timeout handling, retry logic |
| Kill the database | LitmusChaos | Circuit breaker, fallback |
| Saturate disk I/O | fio, LitmusChaos | Log rotation, disk-full handling |

```bash
# Inject 100ms network latency + 10% packet loss to a pod
kubectl exec -it chaos-pod -- \
  tc qdisc add dev eth0 root netem delay 100ms loss 10%
```

**Always combine chaos with a baseline load** — run at 50% of expected peak so the system has load to respond to when faults occur.

---

## Pre-Production Performance Gates

### Gate Criteria (must pass before production deploy)

```
1. Smoke test (1 VU):           Zero errors, correct responses
2. Load test (target VUs):      p95 < SLA, error rate < 1%
3. Stress test (1.5× target):   Graceful degradation, no data corruption
4. Regression comparison:       p95 within 10% of previous passing run
```

### Exemptions and Escalation
- If a test fails, categorize: **known regression** (documented, fix tracked) vs **unexpected regression** (block deploy).
- SLA breach of < 5% may be acceptable with product owner sign-off.
- Emergency hotfixes may skip stress test but must pass smoke + load.

---

## Safety Controls for Production Testing

### Mandatory Safety Controls

| Control | Implementation |
|---|---|
| **Blast radius limit** | Never inject more than 10–20% of prod capacity without approval |
| **Kill switch** | A single command / button to stop all injectors immediately |
| **Auto-abort on error spike** | Test stops automatically if error rate exceeds threshold |
| **Rollback plan** | Documented and tested rollback procedure ready |
| **Communication** | Ops/SRE on standby; incident channel open |
| **Synthetic user tagging** | All test requests tagged to exclude from real user metrics |
| **No real user data** | Synthetic test data only — never process real PII in load tests |

### Auto-abort configuration

```javascript
// k6: auto-abort on error spike
export const options = {
  thresholds: {
    http_req_failed: [{
      threshold: 'rate<0.05',  // Abort if error rate > 5%
      abortOnFail: true,
      delayAbortEval: '30s',   // Wait 30s before aborting (avoid false positives)
    }],
  },
};
```

```bash
# JMeter: stop test on error rate threshold via Backend Listener + custom alerting
# Or use Taurus wrapper:
bzt test.jmx \
  -o modules.passfail.checks[0].subject=fail \
  -o modules.passfail.checks[0].threshold=5% \
  -o modules.passfail.checks[0].condition=over \
  -o modules.passfail.checks[0].timeframe=60s \
  -o modules.passfail.checks[0].stop=true
```

---

## Environments Progression

```
Developer Laptop (smoke, 1 VU)
    ↓
CI Environment (smoke + light load, 10–25 VUs, fast feedback)
    ↓
Staging / Performance Environment (full load + stress tests)
    ↓
Pre-Production / Mirror (shadow tests, canary validation)
    ↓
Production (synthetic monitoring, canary, controlled injection)
```

Each environment adds fidelity; the production stage assumes all previous stages passed.

---

## Incident Response During a Performance Test

If something breaks during a test:

1. **Immediately stop the test** (kill switch — stop all injectors).
2. **Capture current state** — snapshot metrics, take thread dumps (Java), capture logs.
3. **Assess impact** — are real users affected? Is the failure isolated to test traffic?
4. **Roll back if needed** — restore previous version, restart services.
5. **Preserve evidence** — don't restart services without capturing logs and heap dumps.
6. **Post-mortem** — document what broke, at what load, what the root cause was.
