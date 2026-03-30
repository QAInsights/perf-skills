# Test Execution

Covers how to run performance tests reliably — locally, in CI/CD pipelines, distributed across injectors, and in the cloud.

---

## Execution Modes

| Mode | When to Use | Tooling |
|---|---|---|
| Local (single process) | Development, debugging, smoke tests | All tools |
| Distributed (multiple agents) | > 500–1000 VUs, high RPS | JMeter, Locust, k6 |
| Cloud (managed) | Production-scale, geographic distribution | Grafana Cloud, BlazeMeter, OctoPerf, Gatling Enterprise, NeoLoad Cloud |
| CI/CD (automated) | Regression gate, scheduled runs | All tools via CLI |

---

## Local Execution Best Practices

- Always use **non-GUI / headless mode** for actual load runs (GUI mode adds overhead).
- Run scripts from the same network segment as the system under test when possible.
- Monitor injector machine resources during the test — if CPU/memory of the injector saturates, results are invalid.
- Set JVM heap appropriately for JMeter: `JVM_ARGS="-Xms2g -Xmx4g" jmeter -n -t test.jmx`

---

## Distributed Execution

### JMeter Distributed

**Architecture:**
```
Controller (orchestrate)  →  Injector 1 (1/N VUs)
                          →  Injector 2 (1/N VUs)
                          →  Injector 3 (1/N VUs)
```

**Setup:**
```bash
# On each injector
./bin/jmeter-server -Djava.rmi.server.hostname=<injector-ip>

# On controller (bin/jmeter.properties)
remote_hosts=injector-1:1099,injector-2:1099,injector-3:1099

# Run (controller distributes test plan and aggregates results)
jmeter -n -t test.jmx -r -l results.jtl -Jthreads=300 -Jduration=600
```

**Ports to open:** 1099 (RMI control), 50000+ (dynamic data ports) — firewall rules critical.

**Scaling:** Each injector can typically handle 300–500 VUs for HTTP at moderate response times. For 10k VUs, plan 20–30 injectors.

### k6 Distributed

```bash
# k6 Operator on Kubernetes (recommended for large-scale k6)
# 1. Install k6 Operator
kubectl apply -f https://github.com/grafana/k6-operator/releases/latest/download/bundle.yaml

# 2. Create TestRun resource
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: k6-load-test
spec:
  parallelism: 5         # 5 pods, each running part of the test
  script:
    configMap:
      name: k6-test-script
      file: test.js
  arguments: --out influxdb=http://influx:8086/k6
```

### Locust Distributed

```bash
# Master (web UI on port 8089)
locust -f locustfile.py --master --expect-workers=4

# Workers (connect to master)
locust -f locustfile.py --worker --master-host=<master-ip>

# Headless distributed
locust -f locustfile.py --master --headless \
  --users 2000 --spawn-rate 50 --run-time 10m \
  --expect-workers=4 --html=report.html
```

---

## Cloud Execution

### k6 → Grafana Cloud
```bash
# Authenticate
k6 login cloud --token $K6_CLOUD_TOKEN

# Run in cloud
k6 cloud --vus 1000 --duration 10m script.js

# Or tag the project
k6 cloud --project-id $PROJECT_ID script.js
```

### JMeter → BlazeMeter / OctoPerf
```bash
# BlazeMeter CLI
bzt tests/load-test.jmx \
  -o modules.blazemeter.token=$BZ_TOKEN \
  -o modules.blazemeter.project="MyApp Load Test" \
  -o modules.blazemeter.concurrency=1000 \
  -o modules.blazemeter.duration=600
```

### AWS (DIY Cloud Execution)
```bash
# Spin up EC2 injectors via Terraform, then run JMeter/k6
# Use Auto Scaling Groups for burst capacity
# Use S3 to store results, CloudWatch for metrics

# Cost optimization: use Spot instances for injectors (price vs reliability tradeoff)
```

---

## CI/CD Integration Patterns

### Performance Gate Pattern
```
Code PR  →  Build  →  Unit/Integration Tests  →  [Performance Gate]  →  Deploy
                                                         │
                                                         ├─ Run smoke (1 VU)
                                                         ├─ Run load test (target VUs)
                                                         ├─ Check SLA thresholds
                                                         └─ PASS → Deploy / FAIL → Block
```

### GitHub Actions — k6

```yaml
name: Performance Gate

on:
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * *'  # Nightly soak test

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Start Application (staging)
        run: docker-compose up -d
        
      - name: Wait for readiness
        run: |
          timeout 60 sh -c 'until curl -sf http://localhost:8080/health; do sleep 2; done'

      - name: Run k6 Load Test
        uses: grafana/k6-action@v0.3.1
        with:
          filename: tests/load/main.js
          flags: --out json=results/results.json
        env:
          BASE_URL: http://localhost:8080
          TARGET_VUS: 50
          DURATION: 5m

      - name: Upload Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: k6-results-${{ github.run_id }}
          path: results/

      - name: Check SLA Thresholds
        run: |
          # k6 already exits non-zero if thresholds fail
          # For additional reporting, parse results.json here
```

### GitHub Actions — JMeter

```yaml
      - name: Run JMeter Tests
        run: |
          jmeter -n \
            -t tests/load.jmx \
            -l results/results.jtl \
            -e -o results/dashboard \
            -Jbase_url=${{ vars.STAGING_URL }} \
            -Jthreads=100 \
            -Jduration=300
          
      - name: Check Error Rate
        run: |
          python3 scripts/check_jtl.py results/results.jtl \
            --max-error-rate 1.0 \
            --max-p95 500
```

### GitLab CI

```yaml
performance-test:
  stage: performance
  image: grafana/k6:latest
  script:
    - k6 run --vus $VUS --duration $DURATION tests/load.js
  variables:
    VUS: "100"
    DURATION: "5m"
    BASE_URL: $STAGING_URL
  artifacts:
    reports:
      performance: results.json
    paths:
      - results.json
  only:
    - main
    - /^release\/.*$/
```

### Jenkins Pipeline

```groovy
stage('Performance Test') {
    steps {
        sh '''
            k6 run \
              --out json=results.json \
              -e BASE_URL=${STAGING_URL} \
              tests/load.js
        '''
    }
    post {
        always {
            archiveArtifacts artifacts: 'results.json'
            perfReport 'results.json'  // Jenkins Performance Plugin
        }
    }
}
```

---

## Environment Isolation for Tests

### Dedicated Staging Environment
- Mirror production architecture (same instance types, same DB configuration).
- No shared services with other teams during test window.
- Disable rate limiting or coordinate with the team running the test.

### Test Window Management
- Book a test window and notify all stakeholders.
- Disable non-essential background jobs that would skew results.
- Pre-warm caches if testing steady-state behavior (not cold-start).

### Infrastructure Monitoring During Test
Always monitor the injector resources alongside the SUT:
- CPU, memory, network I/O on each injector.
- If injector CPU hits 80%+, the injector is the bottleneck, not the SUT.

---

## Test Execution Checklist

- [ ] Non-GUI / headless mode configured
- [ ] Injector capacity verified (CPU, network bandwidth, thread limit)
- [ ] Data files deployed to all injector nodes
- [ ] SUT environment isolated and confirmed healthy
- [ ] APM/observability hooks active
- [ ] SLA thresholds configured for CI pass/fail gate
- [ ] Results output path configured (JTL, JSON, InfluxDB)
- [ ] Smoke test (1 VU) passed before full run
- [ ] Team notified of test window
- [ ] Rollback plan in place (especially for production testing)
