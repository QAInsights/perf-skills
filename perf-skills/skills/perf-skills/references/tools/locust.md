# Locust Reference

> Targets: Locust 2.20+, Python 3.9+

Locust is a Python-based, open-source load testing tool. Tests are plain Python code — no DSL, no XML. It supports distributed testing and has a built-in web UI.

---

## Core Concepts

| Concept | Description |
|---|---|
| **User class** | Defines behavior of a simulated user |
| **TaskSet** | Group of tasks; can be nested |
| **task decorator** | Marks a method as a task with optional weight |
| **wait_time** | Think time between tasks |
| **HttpUser** | Preconfigured User with HTTP client |
| **FastHttpUser** | High-performance HTTP client (gevent-based) |
| **events** | Hooks for setup, teardown, request success/failure |

---

## Basic Script

```python
from locust import HttpUser, task, between
import json, random

class ShopUser(HttpUser):
    wait_time = between(1, 4)  # Think time: 1–4 seconds (uniform random)
    host = "https://api.example.com"

    def on_start(self):
        """Runs once per VU at startup"""
        res = self.client.post("/auth/login", json={
            "username": f"user{random.randint(1, 1000)}@test.com",
            "password": "testpass123"
        })
        self.token = res.json()["access_token"]
        self.client.headers.update({"Authorization": f"Bearer {self.token}"})

    @task(3)                   # Weight: called 3x more than weight-1 tasks
    def browse_products(self):
        self.client.get("/products", name="GET /products")

    @task(1)
    def add_to_cart(self):
        product_id = random.randint(1, 100)
        with self.client.post(
            f"/cart/items",
            json={"productId": product_id, "qty": 1},
            catch_response=True,    # Manually handle response
            name="POST /cart/items"
        ) as response:
            if response.status_code == 200:
                response.success()
            else:
                response.failure(f"Expected 200, got {response.status_code}")

    def on_stop(self):
        self.client.post("/auth/logout")
```

---

## Running Locust

```bash
# Web UI mode
locust -f locustfile.py --host https://api.example.com

# Headless (CLI) mode
locust -f locustfile.py \
  --headless \
  --users 100 \
  --spawn-rate 10 \
  --run-time 5m \
  --host https://api.example.com \
  --html report.html \
  --csv results

# Distributed mode
# On master:
locust -f locustfile.py --master --users 1000 --spawn-rate 50
# On each worker:
locust -f locustfile.py --worker --master-host=<master-ip>
```

---

## Custom Wait Times

```python
from locust import constant, constant_pacing, between, constant_throughput

wait_time = between(1, 5)              # Uniform random 1–5s
wait_time = constant(2)                # Fixed 2s
wait_time = constant_pacing(10)        # Paces to 1 iteration per 10s (handles slow responses)
wait_time = constant_throughput(0.1)   # Target 0.1 RPS per VU (10s average)
```

---

## Custom Events and Hooks

```python
from locust import events

@events.test_start.add_listener
def on_test_start(environment, **kwargs):
    print("Test starting — seeding test data")

@events.request.add_listener
def on_request(request_type, name, response_time, response_length, exception, **kwargs):
    if exception:
        print(f"FAILED: {name} — {exception}")

@events.test_stop.add_listener
def on_test_stop(environment, **kwargs):
    print("Test finished — cleaning up data")
```

---

## Locust-Specific Tips

- Always use `name=` parameter to group parameterized URLs (e.g., `/products/123` → `name="GET /products/{id}"`).
- Use `catch_response=True` for custom validation — HTTP 200 with error body passes silently otherwise.
- Use `FastHttpUser` instead of `HttpUser` for CPU-bound or high-throughput scenarios.
- Use distributed mode above 500–1000 VUs — a single worker process saturates one CPU core.

> For anti-patterns, assertions, think time, and parameterization principles, see **Key Principles** in `SKILL.md`.
> For CI/CD integration and distributed execution details, see `references/topics/test-execution.md`.
