# JMeter Reference

> Targets: JMeter 5.6+, Plugins Manager 1.10+

Apache JMeter is the most widely used open-source performance testing tool. It supports a broad range of protocols and is extended through a rich plugin ecosystem.

---

## Core Concepts

| Concept | Description |
|---|---|
| **Test Plan** | Root container — the `.jmx` file |
| **Thread Group** | Defines VU count, ramp-up, and loop count |
| **Sampler** | Makes a request (HTTP, JDBC, JMS, TCP, etc.) |
| **Controller** | Logic: Loop, If, While, Throughput |
| **Config Element** | HTTP Defaults, CSV Data Set, Cache Manager |
| **Pre/Post Processor** | Run before/after a sampler (BeanShell, JSR223) |
| **Assertion** | Validates the response (Response Code, Duration, JSON) |
| **Listener** | Collects and displays results (View Results Tree, Summary) |
| **Timer** | Adds think time (Constant, Gaussian, Synchronizing) |

---

## Thread Group Configuration Patterns

### Standard load ramp
```
Thread Group:
  Number of Threads (users): 100
  Ramp-Up Period (seconds):   120   ← 1 VU every 1.2s, avoids thundering herd
  Loop Count:                 -1    ← run forever until duration
  Duration:                   600   ← 10 minutes
  Startup Delay:              0
```

### Stepping Thread Group (requires JMeter Plugins)
Use `jp@gc - Stepping Thread Group` for staged load:
- Start 10 VUs, hold 60s → add 10 every 30s → max 100 VUs → hold 300s → ramp down

### Concurrency Thread Group (preferred for modern tests)
Use `jp@gc - Concurrency Thread Group` for target-concurrency model:
- Maintains a target concurrency level dynamically, compensates for slow VUs

---

## HTTP Sampler Best Practices

- Always use **HTTP Request Defaults** config element for base URL/port/protocol — never hardcode in individual samplers.
- Set **Content-Type** header in an HTTP Header Manager at Thread Group level, not per sampler.
- Use **KeepAlive** (default on) for realistic connection reuse.
- Use **Follow Redirects** only when your app requires it; disable otherwise for accuracy.
- Avoid using **View Results Tree** listener in load tests — it's memory-intensive; use it only during script development.

---

## Correlation: Extracting Dynamic Values

Correlation is the #1 cause of script failures in session-heavy apps.

### Regular Expression Extractor
```
Reference Name:    csrf_token
Regular Expression: name="_token" value="(.+?)"
Template:          $1$
Match No.:         1
Default Value:     NOT_FOUND
```

### JSON Extractor (REST APIs)
```
Reference Name:    access_token
JSON Path:         $.data.token
Match No.:         1
Default Value:     NOT_FOUND
```

### Boundary Extractor (fastest, no regex overhead)
```
Reference Name:    session_id
Left Boundary:     "sessionId":"
Right Boundary:    "
```

### Using Extracted Values
Reference with `${variable_name}` in subsequent requests.

Always add an assertion on the extracted value:
```
Response Assertion → Variable: ${csrf_token} → Pattern: NOT_FOUND → Negate
```

---

## CSV Data Set Config

```
Filename:           ${__P(data.dir,./data)}/users.csv
Variable Names:     username,password,account_id
Delimiter:          ,
Allow Quoted Data:  true
Recycle on EOF:     true
Stop Thread on EOF: false
Sharing Mode:       All Threads  ← or "Current Thread Group" if isolated data needed
```

**Tips:**
- Use `${__P(data.dir,...)}` property for portable paths across environments.
- For high VU counts, ensure the CSV has at least as many rows as peak VU count to avoid data collisions.
- In distributed tests, the CSV file must exist on **each injector node**, not just the controller.

---

## JSR223 Scripting (Groovy)

Always use **JSR223** over BeanShell — Groovy is compiled and cached, BeanShell is not.

### Pre-processor: Generate a dynamic timestamp
```groovy
import java.time.Instant
vars.put("timestamp", Instant.now().toEpochMilli().toString())
```

### Post-processor: Parse JSON response
```groovy
import groovy.json.JsonSlurper
def json = new JsonSlurper().parseText(prev.getResponseDataAsString())
vars.put("userId", json.data.id.toString())
```

### Pre-processor: Compute HMAC signature
```groovy
import javax.crypto.Mac
import javax.crypto.spec.SecretKeySpec
def key   = vars.get("api_secret")
def data  = vars.get("request_body")
def mac   = Mac.getInstance("HmacSHA256")
mac.init(new SecretKeySpec(key.bytes, "HmacSHA256"))
vars.put("signature", mac.doFinal(data.bytes).encodeHex().toString())
```

---

## Assertions

### Response Code Assertion
```
Field to Test:  Response Code
Pattern:        200
```

### JSON Assertion
```
JSON Path:      $.status
Expected Value: success
```

### Duration Assertion
```
Duration to assert: 2000   ← flag any response > 2000ms
```

### Response Size Assertion
Use to detect incomplete responses — flag if body < 100 bytes unexpectedly.

**Important:** Add assertions to the **transaction controller level** where possible, not individual samplers, to get meaningful business-transaction-level validation.

---

## Transaction Controllers

Wrap related HTTP samplers in Transaction Controllers to measure end-to-end business transaction time:

```
Transaction Controller: Login Flow
  ├── HTTP Sampler: GET /login
  ├── HTTP Sampler: POST /authenticate
  └── HTTP Sampler: GET /dashboard
```

Set **Generate Parent Sample = true** to log only the aggregate transaction (not individual child requests) in results.

---

## Timers (Think Time)

| Timer | Use Case |
|---|---|
| Constant Timer | Simple fixed think time |
| Uniform Random Timer | Range between min–max |
| Gaussian Random Timer | Bell-curve distribution — most realistic |
| Throughput Shaping Timer | Target a specific RPS regardless of VU count |

Gaussian formula: `delay = Constant + Gaussian(deviation)`
Realistic web-app think time: Gaussian with constant=3000ms, deviation=1500ms.

---

## Running JMeter Non-GUI (Command Line)

> For distributed testing architecture and setup, see `references/topics/test-execution.md`.

Always run load tests in non-GUI mode:
```bash
jmeter -n \
  -t test.jmx \
  -l results/results.jtl \
  -e -o results/dashboard \
  -Jenv=staging \
  -Jthreads=100 \
  -Jduration=600
```

| Flag | Purpose |
|---|---|
| `-n` | Non-GUI mode |
| `-t` | Test plan path |
| `-l` | JTL results file |
| `-e -o` | Generate HTML dashboard after run |
| `-J` | Set JMeter property (use `${__P(env,dev)}` in plan) |
| `-G` | Set global property (available to remote engines) |

---

## Key Plugins (JMeter Plugins Manager)

| Plugin | Purpose |
|---|---|
| `Concurrency Thread Group` | Maintain target concurrency |
| `Stepping Thread Group` | Step-ramp load profile |
| `Throughput Shaping Timer` | Control RPS exactly |
| `PerfMon` | Collect server-side CPU/memory via agent |
| `3 Basic Graphs` | Lightweight real-time charting |
| `JDBC Connection Configuration` | Database load testing |
| `WebSocket Sampler` | WS protocol support |
| `gRPC Sampler` | gRPC protocol support |

Install plugins via: **Options → Plugins Manager → Available Plugins**

---

## JMeter-Specific Tips

- **Always run in non-GUI mode** for load tests — GUI mode consumes JMeter's own resources and distorts results.
- **Use HTTP Request Defaults** — hardcoded hosts make environment switching painful.
- **Never put listeners inside loops** — View Results Tree in a loop will OOM the JVM.
- **Use JSR223 (Groovy) over BeanShell** — Groovy is compiled and cached; BeanShell is interpreted per invocation.
- **Set JVM heap** for large tests: `JVM_ARGS="-Xms2g -Xmx4g" jmeter -n -t test.jmx`

> For CI/CD integration (Maven, GitHub Actions, GitLab, Jenkins), see `references/topics/test-execution.md`.
> For anti-patterns, assertions, think time, and parameterization principles, see **Key Principles** in `SKILL.md`.
