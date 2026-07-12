# Artillery Reference

> Targets: Artillery v2.x (YAML, JS, and TS test definitions; `http`, `playwright`, `socketio`, `ws` engines)

Artillery is a developer-centric, open-source load testing tool that runs on Node.js. Tests can be written in YAML, JavaScript, or TypeScript, and it scales from a single laptop to distributed AWS Lambda / Fargate runs (Artillery Cloud). It is the closest JS/TS-native alternative to k6 and is a strong fit for teams already in the Node ecosystem.

---

## Core Concepts

| Concept | Description |
|---|---|
| **Virtual User (VU)** | A single simulated user executing a `flow` |
| **Arrival (open model)** | New VUs generated per second - the default load model |
| **Phase** | A timed load segment (`arrivalRate`, `rampTo`, `arrivalCount`, `pause`) |
| **Scenario** | Named user journey; a `flow` of requests/actions |
| **Flow** | Ordered list of actions (request, `think`, `capture`, `loop`, `function`) |
| **Capture** | Extract a dynamic value from a response for later reuse (correlation) |
| **Processor** | Custom JS/TS module supplying hooks and metric logic |
| **`ensure`** | SLO/assertion plugin - FAILS the run (non-zero exit) if breached |
| **Environment** | Named config profile switched with `-e` |

> Artillery uses an **open (arrival-rate) load model by default**. `arrivalRate` is *new users per second*, NOT concurrent users. Use `maxVusers` to cap real concurrency. For closed/concurrency modeling, see `../topics/workload-design.md`.

---

## Script Structure (YAML)

```yaml
config:
  target: 'https://staging.example.com'
  phases:
    - duration: '2m'
      arrivalRate: 10
      rampTo: 50
      name: ramp-up
    - duration: '5m'
      arrivalRate: 50
      maxVusers: 200
      name: sustain
  ensure:
    thresholds:
      - 'http.response_time.p95': 500
      - 'http.response_time.p99': 1000
    conditions:
      - expression: 'http.codes.5xx < http.codes.2xx * 0.01'  # <1% 5xx
  processor: './helpers.js'

scenarios:
  - name: 'Browse + Checkout'
    weight: 1
    flow:
      - post:
          url: '/auth'
          json:
            username: '{{ username }}'
            password: '{{ password }}'
          capture:
            - json: '$.id_token'
              as: token
      - think: 2
      - get:
          url: '/products'
          headers:
            authorization: 'Bearer {{ token }}'
      - post:
          url: '/checkout'
          json:
            itemId: '{{ $uuid }}'
          capture:
            - json: '$.orderId'
              as: orderId
      - get:
          url: '/orders/{{ orderId }}'
          headers:
            authorization: 'Bearer {{ token }}'
```

---

## Script Structure (JS / TS)

```javascript
export const config = {
  target: 'https://staging.example.com',
  phases: [
    { duration: '2m', arrivalRate: 10, rampTo: 50, name: 'ramp-up' },
    { duration: '5m', arrivalRate: 50, maxVusers: 200, name: 'sustain' },
  ],
  ensure: {
    thresholds: [
      'http.response_time.p95: 500',
      'http.response_time.p99: 1000',
    ],
  },
  processor: './helpers.js',
};

export const scenarios = [
  {
    name: 'Browse + Checkout',
    flow: [
      {
        post: {
          url: '/auth',
          json: { username: '{{ username }}', password: '{{ password }}' },
          capture: [{ json: '$.id_token', as: 'token' }],
        },
      },
      { think: 2 },
      {
        get: {
          url: '/products',
          headers: { authorization: 'Bearer {{ token }}' },
        },
      },
    ],
  },
];
```

---

## Load Phases

`config.phases` is an array executed sequentially. Four phase kinds:

| Phase kind | Key options | Use case |
|---|---|---|
| **Constant arrival** | `arrivalRate` | Steady RPS-style load (open model) |
| **Ramp** | `arrivalRate` + `rampTo` (both over `duration`) | Warm-up / ramp-up |
| **Fixed count** | `arrivalCount` | Exact N total users spread over `duration` |
| **Pause** | `pause` | Idle gap (soak cool-down, between spikes) |

```yaml
phases:
  - duration: '30m'
    arrivalRate: 1
    rampTo: 100
    name: ramp-up
  - duration: '3h'
    arrivalRate: 100
    name: sustain          # soak/endurance
  - duration: '1m'
    arrivalRate: 500
    name: spike            # spike test
  - pause: 60
```

- `duration` / `pause` accept human-readable units (`'5m'`, `'3h'`) as well as seconds.
- `maxVusers` caps in-flight VUs for any phase - essential to bound concurrency on slow servers (open-model load otherwise queues unbounded pending VUs).
- `name` makes phases identifiable in CLI output and Artillery Cloud.

---

## Correlation (Capture / Dynamic Values)

Use `capture` on a request to extract a value for later steps. Requires `as` and one extractor:

| Extractor | Syntax | Example |
|---|---|---|
| JSONPath | `json: '$.path'` | `json: '$.id_token'` |
| XPath | `xpath: '//node/text()'` | SOAP / XML bodies |
| Regex | `regexp: 'pattern'`, optional `group`, `flags` | `regexp: 'sid=([^&]+)'` |
| Header | `header: 'X-Custom'` | `header: 'Set-Cookie'` |
| Selector | `selector: 'a.product'`, `attr`, `index` | HTML scraping |

```yaml
- get:
    url: '/login'
    capture:
      - json: '$.csrf'
        as: csrf
      - header: 'set-cookie'
        as: cookie
- post:
    url: '/submit'
    headers:
      x-csrf-token: '{{ csrf }}'
    cookie:
      session: '{{ cookie }}'
```

- **Captures are strict by default**: a failed capture stops that VU. Set `strict: false` only when a later request can safely 404.
- For multi-step journeys, capture once near the top and reuse via `{{ var }}` in every later request.
- Capture multiple values from one response with an array of capture specs.

> For framework-specific extraction rules (ASP.NET ViewState, JSF, JWT, SAP `sap-contextid`), see `../topics/correlation.md`.

---

## Parameterization

### CSV payload (`config.payload`)
```yaml
config:
  payload:
    path: 'users.csv'
    fields: ['username', 'password']
    skipHeader: true
    order: sequence        # 'random' (default) | 'sequence'
scenarios:
  - flow:
      - post:
          url: '/auth'
          json:
            username: '{{ username }}'
            password: '{{ password }}'
```
- `order: sequence` is deterministic but **breaks under distributed runs** (each worker has its own copy). Use `random` (default) for distributed tests.
- `loadAll: true` + `name` exposes the whole dataset to each VU for `loop`.
- `cast: false` keeps values as strings; `delimiter` overrides the comma.

### Inline variables (`config.variables`)
```yaml
config:
  variables:
    postcode: ['SE1', 'EC1', 'E8']
    id: ['8731', '9965', '2806']
```
One value is picked at random per VU. Cannot template `config` values.

### Environment variables (`$env`)
```yaml
headers:
  x-api-key: '{{ $env.API_KEY }}'
```
Run with `API_KEY=xxx artillery run script.yml` or `--env-file .env`. Keeps secrets out of source.

### Environments (`-e`)
Reuse one script across dev/staging/prod by defining `config.environments` with per-env `target` and `phases`:
```bash
artillery run -e production script.yml
```
Access the active name via `{{ $environment }}` (e.g. to pick a CSV: `path: '{{ $environment }}-logins.csv'`).

---

## SLO Checks with `ensure` (Assertions)

`ensure` is Artillery's SLA gate - **without it, the run reports metrics but always exits 0**, so CI never fails on latency. Always add it for CI.

```yaml
config:
  plugins:
    ensure:
      thresholds:                 # value must be LESS than this
        - 'http.response_time.p95': 500
        - 'http.response_time.p99': 1000
      conditions:                 # advanced boolean/numeric expressions
        - expression: 'http.response_time.p95 < 500 and http.request_rate > 1000'
        - expression: 'http.codes.5xx <= http.codes.2xx * 0.01'
          strict: false           # optional check; failure won't fail the run
```

- `thresholds` check a metric's aggregate is **below** the integer.
- `conditions` combine metrics with `+ - * / % ^`, comparisons, `and`/`or`/`not`, and `ceil/floor/round`.
- `strict: true` (default) fails the run on breach; `strict: false` reports only.
- Using a non-existent metric name makes that check fail.

### Key metrics for `ensure`
| Metric | Meaning |
|---|---|
| `http.response_time.p95` / `.p99` | Latency percentile (ms) |
| `http.request_rate` | Requests/sec |
| `http.codes.2xx` / `.4xx` / `.5xx` | Status-code counters |
| `http.downloaded_bytes` | Total payload bytes |
| `vusers.completed` / `vusers.failed` | VU outcomes |

> `http.response_time.*` is **TTFB** by default. Enable `config.http.extendedMetrics: true` for full `http.total.*` (download-complete) timing. For SLA baseline tables, see SKILL.md "Threshold Starting Points" and `../topics/results-analysis.md`.

---

## Custom Logic (Processor Hooks)

Load JS/TS via `config.processor`:

```javascript
// helpers.js
module.exports = {
  setApiKey(context, events, done) {
    context.vars.apiKey = process.env.API_KEY;
    return done();
  },
  assertOrder(context, events, done) {
    const status = context.vars.orderStatus;
    if (status !== 'confirmed') {
      events.emit('counter', 'order_failures', 1);
    }
    return done();
  },
};
```

Hook points:
- **`beforeRequest` / `afterResponse`** - set on a request; customize/inspect URL, headers, body.
- **`beforeScenario` / `afterScenario`** - set on a scenario.
- **`function`** step - run arbitrary code mid-flow.

Async hooks are supported (v2.0.7+). Use `events.emit('counter'|'histogram'|'rate', name, value)` for custom metrics.

---

## Output and Observability

| Output | How |
|---|---|
| Terminal summary | Default (`artillery run script.yml`) |
| JSON report | `artillery run -o json=report.json script.yml` |
| Artillery Cloud | `artillery run --record script.yml` (dashboards, historical trends) |
| CSV | `artillery run -o csv=results.csv script.yml` |
| Distributed | AWS Lambda / Fargate workers via `artillery run-fargate` |

Enable `config.http.distributedTracing: true` to attach a W3C `traceparent` header to every request - correlates load with backend spans in your APM.

---

## Artillery-Specific Tips

- **`arrivalRate` is new-users-per-second, not concurrency.** A slow backend makes pending VUs pile up. Always set `maxVusers` to bound real concurrency, or switch to `arrivalCount`/closed-model thinking via `workload-design.md`.
- **Always add `ensure` for CI.** A test with no `ensure` exits 0 regardless of latency or error spikes - it generates traffic but enforces nothing.
- **Captures are strict by default.** A missed extractor aborts the whole VU. Only set `strict: false` when a downstream 404 is acceptable.
- **Add `think` between steps.** Zero think time maximizes RPS unrealistically; use `think` (seconds or `ms` units) to model real pacing.
- **Default `payload.order` is `random`** - deterministic `sequence` ordering does not work correctly in distributed runs.
- **`http.response_time` is TTFB.** Turn on `extendedMetrics` if you need full download time (`http.total.*`).
- **Secrets via `$env` / `--env-file`**, never inline. Use `config.environments` + `-e` to promote the same script dev → staging → prod.
- **Avoid heavy `log` actions under load** - they add overhead; prefer `ensure`/custom counters for visibility.
- **Browser load?** Use the `playwright` engine (`engines: { playwright: {} }`) to drive real pages; note it is far heavier per VU than HTTP.

> For CI/CD integration (GitHub Actions, GitLab CI, distributed execution), see `../topics/test-execution.md`.
> For anti-patterns, assertions, think time, and parameterization principles, see **Key Principles** in `SKILL.md`.
