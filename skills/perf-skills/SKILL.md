---
name: perf-skills
description: "Performance testing guidance for JMeter, k6, Gatling, Locust, NeoLoad, LoadRunner, OctoPerf"
---

# Performance Testing Skill (perf-skills)

This skill provides expert, opinionated guidance across the full performance testing lifecycle — from workload design through production observation. It covers both commercial tools (LoadRunner, NeoLoad, OctoPerf) and open-source tools (JMeter, k6, Gatling, Locust).

---

## How to Use This Skill

Read the relevant reference files based on what the user needs. Multiple files may apply.

### Loading Priority Rules

1. **Tool-specific syntax/config** → load the tool file only.
2. **Strategy/concepts** (workload design, test data, analysis) → load the topic file only.
3. **Both apply** (e.g., "JMeter CI/CD") → load the topic file first for patterns, then the tool file for syntax.
4. **Never load all files at once** — select the 1–2 most relevant.
5. **Cross-cutting principles** (assertions, think time, parameterization) → this file's Key Principles section is the single source of truth.

### Reference Map

| User needs help with... | Read this file |
|---|---|
| Choosing the right tool | This file — see Tool Selection Matrix below |
| JMeter scripts, plugins, config | `references/tools/jmeter.md` |
| k6 scripting, extensions, cloud | `references/tools/k6.md` |
| Gatling simulations, Scala/Java DSL | `references/tools/gatling.md` |
| Locust Python tests, distributed | `references/tools/locust.md` |
| NeoLoad projects, GUI, APIs | `references/tools/neoload.md` |
| LoadRunner scripts, protocols, VuGen | `references/tools/loadrunner.md` |
| OctoPerf cloud test management | `references/tools/octoperf.md` |
| Designing workloads, concurrency, pacing | `references/topics/workload-design.md` |
| Test data, parameterization, CSV feeds | `references/topics/test-data.md` |
| Script patterns, correlation, best practices | `references/topics/script-generation.md` |
| CI/CD, distributed execution, cloud runners | `references/topics/test-execution.md` |
| Analyzing results, percentiles, SLAs | `references/topics/results-analysis.md` |
| APM, metrics, tracing, dashboards | `references/topics/observability.md` |
| Staging vs production testing strategies | `references/topics/production-testing.md` |
| gRPC, GraphQL, WebSocket, messaging protocols | `references/topics/protocol-testing.md` |
| Database load testing (JDBC, connection pools) | `references/topics/database-testing.md` |
| Microservices, K8s, serverless performance | `references/topics/modern-architectures.md` |

### Protocol Routing Table

When the user's question is protocol-specific, use this to select the right tool and reference:

| Protocol | Recommended Tools | Reference |
|---|---|---|
| HTTP / REST | k6, Gatling, JMeter | Tool file |
| gRPC | k6, Gatling, JMeter (plugin) | `references/topics/protocol-testing.md` + tool file |
| GraphQL | k6, Gatling | `references/topics/protocol-testing.md` + tool file |
| WebSocket / SSE | Gatling, k6 | `references/topics/protocol-testing.md` + tool file |
| JDBC / Database | JMeter | `references/topics/database-testing.md` + `jmeter.md` |
| Kafka / Message Queues | k6 (xk6-kafka), JMeter | `references/topics/protocol-testing.md` |
| SOAP / WSDL | LoadRunner, JMeter | Tool file |
| SAP / Citrix | LoadRunner, NeoLoad | Tool file |

---

## Tool Selection Matrix

Use this to recommend the right tool when the user hasn't decided yet.

| Criteria | JMeter | k6 | Gatling | Locust | NeoLoad | LoadRunner | OctoPerf |
|---|---|---|---|---|---|---|---|
| **Language** | GUI/XML + Groovy | JavaScript/TypeScript | Scala/Java | Python | GUI + NeoLoad DSL | VuGen C-like | Web UI (JMeter-based) |
| **Open source** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ (SaaS) |
| **Protocol support** | HTTP, JDBC, JMS, MQTT, FTP, gRPC | HTTP, gRPC, WS | HTTP, JMS, gRPC | HTTP, gRPC | HTTP, gRPC, WS, SAP | HTTP, Citrix, SAP, Flex | HTTP (JMeter-backed) |
| **Developer-friendly** | Medium | High | High | High | Low | Low | Medium |
| **Enterprise support** | Community + BlazeMeter | Grafana Cloud | Gatling Enterprise | Limited | ✅ | ✅ | ✅ |
| **CI/CD integration** | Good (Maven/Gradle) | Excellent | Excellent | Good | Good | Moderate | Good |
| **Cloud execution** | BlazeMeter, OctoPerf | Grafana Cloud | Gatling Enterprise | Self-managed | NeoLoad Cloud | AWS/on-prem | OctoPerf Cloud |
| **Best for** | Legacy systems, JDBC, protocols | Modern APIs, TypeScript devs | High-throughput HTTP | Python teams, flexible | SAP/Citrix enterprise | Mainframe, legacy enterprise | JMeter teams needing cloud UI |

### Quick decision rules
- **Team writes code** → k6 or Gatling
- **Team uses GUI** → JMeter or NeoLoad
- **Python shop** → Locust
- **SAP / mainframe / Citrix** → LoadRunner or NeoLoad
- **Need cloud SaaS with minimal setup** → OctoPerf (JMeter) or Grafana Cloud (k6)
- **Free + protocol variety** → JMeter
- **Correlation needed for session-heavy flows** → JMeter (with Correlation Recorder) or LoadRunner
- **gRPC or GraphQL APIs** → k6 or Gatling
- **Message queues (Kafka, RabbitMQ)** → k6 (xk6-kafka) or JMeter

---

## Performance Testing Lifecycle Overview

Always think through these phases when helping a user — they often ask about one phase but need context from others.

```
1. PLAN
   └─ Workload design → concurrency model → SLA targets → test type selection
       → references/topics/workload-design.md

2. DATA
   └─ Identify variables → parameterization strategy → test data generation
       → references/topics/test-data.md

3. SCRIPT
   └─ Record or code → correlation → parameterization → assertions → script review
       → references/topics/script-generation.md + tool-specific file

4. EXECUTE
   └─ Local → distributed → CI/CD → cloud burst → monitoring hooks
       → references/topics/test-execution.md

5. OBSERVE
   └─ APM → metrics → logs → traces → dashboards
       → references/topics/observability.md

6. ANALYZE
   └─ Throughput, latency percentiles, errors → bottleneck ID → report
       → references/topics/results-analysis.md

7. PRODUCTION
   └─ Canary testing → shadow load → chaos → synthetic monitoring
       → references/topics/production-testing.md
```

---

## Common Performance Test Types

| Test Type | Goal | Key Metric |
|---|---|---|
| **Load** | Validate system at expected load | Response time, throughput, error rate |
| **Stress** | Find the breaking point | Max VUs before degradation, error onset |
| **Soak/Endurance** | Detect memory leaks, slow degradation | Resource trend over time (hours) |
| **Spike** | Behavior under sudden traffic burst | Recovery time, error spike |
| **Capacity** | Find max sustainable load | Throughput ceiling at SLA thresholds |
| **Smoke** | Quick sanity check | Single VU — no errors |
| **Breakpoint** | Incremental ramp until failure | Failure threshold VU count |

---

## Key Principles to Always Apply

1. **Never test against production blindly** — always have a rollback plan and alerting in place.
2. **Baseline first** — always establish a baseline before stress or soak runs.
3. **Think time and pacing matter** — unrealistic zero-think-time tests produce misleading results.
4. **Parameterize everything** — hardcoded credentials, tokens, and IDs will fail at scale.
5. **Assertions are not optional** — tests without assertions are just generating traffic, not validating behavior.
6. **Isolate the system under test** — shared environments invalidate results.
7. **Correlate dynamic values** — session tokens, CSRF, ViewState, etc. must be extracted and reused.

---

## Asking the Right Questions

When a user brings a performance problem, ask (or infer) these before prescribing a solution:

- What is the **target concurrency** (VUs or RPS)?
- What is the **SLA** (e.g., p95 < 500ms, error rate < 1%)?
- What is the **protocol** (HTTP/REST, gRPC, JDBC, WebSocket)?
- Is the app **stateful** (session-based) or **stateless** (token-based)?
- Where will tests **run from** (local, CI, cloud)?
- What **environment** is being tested (dev, staging, prod)?
- Is there an **APM tool** in place (Datadog, Dynatrace, Grafana, New Relic)?
