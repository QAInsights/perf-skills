# Gatling Reference

> Targets: Gatling 3.10+, Java DSL 3.7+

Gatling is a high-performance, Scala/Java-based load testing tool designed for HTTP-heavy applications. Its simulation DSL is expressive, and its async Netty-based engine handles very high concurrency with low resource overhead.

---

## Core Concepts

| Concept | Description |
|---|---|
| **Simulation** | Top-level test class extending `Simulation` |
| **Protocol** | HTTP, JMS, or gRPC configuration |
| **Scenario** | Named sequence of actions (chain of steps) |
| **Feeder** | Data source for parameterization (CSV, JSON, JDBC) |
| **Check** | Assertion on a response (status, body, header) |
| **Population** | VU injection strategy (open/closed model) |
| **Session** | VU-level state store (like JMeter `vars`) |

---

## Simulation Structure

```scala
import io.gatling.core.Predef._
import io.gatling.http.Predef._
import scala.concurrent.duration._

class CheckoutSimulation extends Simulation {

  // 1. Protocol config
  val httpProtocol = http
    .baseUrl("https://api.example.com")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .shareConnections  // Reuse connections across VUs (more realistic)

  // 2. Feeders (data)
  val userFeeder = csv("data/users.csv").circular

  // 3. Scenario (chain of steps)
  val loginScenario = scenario("Login and Browse")
    .feed(userFeeder)
    .exec(
      http("POST Login")
        .post("/auth/login")
        .body(StringBody("""{"username":"#{username}","password":"#{password}"}""")).asJson
        .check(status.is(200))
        .check(jsonPath("$.token").saveAs("authToken"))
    )
    .pause(1, 3)  // Think time: 1–3 seconds
    .exec(
      http("GET Products")
        .get("/products")
        .header("Authorization", "Bearer #{authToken}")
        .check(status.is(200))
        .check(jsonPath("$[0].id").saveAs("productId"))
    )
    .pause(2)

  // 4. Injection (load profile)
  setUp(
    loginScenario.inject(
      nothingFor(5.seconds),         // Wait before starting
      atOnceUsers(5),                // Smoke check
      rampUsers(100).during(2.minutes),
      constantUsersPerSec(20).during(5.minutes),
      rampUsersPerSec(20).to(0).during(1.minute)
    ).protocols(httpProtocol)
  ).assertions(
    global.responseTime.percentile(95).lt(500),
    global.failedRequests.percent.lt(1)
  )
}
```

---

## Injection Profiles

### Closed model (VU-based)
```scala
rampUsers(100).during(2.minutes)        // Ramp to 100 VUs over 2 min
constantConcurrentUsers(100).during(5.minutes)  // Hold at 100 VUs
```

### Open model (arrival-rate)
```scala
constantUsersPerSec(50).during(5.minutes)           // 50 new users/sec
rampUsersPerSec(10).to(100).during(3.minutes)       // Ramp RPS from 10 to 100
heavisideUsers(1000).during(20.seconds)             // S-curve injection (realistic burst)
```

> For closed vs open model guidance and when to use each, see `references/topics/workload-design.md`.

---

## Feeders (Parameterization)

```scala
// CSV
val csvFeeder = csv("data/users.csv").circular      // Loop forever
val csvFeeder = csv("data/users.csv").random        // Pick random row
val csvFeeder = csv("data/users.csv").shuffle       // Randomize order, no repeat
val csvFeeder = csv("data/users.csv").queue         // Each VU gets next row; fail if empty

// JSON
val jsonFeeder = jsonFile("data/products.json").random

// Custom feeder
val customFeeder = Iterator.continually(Map(
  "timestamp" -> System.currentTimeMillis(),
  "uuid"      -> java.util.UUID.randomUUID().toString
))

// Use in scenario
scenario("test").feed(csvFeeder).exec(...)
```

---

## Checks (Assertions)

```scala
// Status code
.check(status.is(200))
.check(status.in(200, 201))

// JSON body
.check(jsonPath("$.userId").exists)
.check(jsonPath("$.token").saveAs("token"))
.check(jsonPath("$.items.length()").is("5"))

// Header
.check(header("X-Request-Id").exists)

// Response time
.check(responseTimeInMillis.lte(2000))

// Body string
.check(bodyString.contains("success"))
.check(substring("access_granted").count.is(1))

// Multiple checks (all must pass)
.check(status.is(200), jsonPath("$.status").is("ok"))
```

---

## Session Variables

```scala
// Save during check
.check(jsonPath("$.orderId").saveAs("orderId"))

// Access in body
.body(StringBody("""{"orderId":"#{orderId}"}"""))

// Access in Exec block
.exec(session => {
  val orderId = session("orderId").as[String]
  println(s"Processing order: $orderId")
  session
})

// Conditional logic
.doIf(session => session("isAdmin").as[Boolean]) {
  exec(http("Admin Action").get("/admin"))
}
```

---

## Assertions (Global SLA)

```scala
setUp(...).assertions(
  global.responseTime.percentile(95).lt(500),     // p95 < 500ms
  global.responseTime.max.lt(2000),               // max < 2s
  global.failedRequests.percent.lt(1),            // Error rate < 1%
  global.requestsPerSec.gte(100),                 // Throughput >= 100 RPS
  forAll.failedRequests.count.is(0),              // No failures on any request
  details("POST Login").responseTime.percentile(99).lt(1000)  // Named request SLA
)
```

---

## Protocol Configuration

### HTTP
```scala
val httpProtocol = http
  .baseUrl("https://api.example.com")
  .proxy(Proxy("proxy.corp.net", 8080))
  .acceptHeader("application/json")
  .acceptEncodingHeader("gzip, deflate")
  .userAgentHeader("Gatling/Perf-Test")
  .maxConnectionsPerHost(10)
  .warmUp("https://api.example.com/health")
  .disableFollowRedirect           // Disable for accuracy
  .disableAutoReferer
```

### WebSocket
```scala
import io.gatling.http.Predef._

exec(
  ws("Connect WS").connect("/ws/chat")
    .await(5.seconds)(
      ws.checkTextMessage("message received")
        .matching(jsonPath("$.type").is("ack"))
    )
)
.exec(ws("Send Message").sendText("""{"msg":"hello"}"""))
.exec(ws("Close").close())
```

---

## Running Gatling

```bash
# Maven (Scala DSL)
mvn gatling:test -Dgatling.simulationClass=CheckoutSimulation

# Gradle (Java DSL)
./gradlew gatlingRun

# Bundle (no build tool)
./bin/gatling.sh -s CheckoutSimulation -rd "Staging load test"

# With overrides
mvn gatling:test \
  -Dgatling.simulationClass=CheckoutSimulation \
  -DbaseUrl=https://staging.example.com \
  -Dusers=100 \
  -Dduration=300
```

---

## Java DSL (Gatling 3.7+)

Gatling now supports Java and Kotlin natively — no Scala required:

```java
import io.gatling.javaapi.core.*;
import io.gatling.javaapi.http.*;
import static io.gatling.javaapi.core.CoreDsl.*;
import static io.gatling.javaapi.http.HttpDsl.*;

public class LoginSimulation extends Simulation {
  HttpProtocolBuilder httpProtocol = http.baseUrl("https://api.example.com");

  ScenarioBuilder scenario = scenario("Login")
    .exec(
      http("POST Login")
        .post("/auth/login")
        .body(StringBody("{\"user\":\"test\"}")).asJson()
        .check(status().is(200))
    );

  { setUp(scenario.injectOpen(rampUsers(100).during(60))).protocols(httpProtocol); }
}
```

---

## Results

Gatling generates an HTML report in `target/gatling/<simulation-timestamp>/index.html` — commit the link to CI artifacts or use Gatling Enterprise for centralized dashboards.

---

## Gatling-Specific Tips

- **Never use blocking/synchronous calls inside `exec`** — Gatling's engine is async; blocking calls degrade throughput significantly.
- **Use `.warmUp()` or an initial ramp phase** — JVM JIT compilation distorts early metrics without warm-up.
- **Use `global` or named `details()` assertions** — asserting on individual requests instead of aggregated transactions is noisy.
- **Prefer `heavisideUsers` for burst injection** — S-curve injection is more realistic than `atOnceUsers` for spike tests.

> For CI/CD integration (Maven, Gradle, GitHub Actions), see `references/topics/test-execution.md`.
> For anti-patterns, assertions, think time, and parameterization principles, see **Key Principles** in `SKILL.md`.
