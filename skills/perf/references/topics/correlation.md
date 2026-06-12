# Dynamic Value Correlation

Correlation is the process of capturing dynamic values (e.g., session IDs, authentication tokens, CSRF tokens, order IDs, state parameters) from server responses and reusing them in subsequent requests. Without proper correlation, load tests will replay stale identifiers, leading to authentication errors, session mismatches, or invalid application states.

Apply these principles to JMeter, k6, Gatling, Locust, LoadRunner, or any other performance testing tool.

---

## 1. The Core Correlation Workflow

Every correlation task follows a four-step lifecycle:

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. IDENTIFY                                                     │
│    Detect dynamic values by comparing diffs or analyzing errors │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. LOCATE & EXTRACT                                             │
│    Find the value in headers/body and write an extraction rule  │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. PARAMETERIZE (INJECT)                                        │
│    Replace hardcoded values in subsequent requests with variable│
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. VERIFY                                                       │
│    Run a single-user sanity check and print variable value      │
└─────────────────────────────────────────────────────────────────┘
```

### What to Correlate vs. What to Skip

Not every value that changes between recordings needs correlation. Over-correlating wastes effort and makes scripts brittle.

**Correlate these** (dynamic per-session or per-request):
* Session identifiers (`JSESSIONID`, `ASP.NET_SessionId`, `PHPSESSID`)
* Anti-forgery / CSRF tokens (`_csrf`, `__RequestVerificationToken`, `authenticity_token`)
* Page state (`__VIEWSTATE`, `javax.faces.ViewState`)
* OAuth/OIDC artifacts (`code`, `state`, `nonce`, `access_token`, `refresh_token`)
* Transaction / order IDs generated server-side
* Timestamps or nonces used in request signing (HMAC, AWS SigV4)

**Do NOT correlate these** (static per-environment or cosmetic):
* Build version strings, Git commit hashes
* Static asset fingerprints (e.g., `main.a3f2b1.js`)
* Server hostnames in headers (`X-Served-By`)
* Analytics/tracking IDs that do not affect application flow
* Feature flag values (unless they gate the flow under test)
* Values that differ only because of test data differences (usernames, emails)

**Rule of thumb**: If removing the value causes a functional failure (HTTP 4xx, redirect to login, business error), it needs correlation. If removing it only affects logging or cosmetics, skip it.

### Chained / Multi-Hop Correlation

Real-world flows often require multi-hop correlation chains where each step depends on a value from a prior step:

```
Login → sessionId → Browse → productId → Add to Cart → cartId → Checkout → orderId → Payment
```

**Rules for chained correlation:**
1. **Map the full chain before scripting.** Draw the dependency graph: which response feeds which request. Missing one link breaks everything downstream.
2. **Fail fast on broken links.** If step 2 fails to extract `productId`, steps 3–5 will all fail with confusing errors. Add assertions/checks after every extraction to verify the variable is non-empty.
3. **Use descriptive default values.** Set defaults like `PRODUCT_ID_NOT_FOUND` (not empty string) so failures are immediately visible in logs.
4. **Consider conditional flow control.** In JMeter, use an If Controller to skip downstream requests when extraction fails. In k6/Gatling/Locust, use early returns or conditional blocks.

---

## 2. Strategies for Identifying Dynamic Values

### Diffing Strategy (Record Twice)
1. **Record the same scenario twice** using identical test data but different sessions/credentials (or simply run it twice consecutively).
2. **Export both recordings** to a text format (e.g., JMeter JMX, HTTP Archive HAR, or k6 scripts).
3. **Compare/Diff the files** using a diff tool.
4. **Flag values that change** (e.g., `session_id`, `state`, `__VIEWSTATE`, `csrf_token`, timestamps). These are prime candidates for correlation.

### Error Analysis Strategy
* **HTTP 401 Unauthorized / HTTP 403 Forbidden**: Check for missing or stale CSRF tokens, JWTs, or session cookies.
* **HTTP 302 Redirecting to Login**: The session token was not propagated correctly; subsequent requests are rejected.
* **HTTP 200 with "Session Expired" or "Invalid Transaction" in body**: The application returns a successful status code but fails at the business logic layer due to uncorrelated transaction/order IDs.

### Unconventional & Advanced Identification Strategies

#### 1. Runtime Hooking (Browser Storage & Network Interception)
* **How it works**: Instead of comparing static recordings, you inject a monkey-patch/hook script into the browser console during your manual recording session (via Tampermonkey, Playwright, or Selenium).
* **Mechanism**: The hook intercepts `window.localStorage.setItem`, `window.sessionStorage.setItem`, and `XMLHttpRequest.send`/`fetch`. Whenever a token is stored or read, the hook logs exactly which response key/value maps to which request header/payload.
* **Benefit**: Pinpoints exactly where the token is stored and how it is used at runtime, eliminating the need to search through hundreds of recorded HTML/JSON responses.

#### 2. Self-Healing Runtime Scavenging
* **How it works**: Write a tool-specific global pre-processor or failure handler (e.g., a JMeter JSR223 Listener or k6 custom response handler).
* **Mechanism**: When a request fails with an HTTP 401/403 (or contains a known error message), the handler automatically scans the response history or the current response body for keywords like `csrf`, `token`, `nonce`, or input fields. It extracts the value on the fly, updates the thread's variable, and retries the failed request.
* **Benefit**: The script continues working even if the developer changes the page layout or login flow.

#### 3. OpenAPI/Swagger Contract Dependency Mapping
* **How it works**: Parse the target application's OpenAPI/Swagger specification JSON/YAML file.
* **Mechanism**: Use a script to build a dependency graph of all endpoints. Look for endpoints where Response Schemas define a field (e.g., `id` or `token`) that matches the Request Schema parameter name of a subsequent endpoint.
* **Benefit**: Discovers potential correlation points *before* recording a single request.

#### 4. Semantic LLM Extraction (Hybrid Execution)
* **How it works**: During script maintenance or runtime validation, feed the raw response headers or HTML directly to an LLM agent.
* **Mechanism**: Ask the model to "Locate the dynamic CSRF or session identifier in this response and output the JSONPath/CSS Selector/Regex."
* **Benefit**: Excellent for dealing with structurally unstable layouts or highly obfuscated legacy web forms.

#### 5. SPA / Client-Side Rendered App Detection
* **How it works**: Modern SPAs (React, Angular, Vue, Next.js) embed dynamic values in JavaScript rather than HTML forms.
* **Where to look**:
  - `window.__INITIAL_STATE__` or `window.__NEXT_DATA__` inside `<script>` tags (hydration data)
  - JSON payloads in `<script type="application/json">` elements
  - Response headers from XHR/fetch API calls (not the initial HTML page load)
  - `localStorage` / `sessionStorage` writes from initialization scripts
* **Extraction approach**: Regex on the raw response body targeting the JS object (e.g., `__NEXT_DATA__\s*=\s*({.+?})\s*</script>`), then parse the extracted JSON with a JSONPath extractor as a second step.
* **Pitfall**: The initial page HTML may not contain the token at all - it arrives in a subsequent XHR call. Record network traffic at the API level, not just page loads.

---

## 3. Extractor Selection Guide

**Golden rule: if a structured extractor exists for the response format, use it instead of regex.** JSONPath for JSON, CSS Selectors for HTML, Header Extractor for headers. Regex should be the last resort - used only for unstructured text, mixed formats, or inline JavaScript.

| Response Format | Extractor Type | Best For | Brittle/Fragile For |
| :--- | :--- | :--- | :--- |
| **JSON** | **JSONPath / jq** | REST APIs, microservices, token payloads | Structurally fluid JSON (e.g., dynamic key names) |
| **HTML / XML** | **CSS Selectors / XPath** | Web pages, forms, SOAP, legacy XML APIs | Brittle if using deep absolute paths (e.g., `/html/body/div[2]/div/form`) |
| **HTTP Headers** | **Regex / Header Extractor** | `Location`, `Set-Cookie`, custom headers | Header order changes |
| **GraphQL** | **JSONPath on `data` envelope** | `{"data":{...}}` responses | Deeply nested or aliased queries - use the alias name in the path, not positional indices |
| **Unstructured / Mixed** | **Regular Expression** | Quick captures, custom patterns | Overly broad matching, changing HTML structure |
| **Plain Text** | **Boundary Extractor** | Delimited strings (e.g., `id="123"`) | When left/right boundaries are not unique |

### Cookie-Based Correlation

Most tools provide automatic cookie management (JMeter's HTTP Cookie Manager, k6's cookie jar, Gatling's cookie store, Locust's `requests.Session`). **Rely on the cookie manager by default** - manual cookie extraction is only needed when:

* **Multi-domain SSO**: Cookies set on `auth.example.com` need to be forwarded to `app.example.com`. Cookie managers scope by domain and will not transfer automatically.
* **SameSite / Secure flags**: Cookies with `SameSite=Strict` or `Secure` may not attach on cross-origin or non-HTTPS requests during testing.
* **Cookie path mismatch**: A cookie scoped to `/api/v2` will not attach to `/api/v3` requests.
* **Non-cookie transport**: Some frameworks return session IDs in response bodies or `Location` headers instead of `Set-Cookie` (e.g., URL-rewritten `JSESSIONID`).

**Debugging**: In JMeter, add a Debug Sampler to view `CookieManager` contents. In k6, inspect `http.cookieJar().cookiesForURL(url)`. In Locust, log `self.client.cookies.get_dict()`.

---

## 4. Tool-Specific Correlation Implementations

Here is how to implement the same correlation logic across different load testing tools:

### Scenario: Capturing a JWT from a JSON Response and placing it in an Authorization Header

```json
{
  "status": "success",
  "data": {
    "token": "eyJhbGciOi..."
  }
}
```

#### JMeter
* **Extractor**: **JSON Extractor** (added as a child of the authentication Sampler).
  * **Names of created variables**: `authToken`
  * **JSON Path expressions**: `$.data.token`
  * **Match No. (0 for Random)**: `1`
  * **Default Value**: `TOKEN_NOT_FOUND`
* **Injection**: In the HTTP Header Manager of subsequent requests:
  * **Name**: `Authorization`
  * **Value**: `Bearer ${authToken}`

#### k6
* **Extractor & Injection**: Handled via JavaScript code in the default VU script.
```javascript
import http from 'k6/http';
import { check } from 'k6';

export default function () {
  let res = http.post('https://api.example.com/login', JSON.stringify({
    username: 'user',
    password: 'password'
  }), { headers: { 'Content-Type': 'application/json' } });

  // Extract
  let token = res.json('data.token');
  
  // Inject
  let params = {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  };
  
  let profileRes = http.get('https://api.example.com/profile', params);
  check(profileRes, { 'status is 200': (r) => r.status === 200 });
}
```

#### Gatling (Java DSL)
* **Extractor & Injection**: Handled via Gatling's session API.
```java
// Extract
http("Login")
  .post("/login")
  .body(StringBody("{\"username\":\"user\",\"password\":\"password\"}"))
  .asJson()
  .check(jsonPath("$.data.token").saveAs("authToken"))

// Inject
http("Get Profile")
  .get("/profile")
  .header("Authorization", "Bearer #{authToken}")
```

#### Locust
* **Extractor & Injection**: Handled via standard Python dictionary/json parsing.
```python
from locust import HttpUser, task, between

class WebsiteUser(HttpUser):
    wait_time = between(1, 2)
    auth_token = None

    @task
    def login(self):
        response = self.client.post("/login", json={"username": "user", "password": "password"})
        if response.status_code == 200:
            # Extract
            self.auth_token = response.json().get("data", {}).get("token")

    @task
    def get_profile(self):
        if self.auth_token:
            # Inject
            headers = {"Authorization": f"Bearer {self.auth_token}"}
            self.client.get("/profile", headers=headers)
```

#### LoadRunner (Web HTTP/HTML)
* **Extractor & Injection**: Bound using `web_reg_save_param_*` before the action.
```c
// Extract
web_reg_save_param_json(
    "ParamName=authToken",
    "QueryString=$.data.token",
    SEARCH_FILTERS,
    LAST);

web_custom_request("login",
    "URL=https://api.example.com/login",
    "Method=POST",
    "Body={\"username\":\"user\",\"password\":\"password\"}",
    LAST);

// Inject
web_add_header("Authorization", "Bearer {authToken}");

web_url("profile",
    "URL=https://api.example.com/profile",
    LAST);
```

### Multi-Value / Array Extraction

When extracting multiple values (e.g., all product IDs from a search result) rather than a single scalar:

```json
{ "results": [{ "id": "P001" }, { "id": "P002" }, { "id": "P003" }] }
```

| Tool | Extraction | Random Selection |
| :--- | :--- | :--- |
| **JMeter** | JSON Extractor with `Match No.` = `-1` → creates `productId_1`, `productId_2`, ..., `productId_matchNr` | `${__Random(1,${productId_matchNr})}` or ForEach Controller |
| **k6** | `let ids = res.json('results.#.id');` → returns JS array | `ids[Math.floor(Math.random() * ids.length)]` |
| **Gatling** | `.check(jsonPath("$.results[*].id").findAll.saveAs("ids"))` → List in session | `session.getList("ids").get(rnd.nextInt(size))` |
| **Locust** | `ids = [r["id"] for r in resp.json()["results"]]` | `random.choice(ids)` |

---

## 5. Correlation Failure Debugging

### Diagnostic Playbook

When correlation fails, work through this checklist in order:

| Symptom | Root Cause | Fix |
| :--- | :--- | :--- |
| Variable is empty / shows default value | Extractor did not match - wrong path, format changed, or extractor attached to wrong request | Inspect raw response body. Verify path against actual response. |
| Variable has a value, but it is wrong | Matched wrong occurrence (e.g., `Match No. 1` picked a different element) | Use more specific path/selector. Debug Sampler (JMeter) to see all candidates. |
| Variable correct, request still fails | Injection target wrong - encoding mismatch, wrong header/field, Content-Type mismatch | Compare injected request against a successful manual request byte-for-byte. |
| Works with 1 VU, fails with N VUs | Thread safety / scope issue - variable shared or overwritten across VUs | See Thread Safety section below. |
| Works first iteration, fails on second | Token is single-use (nonce) or expired. Needs re-extraction every iteration. | Move extraction inside the iteration loop. Verify token TTL. |
| Works locally, fails distributed | Variable extracted on one node unavailable on another | Ensure extraction and injection happen within same node's request chain. |
| Intermittent failures under load | Race condition - response arrives before prior extraction completes (async) | Add explicit waits / ordering. Check for HTTP/2 multiplexing reordering. |

### Thread Safety / VU Scope

Each tool scopes variables differently. Getting this wrong causes User A's token to leak to User B:

| Tool | Default Scope | Cross-VU Sharing Trap |
| :--- | :--- | :--- |
| **JMeter** | Thread-local (`vars`) | `__setProperty`/`props` is global - never use for session tokens. |
| **k6** | VU-scoped (inside `default()`) | Module-level `let`/`var` outside `default()` is shared - keep state inside `default()`. |
| **Gatling** | Session-scoped (per user) | `global` feeders with `.circular` can cause two users to get same row. |
| **Locust** | Instance-scoped (`self.*`) | Module-level variables are shared across all User instances. |
| **LoadRunner** | Vuser-scoped (`lr_save_string`) | Shared data tables with `Unique` allocation can exhaust rows. |

### Encoding & Escaping

Mismatched encoding is the #1 cause of "variable is correct but request fails":

| Scenario | Problem | Solution |
| :--- | :--- | :--- |
| Value in URL query string | `=`, `&`, `+`, spaces break URL parsing | URL-encode: JMeter `${__urlencode()}`, k6 `encodeURIComponent()` |
| Value in JSON body | Quotes, backslashes, newlines break JSON | JSON-escape: k6 `JSON.stringify()`, JMeter `${__groovy()}` |
| Value in XML/SOAP body | `<`, `>`, `&` break XML parsing | XML-escape: JMeter `${__escapeXml()}` |
| ViewState / SAML from HTML | HTML-entity-encoded (`&amp;` vs `&`) | Decode entities before injection: JMeter `${__unescapeHtml()}` |
| URL-encoded form with pre-encoded values | Double-encoding: `%253D` instead of `%3D` | Extract raw value, let the tool's form encoder handle it |
| Base64 tokens in URL | `+`, `/`, `=` are URL-unsafe | Use base64url encoding for URLs; raw base64 for headers/body |

---

## 6. Brittle Regex Patterns vs. Robust Regex Patterns

When Regular Expressions are necessary, follow these practices to avoid brittle matches that break on minor code changes:

| Target Value | Brittle Regex | Robust Regex | Rationale |
| :--- | :--- | :--- | :--- |
| **CSRF Token in HTML input** | `value="(.+?)"` | `<input[^>]*name="_csrf"[^>]*value="([^"]+)"` | Brittle regex matches the first input value it encounters. Robust regex anchors specifically to the `_csrf` input element. |
| **Session ID in URL** | `jsessionid=(.*)` | `jsessionid=([a-zA-Z0-9\-]+)` | Brittle regex matches greedily, capturing trailing URL pathing. Robust regex restricts matching to hexadecimal or alphanumeric characters. |
| **JSON Attribute** | `"id":(\d+)` | `"id"\s*:\s*(\d+)` | Brittle regex breaks if the server formatter adds or removes spaces around the colon. |

---

## 7. Known Correlation Rules for Common Tech Stacks

Many enterprise application stacks rely on standardized frameworks with pre-determined dynamic values. Knowing your target application's tech stack tells you exactly what to search for:

### ASP.NET (IIS / WebForms / MVC)
* **`__VIEWSTATE`**: Base64 encoded page state (HTML hidden input). Changes on every page state alteration. Use **CSS Selector**: `input[name=__VIEWSTATE]` or **Regex**: `name="__VIEWSTATE"\s+id="__VIEWSTATE"\s+value="([^"]+)"`.
* **`__EVENTVALIDATION`**: Form validation parameter. Use **CSS Selector**: `input[name=__EVENTVALIDATION]`.
* **`__VIEWSTATEGENERATOR`**: Layout hash identifier. Use **CSS Selector**: `input[name=__VIEWSTATEGENERATOR]`.
* **`__RequestVerificationToken`**: Anti-forgery CSRF token. Often found in cookies or hidden forms.

### Java (JSF / Spring Security / Liferay)
* **`javax.faces.ViewState`**: JSF view state. Vital for page progression. Use **CSS Selector**: `input[name*='ViewState']` or `[id$='ViewState']` (due to namespace prefixes).
* **`_csrf`**: Spring Security anti-CSRF token. Usually in a form input (`input[name=_csrf]`) or in HTML header meta tags (`meta[name="_csrf"]`).
* **`JSESSIONID`**: Session identifier cookie. If the client blocks cookies, Java rewrites URLs containing `;jsessionid=VALUE`. Must be correlated in paths.

### Oracle (ADF / PeopleSoft / WebLogic)
* **`_afrLoop`**: ADF request loop parameter in query strings. Captured using Regex from response scripts or redirects: `_afrLoop=([a-zA-Z0-9\-]+)`.
* **`Adf-Page-State-Id`**: ADF page view state identifier.
* **`PS_TOKEN`**: PeopleSoft single sign-on token (Cookie).

### SAP (NetWeaver / WebDynpro / SAP Portal)
* **`sap-contextid` / `sap-wd-secure-id`**: Required for session state validation. Captured from JS script blocks or XML redirects.
* **`sap-ext-sid`**: SAP external session ID.
* **`jSessionID` / `sap-j2ee-engine`**: J2EE engine session cookies.

### Authentication & Authorization Frameworks (OAuth 2.0 / OIDC / SAML)
* **`code`**: Authorization Code flow redirect parameter. Captured from the redirect URI `Location` header or dynamic script redirects: `code=([^&]+)`.
* **`state`**: Session binding token. Passed to authorize endpoint and must match on response redirect.
* **`SAMLResponse`**: XML payload representing SAML assertion. Encoded base64 in HTML form input: `input[name=SAMLResponse]`.

### Django
* **`csrfmiddlewaretoken`**: Hidden form input. Use **CSS Selector**: `input[name=csrfmiddlewaretoken]`. Also sent as `csrftoken` cookie - verify the cookie manager handles it; if not, extract from the form.

### Ruby on Rails
* **`authenticity_token`**: Anti-CSRF token in hidden form inputs. Use **CSS Selector**: `input[name=authenticity_token]`. Also in `<meta name="csrf-token">` for AJAX requests.

### PHP / Laravel
* **`_token`**: Laravel CSRF token in hidden form fields. Use **CSS Selector**: `input[name=_token]`.
* **`XSRF-TOKEN`**: Cookie-based CSRF (Laravel default). Extracted from `Set-Cookie` header, injected as `X-XSRF-TOKEN` request header (URL-decoded).

### Next.js / React SSR
* **`__NEXT_DATA__`**: Server-rendered hydration payload inside `<script id="__NEXT_DATA__">`. Contains `buildId`, props, and server-side tokens. Extract with **Regex**: `<script id="__NEXT_DATA__"[^>]*>({.+?})</script>`, then parse the JSON.
* **Client-side tokens**: Often fetched via XHR after page load. Correlate from the API response, not the HTML page.

---

## 8. How the LLM Can Help You (Interactive Framework)

If you need help configuring correlation, copy and paste the following prompt template along with your data:

> **Correlation Assistance Request**
> 1. **Target Tool**: [e.g., JMeter, k6, Gatling, Locust]
> 2. **Pre-identified Dynamic Values**: [List any parameters/headers/tokens you already know need correlation, e.g., `JSESSIONID`, `__VIEWSTATE`, `csrf_token`, or how the target value behaves if known]
> 3. **Step 1 Response (Source)**:
>    ```
>    [Paste the HTTP response body or headers containing the dynamic value]
>    ```
> 4. **Step 2 Request (Destination)**:
>    ```
>    [Paste the HTTP request body, URL, or headers where the value needs to be used]
>    ```
> 5. **Goal**: Help me extract the dynamic value from Step 1 and inject it into Step 2.

### The LLM Response Action Plan:
1. Parse the response structure (JSON, XML, HTML, or Headers).
2. Propose the optimal Extractor type (e.g., JSONPath, CSS Selector, Regex) to minimize brittleness.
3. Write the exact extraction syntax for the target tool.
4. Show the exact injection syntax (including variable syntax, e.g., `${varName}`, `#{varName}`, `{varName}`).
5. Explain how to debug and verify that the correlation succeeded.
