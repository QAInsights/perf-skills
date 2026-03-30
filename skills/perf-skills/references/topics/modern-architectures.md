# Microservices, Kubernetes & Serverless Performance Testing

Covers performance testing concerns specific to modern distributed architectures — microservices, container orchestration, service meshes, and serverless platforms.

---

## Microservices Performance Testing

Microservices introduce distributed latency, cascading failures, and complex dependency chains that monoliths don't have.

### Key Challenges

| Challenge | Why It Matters |
|---|---|
| **Cascading latency** | A slow downstream service adds latency to every upstream caller |
| **Retry storms** | Aggressive retry policies under load can amplify failures exponentially |
| **Circuit breaker validation** | Circuit breakers must open at the right threshold — too early = false positives, too late = cascading failure |
| **Service mesh overhead** | Sidecar proxies (Envoy/Istio) add 1–5ms per hop — significant in deep call chains |
| **Fan-out amplification** | One API call that fans out to 10 services means 1 RPS at the gateway = 10 RPS backend |
| **Data consistency** | Eventual consistency under load can cause stale reads, lost updates, or duplicate processing |

### Testing Strategy

#### 1. Component-Level Testing (Isolated)
Test each service independently with mocked dependencies to find per-service bottlenecks:
- Max RPS before degradation
- Memory and CPU profile under load
- Connection pool behavior
- Error handling for dependency failures (inject 503s, timeouts)

#### 2. Integration-Level Testing (Chain)
Test the full request path through the service chain:
- End-to-end latency budget (how much latency does each service contribute?)
- Use distributed tracing (Jaeger, Tempo) to identify the slowest hop
- Test with realistic inter-service latency (not localhost-to-localhost)

#### 3. Resilience Testing (Failure Injection)
Combine load testing with chaos engineering:

```javascript
// k6 + chaos: run load while injecting failures
// Step 1: Start load test at 50% target capacity
// Step 2: Kill a downstream service pod
// Step 3: Verify circuit breaker opens within SLA (e.g., < 5s)
// Step 4: Verify error rate stays below threshold
// Step 5: Restore pod, verify recovery
```

### Circuit Breaker Testing

| State | Test Approach |
|---|---|
| **Closed (normal)** | Verify requests pass through with normal latency |
| **Open (tripped)** | Inject failures until breaker opens; verify fallback response |
| **Half-Open (recovery)** | After cooldown, verify breaker allows probe requests and recovers |

Key metrics: time-to-open, fallback response correctness, recovery time.

### Retry Storm Prevention

Test that retry policies don't amplify failure:

```
Scenario: Service B returns 503
  Without backoff: 100 VUs × 3 retries = 300 RPS hitting an already-failing service
  With exponential backoff + jitter: load dissipates over time

Test approach:
1. Run at target load
2. Inject 503s from a dependency
3. Monitor total RPS to that dependency — it should NOT multiply
4. Verify backoff and jitter are working
```

---

## Kubernetes Performance Testing

### Pod Autoscaling (HPA) Validation

The Horizontal Pod Autoscaler should scale pods in response to load — test that it works correctly.

```bash
# Monitor HPA during test
kubectl get hpa -w

# Expected behavior during load test:
# 1. Load increases → CPU/memory rises above target
# 2. HPA triggers scale-up (observe REPLICAS column increasing)
# 3. New pods start and become ready
# 4. Load distributes across new pods
# 5. Latency stabilizes at acceptable levels
```

### What to Test

| Concern | Test Approach |
|---|---|
| **Scale-up speed** | Time from load increase to pods ready and serving traffic |
| **Scale-down behavior** | After load drops, verify pods scale down without disrupting active requests |
| **Pod startup latency** | Time from pod scheduled to first successful health check — critical for burst traffic |
| **Resource limits impact** | Test with and without CPU/memory limits to find optimal settings |
| **Pod disruption budget** | During rolling deploys under load, verify PDB prevents downtime |
| **Node scaling (Cluster Autoscaler)** | If pods can't schedule, verify new nodes provision in time |

### Resource Limits and Throttling

CPU limits cause CFS throttling — the kernel pauses the container when it exceeds its CPU quota, causing latency spikes.

```yaml
# Common mistake: setting CPU limit too close to request
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    cpu: "500m"      # Too tight — causes throttling under burst
    memory: "1Gi"

# Better: allow burst headroom or remove CPU limit
resources:
  requests:
    cpu: "500m"
    memory: "512Mi"
  limits:
    # cpu: omitted — allows burst without throttling
    memory: "1Gi"
```

**Test approach**: Run load test, monitor `container_cpu_cfs_throttled_seconds_total` in Prometheus. If throttling is high, response times will have periodic spikes.

### Service Mesh Overhead (Istio/Envoy)

Sidecar proxies add latency per hop. In a 5-service call chain, that's 10 proxy hops (ingress + egress per service).

**Test approach**:
1. Run load test WITHOUT service mesh → record baseline latency.
2. Enable service mesh → run same load test.
3. Compare: the delta is your mesh overhead.
4. If overhead is too high: tune Envoy concurrency, connection pool settings, or evaluate ambient mesh (Istio ambient mode — no sidecars).

### Ingress Controller Performance

The ingress controller is often the first bottleneck:

| Ingress | Typical Ceiling | Tuning |
|---|---|---|
| NGINX Ingress | ~10k RPS per pod | `worker-processes`, `keepalive-connections`, multiple replicas |
| Envoy / Istio Gateway | ~15k RPS per pod | `concurrency`, circuit breaker settings |
| AWS ALB | Scales automatically | Pre-warm for large tests (request from AWS support) |
| Traefik | ~8k RPS per pod | `maxIdleConnsPerHost`, worker count |

---

## Serverless Performance Testing

Serverless (AWS Lambda, Azure Functions, GCP Cloud Functions) introduces unique latency characteristics that don't exist in traditional deployments.

### Key Challenges

| Challenge | Why It Matters |
|---|---|
| **Cold starts** | First invocation after idle can take 100ms–10s depending on runtime and package size |
| **Concurrency limits** | Account/region-level limits cap how many functions run in parallel |
| **Provisioned concurrency** | Pre-warmed instances eliminate cold starts but cost money — need to test the right amount |
| **Timeout behavior** | Functions have max execution time (Lambda: 15 min) — long-running requests timeout silently |
| **Memory = CPU** | Lambda allocates CPU proportional to memory — 128MB gets less CPU than 1024MB |

### Cold Start Testing

```javascript
// k6: measure cold start latency
// Strategy: invoke function after a known idle period

export const options = {
  scenarios: {
    cold_start: {
      executor: 'per-vu-iterations',
      vus: 1,
      iterations: 1,  // Single invocation after idle = cold start
    },
    warm: {
      executor: 'constant-vus',
      vus: 10,
      duration: '5m',
      startTime: '30s',  // Start after cold start test
    },
  },
  thresholds: {
    'http_req_duration{scenario:cold_start}': ['p(95)<3000'],  // Cold start SLA
    'http_req_duration{scenario:warm}': ['p(95)<200'],          // Warm SLA
  },
};
```

### Concurrency Limit Testing

```
Test approach:
1. Set Lambda reserved concurrency to a known limit (e.g., 100)
2. Ramp k6 VUs to exceed the limit
3. Monitor: invocations should plateau at 100 concurrent
4. Excess requests should get 429 (throttled) responses
5. Verify client-side retry with backoff handles throttling gracefully
```

### Provisioned Concurrency Validation

1. Configure provisioned concurrency (e.g., 50 instances).
2. Run a burst test: 50 VUs simultaneously.
3. Verify: **zero cold starts** — all requests should hit pre-warmed instances.
4. Run 51+ VUs: the 51st should hit a cold start — verify the cold start latency.

### Memory/CPU Tuning Test

Lambda CPU is proportional to memory. Run the same workload at different memory settings:

```
128MB  → p95: 850ms, cost: $0.0001
256MB  → p95: 420ms, cost: $0.00015
512MB  → p95: 210ms, cost: $0.0002
1024MB → p95: 195ms, cost: $0.0004  ← diminishing returns after this
```

Find the memory setting where cost-per-request at target latency is minimized.

---

## Frontend / Browser Performance

For user-facing applications, backend load testing alone is insufficient. Browser-level metrics capture what users actually experience.

### Core Web Vitals

| Metric | What It Measures | Target |
|---|---|---|
| **LCP** (Largest Contentful Paint) | Loading performance | < 2.5s |
| **INP** (Interaction to Next Paint) | Interactivity responsiveness | < 200ms |
| **CLS** (Cumulative Layout Shift) | Visual stability | < 0.1 |

### Tool Support

| Tool | Approach |
|---|---|
| **k6/browser** | Chromium-based browser testing; can measure web vitals under load |
| **Lighthouse CI** | Automated Lighthouse audits in CI pipeline |
| **WebPageTest** | Detailed waterfall analysis from real browsers |
| **Grafana Synthetic Monitoring** | Scheduled browser tests from global probes |

### k6 Browser Example

```javascript
import { browser } from 'k6/browser';
import { check } from 'k6';

export const options = {
  scenarios: {
    browser: {
      executor: 'constant-vus',
      vus: 5,
      duration: '2m',
      options: { browser: { type: 'chromium' } },
    },
  },
  thresholds: {
    browser_web_vital_lcp: ['p(95)<2500'],
    browser_web_vital_cls: ['p(95)<0.1'],
  },
};

export default async function () {
  const page = await browser.newPage();
  try {
    await page.goto(__ENV.BASE_URL);
    await page.waitForSelector('h1');
    check(page, {
      'page loaded': (p) => p.locator('h1').textContent() !== '',
    });
  } finally {
    await page.close();
  }
}
```

### Performance Budget Integration

Define performance budgets in CI to prevent regressions:

```json
{
  "budgets": [
    { "metric": "lcp", "budget": 2500 },
    { "metric": "total-transfer-size", "budget": 500000 },
    { "metric": "script-transfer-size", "budget": 200000 },
    { "metric": "third-party-transfer-size", "budget": 100000 }
  ]
}
```

---

## Modern Architecture Testing Checklist

- [ ] Component-level and integration-level tests defined separately
- [ ] Circuit breaker thresholds tested (open/close/half-open states)
- [ ] Retry policies validated (no retry storms under failure)
- [ ] Service mesh overhead measured and baselined
- [ ] HPA scale-up and scale-down behavior validated under load
- [ ] Pod resource limits tested for CFS throttling impact
- [ ] Ingress controller capacity tested as a potential bottleneck
- [ ] Serverless cold start latency measured and budgeted
- [ ] Concurrency limits and provisioned concurrency validated
- [ ] Frontend Core Web Vitals measured under backend load
- [ ] Performance budgets defined and enforced in CI
