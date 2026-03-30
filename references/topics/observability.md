# Observability in Performance Testing

Observability is what bridges "the test shows slow responses" to "the database is saturating its connection pool at 300 VUs." Without correlated server-side metrics, performance test analysis is guesswork.

---

## The Three Pillars

| Pillar | What It Tells You | Tools |
|---|---|---|
| **Metrics** | Numeric measurements over time (CPU, RPS, latency histograms) | Prometheus, InfluxDB, Datadog, CloudWatch |
| **Logs** | Event records with context (error details, request IDs) | Elasticsearch, Loki, Splunk, CloudWatch Logs |
| **Traces** | End-to-end request path across services | Jaeger, Tempo, Datadog APM, Dynatrace |

A complete performance investigation uses all three, correlated by time and request ID.

---

## Minimal Observability Stack for Performance Testing

### Open Source (self-hosted)
```
Load Tool (k6/JMeter)
    │ pushes metrics
    ▼
InfluxDB  ←──────────────────────────────────
    │                                         │
    ▼                                    Prometheus
Grafana Dashboard                            │
    │                              Node Exporter (host metrics)
    ▼                              cAdvisor (container metrics)
Alerting (Slack/PagerDuty)         JMX Exporter (JVM metrics)
```

### Cloud-native
```
Load Tool → Grafana Cloud (k6 native) → Grafana Dashboard
App → Datadog / Dynatrace / New Relic → correlated in single pane
```

---

## Correlating Load Test with APM

The most powerful workflow: overlay load metrics with APM metrics on the same timeline.

**Step 1:** Add a test identifier to all load-tool requests:
```javascript
// k6: tag all requests
export const options = {
  tags: { testRun: __ENV.BUILD_ID || 'manual', env: 'staging' }
};
```
```bash
# JMeter: set custom header on all requests
# In HTTP Header Manager (at Test Plan level):
X-Load-Test-Run: ${__P(build.id,manual)}
X-Load-Test-Phase: ${__P(phase,load)}
```

**Step 2:** In your APM, filter traces and metrics by the test tag — isolate test traffic from organic traffic.

**Step 3:** Align timelines — zoom in on the window when p95 degraded and look at:
- Service CPU and memory during that window
- DB query duration spike
- Thread pool queue depth
- Cache hit rate drop

---

## Prometheus + Grafana Setup

### k6 → Prometheus (remote write)
```bash
k6 run --out experimental-prometheus-rw \
  -e K6_PROMETHEUS_RW_SERVER_URL=http://prometheus:9090/api/v1/write \
  -e K6_PROMETHEUS_RW_TREND_STATS='p(50),p(95),p(99)' \
  script.js
```

### JMeter → InfluxDB
1. Install **Backend Listener** config element.
2. Set implementation: `InfluxdbBackendListenerClient`
3. Config: `influxdbUrl=http://influx:8086`, `measurement=jmeter`, `token=<token>`

### Useful Grafana Dashboards
| Dashboard | Grafana ID | For |
|---|---|---|
| k6 Load Testing Results | 2587 | k6 + InfluxDB |
| JMeter Load Test | 1152 | JMeter + InfluxDB |
| Node Exporter Full | 1860 | Host system metrics |
| JVM Overview | 4701 | Java app JVM metrics |
| Kubernetes Cluster | 7249 | K8s pod metrics |

---

## JVM Metrics (Java Applications)

For Java services under test, always collect JVM metrics alongside load tool metrics.

### Expose via JMX Exporter (Prometheus)
```yaml
# jmx_config.yaml
rules:
  - pattern: java.lang<type=Memory><HeapMemoryUsage>used
    name: jvm_heap_used_bytes
  - pattern: java.lang<type=GarbageCollector, name=(.+)><CollectionTime>
    name: jvm_gc_collection_seconds_total
    labels:
      gc: $1
```

### Key JVM Metrics to Monitor
| Metric | Normal | Warning |
|---|---|---|
| Heap used % | < 70% | > 85% = GC pressure |
| GC pause duration (p99) | < 200ms | > 500ms = impacting response time |
| GC frequency | Occasional | Continuous = memory leak |
| Thread pool queue size | 0–10 | Growing = throughput ceiling reached |
| JDBC pool active connections | < 80% | > 90% = connection starvation |

---

## Log Correlation

During a performance test, log volume explodes. Use structured logging and filtering:

### Mark test traffic in logs
```java
// Spring Boot: use MDC to add test context
MDC.put("testRun", request.getHeader("X-Load-Test-Run"));
// Log4j/Logback will include this in every log line for this thread
```

### Useful log queries during test (Elasticsearch/Kibana)
```
# Error spike investigation
testRun: "build-42" AND level: ERROR
| group by logger, message
| order by count desc

# Slow transactions
testRun: "build-42" AND duration_ms: >500
| group by endpoint
| percentile(duration_ms, 95)
```

---

## Distributed Tracing

Traces show the end-to-end path of a single request across microservices — invaluable for pinpointing *which service* in a chain is slow.

### Setup for load testing
- Confirm tracing is active in the target environment.
- After the test, sample traces from the slowest requests (p99 window).
- Use the trace waterfall to find which service/DB call accounts for the most latency.

### Adding trace context to load tool requests
```javascript
// k6: pass trace context (Jaeger B3 format)
const traceId = randomString(32, '0123456789abcdef');
http.get(url, {
    headers: {
        'X-B3-TraceId': traceId,
        'X-B3-SpanId': randomString(16, '0123456789abcdef'),
        'X-B3-Sampled': '1',
        'X-Load-Test-Run': __ENV.BUILD_ID
    }
});
```

---

## Synthetic Monitoring (Post-Test)

After a load test validates a build, set up **synthetic monitors** to continuously test a minimal user journey in production:

| Tool | Description |
|---|---|
| Grafana Synthetic Monitoring | k6 scripts run on schedule from cloud probes |
| Datadog Synthetics | Browser + API tests from global locations |
| AWS CloudWatch Synthetics (Canaries) | Node.js scripts on schedule |
| Checkly | API + Browser checks with k6 integration |

Synthetic monitors catch regressions that slip through staging without requiring another full load test.

---

## Performance Testing Observability Checklist

- [ ] APM tool active and capturing traces in test environment
- [ ] Server metrics (CPU, memory, disk I/O) dashboards ready before test start
- [ ] JVM metrics exposed (for Java services)
- [ ] DB slow query logging enabled and queryable
- [ ] Load tool pushing real-time metrics to dashboard (InfluxDB, Grafana Cloud, etc.)
- [ ] Test run tagged with build ID / run ID for filtering
- [ ] Alert thresholds set (so team is notified if something breaks during test)
- [ ] Log aggregation active and searchable
- [ ] Baseline metrics screenshot taken before test starts (for comparison)
- [ ] Metrics retention configured to keep results for trend analysis
