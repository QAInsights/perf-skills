# Test Data Strategy

Test data is a first-class concern in performance testing. Bad data causes false failures, data collisions, unrealistic server behavior, and post-test cleanup nightmares.

---

## Why Test Data Matters

- **Authentication tokens** expire — hardcoded tokens fail within minutes.
- **Unique constraints** (email, username, order number) cause failures at concurrency > 1.
- **State-dependent flows** (checkout, approval) require data in the right state before the test.
- **Cache effects** — if 1000 VUs all hit the same product ID, cache hit rates are unrealistically high; spread data across IDs.

---

## Data Strategy by Test Type

| Test Type | Data Strategy |
|---|---|
| Smoke (1 VU) | Minimal — single known-good user/record |
| Load (realistic VUs) | Pool of pre-created users/records matching production volume |
| Stress (pushing limits) | Large pool; no uniqueness conflicts |
| Soak (hours) | Rotating/recyclable data; handle state cleanup |
| Spike (burst) | Same as load; ensure pool size exceeds spike VU count |

---

## Data Sources

### Strategy 1: Pre-Generated CSV Files

Best for: stable, static data (user credentials, product IDs, account numbers).

```
data/
├── users.csv          (username, password, account_id)
├── products.csv       (product_id, sku, price)
├── accounts.csv       (account_number, routing_number, balance)
└── orders.csv         (order_id, status, customer_id)
```

**Sizing:** CSV row count should be ≥ peak VU count to prevent multiple VUs sharing the same row and causing collisions.

**JMeter:** `CSV Data Set Config` element  
**k6:** `SharedArray` with `open()` or `papaparse`  
**Gatling:** `csv("users.csv").circular`  
**Locust:** Read CSV in `on_start()` or `__init__.py`

---

### Strategy 2: Database-Seeded Data

Best for: complex relational state (orders in specific statuses, accounts with balances, workflows pending approval).

**Approach:**
1. Write a seeding script (Python, SQL, or test framework) to create N records in the right state.
2. Export IDs to CSV for the load tool to consume.
3. After the test, run a cleanup script.

```sql
-- Seed 5000 users for load test
INSERT INTO test_users (username, password_hash, tenant_id)
SELECT 
    'loadtest_user_' || generate_series(1, 5000),
    '$2b$12$fixedhash',
    'perf-test-tenant'
;

-- Export to CSV
COPY (SELECT username, 'testpass123' AS password FROM test_users WHERE username LIKE 'loadtest_%')
TO '/tmp/users.csv' CSV HEADER;
```

---

### Strategy 3: API-Based Data Setup (Setup Hooks)

Best for: data that must be in a specific state per VU (unique cart, session, transaction).

```javascript
// k6 setup() — run before VUs start
export function setup() {
  const orders = [];
  for (let i = 0; i < 200; i++) {
    const res = http.post('https://api.example.com/admin/orders', JSON.stringify({
      status: 'pending',
      amount: Math.random() * 1000
    }), { headers: adminHeaders });
    orders.push(res.json('orderId'));
  }
  return { orders };  // Passed to default() as data argument
}
```

```python
# Locust on_start()
def on_start(self):
    res = self.client.post("/api/checkout/initiate", json={"cartId": new_uuid()})
    self.checkout_id = res.json()["checkoutId"]
```

---

### Strategy 4: Faker / Synthetic Data Generation

Best for: registration flows, profile creation, form submission — where each VU needs unique PII-like data.

```python
# Python (Locust or data generation script)
from faker import Faker
fake = Faker()

user = {
    "name": fake.name(),
    "email": fake.unique.email(),
    "address": fake.address(),
    "phone": fake.phone_number()
}
```

```javascript
// k6 with randomString
import { randomString, randomIntBetween } from 'https://jslib.k6.io/k6-utils/1.4.0/index.js';

const email = `user_${randomString(8)}@loadtest.example.com`;
```

```groovy
// JMeter JSR223
import java.util.UUID
vars.put("email", "user_${UUID.randomUUID()}@loadtest.example.com")
```

---

## Data Isolation Patterns

### Tenant Isolation
Create a dedicated `perf-test` tenant in multi-tenant systems. This prevents test data from polluting other tenants and enables easy cleanup.

### Test Data Tagging
Tag all test records (e.g., `source: "perf-test"`, `env: "load-test"`) so they can be identified and deleted post-test.

### Ephemeral Environments
In cloud-native architectures, spin up a fresh environment for each test run (Infrastructure as Code), then destroy it. Eliminates test data pollution entirely.

### Data Reset Hooks
Register teardown hooks to delete or reset test data after the run:

```python
# Locust teardown
@events.test_stop.add_listener
def cleanup(environment, **kwargs):
    requests.delete(
        f"{HOST}/admin/test-data",
        headers=admin_headers,
        json={"tag": "perf-test"}
    )
```

---

## Parameterization Patterns

### Round-Robin (default for most tools)
Each VU/iteration picks the next row. Ensures even distribution.

### Random
Better for cache-busting tests where you want realistic cache miss rates.

### Unique-Per-VU
Each VU gets a dedicated row — critical for state-dependent flows (user owns specific order). Use VU index (`__VU` in k6, `${__threadNum}` in JMeter) to deterministically select a row.

```javascript
// k6: VU-indexed data
const user = users[__VU - 1];  // VU 1 gets row 0, VU 2 gets row 1, etc.
```

```xml
<!-- JMeter: use __threadNum to assign data -->
<!-- In CSV Data Set Config, Sharing Mode = Current Thread Group -->
<!-- Or use: ${__groovy(vars.get("__threadNum").toInteger() - 1)} -->
```

---

## Test Data Checklist

- [ ] Sufficient data volume (≥ peak VU count rows for unique-per-VU flows)
- [ ] Data is in the correct initial state for each scenario
- [ ] No hardcoded credentials or tokens
- [ ] Data isolated from production (separate tenant, tagging, or ephemeral env)
- [ ] Cleanup mechanism in place post-test
- [ ] CSV files available on all injector nodes (distributed tests)
- [ ] Sensitive data masked/anonymized (not using real PII in test data)
- [ ] Date/time sensitive records accounted for (e.g., expiry dates set far in future)
