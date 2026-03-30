# Script Generation & Best Practices

This reference covers how to write robust, realistic, and maintainable performance test scripts — regardless of tool. Apply these principles to JMeter, k6, Gatling, Locust, or any other tool.

---

## Scripting Workflow

```
1. Understand the flow       ← Get sequence from BA, Swagger, or HAR
2. Record or scaffold        ← Record via proxy or write from API docs
3. Smoke test (1 VU)         ← Validate correctness before load
4. Correlate dynamic values  ← Session tokens, CSRF, IDs, ViewState
5. Parameterize data         ← Replace hardcoded values with variables
6. Add assertions             ← Validate response correctness
7. Add think time            ← Make VU behavior realistic
8. Add error handling        ← Handle expected and unexpected failures
9. Smoke test again          ← Confirm script still works after changes
10. Ramp up gradually        ← 5 → 10 → 25 → 50 → 100 VUs
```

---

## Recording vs. Coding

### Recording (proxy-based)
- **Pros:** Fast, captures real browser behavior, catches all requests (including hidden calls).
- **Cons:** Captures noise (analytics, CDN, fonts), needs cleanup and correlation.
- **Best for:** Complex web apps, SPAs, unfamiliar APIs.
- **Tools:** JMeter HTTP(S) Test Script Recorder, BlazeMeter Chrome Extension, Gatling Recorder, HAR import (OctoPerf, k6).

### Coding from Scratch
- **Pros:** Clean, no recording noise, easier to version control.
- **Cons:** Slower for complex flows.
- **Best for:** REST/GraphQL APIs with Swagger/OpenAPI docs.

### Hybrid (Recommended)
Record a HAR file from the browser → import into your tool → clean up and parameterize → enhance with dynamic logic.

---

## Correlation Deep Dive

Correlation is extracting a dynamic value from a response and using it in a subsequent request. Missing correlation is the #1 cause of performance script failure.

### Values that always need correlation
- **Session tokens** (JSESSIONID, ASP.NET_SessionId)
- **Auth tokens** (JWT, OAuth access_token, refresh_token)
- **CSRF tokens** (`_token`, `authenticity_token`, `__RequestVerificationToken`)
- **View state** (ASP.NET `__VIEWSTATE`, `__EVENTVALIDATION`)
- **Resource IDs** (created order ID, cart ID, uploaded file ID)
- **One-time codes** (OTP, nonce, challenge)
- **Timestamps/signatures** used in request signing

### Correlation debugging approach
1. Run the script with 1 VU.
2. Look for HTTP 4xx errors (especially 403 Forbidden, 422 Unprocessable Entity).
3. Check the request body/headers — is a token missing or stale?
4. Use the tool's debug output or proxy (Fiddler, Charles) to inspect live traffic.
5. Identify where the value appears in a *previous* response.
6. Add an extractor at that response, reference the variable in the failing request.

---

## Assertions (Why They're Non-Negotiable)

Without assertions, your test might generate 1000 RPS of 404 responses or empty bodies — and your metrics will look "fine."

### What to assert
- **Status code** — the obvious one; but also check for 200s that contain error bodies.
- **Response body** — key field exists and has expected value.
- **Response time** — flag individual responses over SLA as failures.
- **Response size** — detect truncated or empty bodies.
- **Content-Type header** — ensure you got JSON, not an HTML error page.

### Assertion layering

```
Level 1: HTTP status (global, every request)
Level 2: Body content check (per transaction type)
Level 3: Business logic check (e.g., balance is non-negative)
Level 4: Duration assertion (flag outliers per request)
```

### Handling false failures
- Exclude known retryable errors (rate limits, 503 during scale events) from SLA breach calculation.
- Add error handling blocks that retry on specific status codes.
- Log all failures with request/response detail for post-test debugging.

---

## Error Handling

### JMeter
Use **If Controller** + **Retry** or set "Continue" behavior in Thread Group on sampler error.

### k6
```javascript
import { check } from 'k6';
import { sleep } from 'k6';

const res = http.post(url, body, params);
if (res.status === 429) {
    sleep(5); // Back off on rate limit
    return;
}
check(res, { 'status 200': (r) => r.status === 200 });
```

### Gatling
```scala
.exec(http("Submit Order")
  .post("/orders")
  .check(status.in(200, 201, 202))  // Accept multiple valid codes
  .checkIf((response, session) => response.status != 200) {
    jsonPath("$.error").saveAs("errorMessage")
  }
)
```

### Locust
```python
with self.client.post("/orders", json=payload, catch_response=True) as res:
    if res.status_code == 200:
        res.success()
    elif res.status_code == 429:
        res.success()  # Don't count rate limits as failures
        time.sleep(5)
    else:
        res.failure(f"Unexpected {res.status_code}: {res.text[:200]}")
```

---

## Session Management Patterns

### Cookie-based sessions
Most tools handle cookies automatically. Verify the Cookie Manager is enabled.

### Token-based (JWT/OAuth)
1. Authenticate in `setup()` or `on_start()`.
2. Store the token in session variables.
3. Add as Authorization header to subsequent requests.
4. Handle token expiry — check for 401 responses and re-authenticate.

```javascript
// k6: handle token refresh
function getToken() {
    const res = http.post('/auth/token', credentials);
    if (res.status !== 200) throw new Error('Auth failed');
    return res.json('access_token');
}

let token = getToken();

export default function () {
    let res = http.get('/api/profile', { headers: { Authorization: `Bearer ${token}` } });
    if (res.status === 401) {
        token = getToken(); // Refresh token
        res = http.get('/api/profile', { headers: { Authorization: `Bearer ${token}` } });
    }
    check(res, { 'profile 200': (r) => r.status === 200 });
}
```

---

## Naming Conventions

Consistent request naming makes results readable:

```
BAD:  "Request 1", "HTTP Request", "sampler_3"
GOOD: "POST /api/orders", "GET /products/{id}", "Login - Authenticate"
```

Use naming patterns:
- REST APIs: `{METHOD} {path}` e.g., `POST /api/v2/orders`
- Business transactions: `{UserAction} - {Step}` e.g., `Checkout - Submit Payment`
- Background calls: `BG - {description}` e.g., `BG - Poll Status`

For parameterized URLs, use a fixed name:
```javascript
// k6
http.get(`/products/${productId}`, { tags: { name: 'GET /products/{id}' } });
```

---

## Script Structure Best Practices

### Modularize reusable flows
Extract common flows (login, logout, navigation) into separate functions/files. Import them in scenarios.

### Use config files for environment portability
```javascript
// k6 config
const config = {
    baseUrl: __ENV.BASE_URL || 'https://staging.example.com',
    timeout: '30s',
    headers: { 'X-Test-Run': __ENV.BUILD_ID || 'manual' },
};
```

### Version control your scripts
Treat performance scripts as production code:
- Git repository alongside application code
- PR reviews for script changes
- Tag script versions to match application releases
- Store results per tag for trend comparison

### Script review checklist
- [ ] No hardcoded URLs, credentials, or IDs
- [ ] All dynamic values correlated and parameterized
- [ ] Assertions on every request (minimum: status code)
- [ ] Think time applied between transactions
- [ ] Error handling for known failure cases (401, 429, 503)
- [ ] Request naming follows convention
- [ ] Single VU smoke test passes cleanly
- [ ] Script loads test data from external source (CSV, API)
- [ ] Environment-specific values in config/env vars
