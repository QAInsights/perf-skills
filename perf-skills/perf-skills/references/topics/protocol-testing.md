# Protocol-Specific Performance Testing

Covers gRPC, GraphQL, WebSocket/SSE, and message queue (Kafka, RabbitMQ, SQS) load testing — protocols with unique challenges beyond standard HTTP/REST.

---

## gRPC Performance Testing

gRPC uses HTTP/2, Protocol Buffers, and persistent connections — fundamentally different from REST at the wire level.

### Key Challenges

| Challenge | Why It Matters |
|---|---|
| **Protobuf compilation** | Tests require compiled `.proto` definitions; can't just send raw JSON |
| **Connection multiplexing** | HTTP/2 multiplexes streams over a single connection — fewer connections needed but different bottleneck profile |
| **Streaming** | Unary, server-streaming, client-streaming, and bidirectional-streaming each need different test strategies |
| **Metadata vs headers** | gRPC metadata is the equivalent of HTTP headers; auth tokens go here |
| **Error codes** | gRPC uses its own status codes (OK, UNAVAILABLE, DEADLINE_EXCEEDED) — not HTTP status codes |

### Tool Support

| Tool | gRPC Support | Notes |
|---|---|---|
| **k6** | `k6/grpc` built-in module | Supports unary and streaming; load `.proto` or use reflection |
| **Gatling** | `gatling-grpc` plugin | Scala/Java DSL; supports unary and streaming |
| **JMeter** | `gRPC Sampler` plugin | GUI-based; requires proto descriptor file |
| **ghz** | Dedicated gRPC benchmarking tool | CLI-only; excellent for quick benchmarks |

### k6 gRPC Example

```javascript
import grpc from 'k6/grpc';
import { check, sleep } from 'k6';

const client = new grpc.Client();
client.load(['definitions'], 'hello.proto');

export default function () {
  client.connect('grpc-server:50051', { plaintext: true });

  const response = client.invoke('hello.HelloService/SayHello', {
    greeting: 'perf-test',
  });

  check(response, {
    'status is OK': (r) => r && r.status === grpc.StatusOK,
    'has message': (r) => r && r.message.reply !== '',
  });

  client.close();
  sleep(1);
}
```

### gRPC Testing Considerations

- **Connection reuse**: Keep connections open across iterations; don't connect/close per request.
- **Streaming throughput**: For server-streaming RPCs, measure messages-per-second, not just request latency.
- **Deadline propagation**: Set gRPC deadlines in tests to match production timeouts.
- **Load balancer awareness**: gRPC over HTTP/2 with persistent connections can cause uneven load across backends — test with client-side load balancing or L7 proxy.
- **Protobuf payload size**: Binary encoding is smaller than JSON — adjust throughput expectations accordingly.

---

## GraphQL Performance Testing

GraphQL introduces query complexity as a variable — the same endpoint can serve trivially cheap or devastatingly expensive requests.

### Key Challenges

| Challenge | Why It Matters |
|---|---|
| **Variable query cost** | A single endpoint can serve queries with wildly different server costs |
| **N+1 resolver problem** | Nested queries can trigger cascading DB calls; load tests expose this at scale |
| **Query depth / complexity** | Deep nesting can cause exponential resolver execution |
| **Batched queries** | Clients may send multiple operations in one request |
| **Persisted queries** | Production may use query allowlists; ad-hoc queries get rejected |

### Testing Strategy

1. **Catalog production queries**: Extract real queries from logs or APM traces — don't invent synthetic queries.
2. **Categorize by cost**: Light (single field), medium (nested 2 levels), heavy (deep joins, lists).
3. **Build a realistic query mix**: Weight tests by actual production query distribution.
4. **Test with and without caching**: GraphQL caching (DataLoader, CDN) dramatically changes performance profiles.

### k6 GraphQL Example

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

const GRAPHQL_ENDPOINT = `${__ENV.BASE_URL}/graphql`;

const QUERIES = {
  lightQuery: `query { user(id: "${__VU}") { name email } }`,
  heavyQuery: `query { users(first: 100) { edges { node { name orders(last: 10) { totalAmount items { name } } } } } }`,
};

export default function () {
  // 80% light queries, 20% heavy queries
  const query = Math.random() < 0.8 ? QUERIES.lightQuery : QUERIES.heavyQuery;

  const res = http.post(GRAPHQL_ENDPOINT, JSON.stringify({ query }), {
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${__ENV.TOKEN}`,
    },
    tags: { name: 'POST /graphql' },
  });

  check(res, {
    'status 200': (r) => r.status === 200,
    'no errors': (r) => !r.json('errors'),
    'has data': (r) => r.json('data') !== null,
  });

  sleep(1);
}
```

### GraphQL-Specific Metrics to Track

- **Query complexity score** (if server exposes it)
- **Resolver execution time** (via APM / tracing)
- **DataLoader batch efficiency** (cache hit rate)
- **Error rate by query type** (complexity-related timeouts vs auth errors)

---

## WebSocket / SSE Performance Testing

Persistent connections have fundamentally different scaling characteristics from request/response HTTP.

### Key Challenges

| Challenge | Why It Matters |
|---|---|
| **Connection count** | Each VU holds an open connection — tests memory/fd limits, not just CPU |
| **Message throughput** | Measure messages/sec independently from connection count |
| **Backpressure** | What happens when the server can't send fast enough? |
| **Reconnection behavior** | Dropped connections should auto-reconnect; test the reconnect storm |
| **Fan-out cost** | Broadcasting to N connections has O(N) server cost |

### Tool Support

| Tool | WebSocket | SSE |
|---|---|---|
| **k6** | `k6/ws` module | Not built-in; use HTTP streaming |
| **Gatling** | Full WebSocket DSL | Not built-in |
| **JMeter** | WebSocket Sampler plugin | Not built-in |
| **Artillery** | Built-in `ws` engine | Built-in SSE |

### k6 WebSocket Example

```javascript
import ws from 'k6/ws';
import { check, sleep } from 'k6';

export default function () {
  const url = `${__ENV.WS_URL}/ws/chat`;
  const params = { headers: { 'Authorization': `Bearer ${__ENV.TOKEN}` } };

  const res = ws.connect(url, params, function (socket) {
    socket.on('open', () => {
      socket.send(JSON.stringify({ type: 'subscribe', channel: 'updates' }));
    });

    socket.on('message', (msg) => {
      const data = JSON.parse(msg);
      check(data, {
        'has type': (d) => d.type !== undefined,
      });
    });

    socket.on('error', (e) => {
      console.error('WS error:', e.error());
    });

    // Hold connection open for 30 seconds, sending periodic pings
    socket.setInterval(() => {
      socket.send(JSON.stringify({ type: 'ping' }));
    }, 5000);

    socket.setTimeout(() => {
      socket.close();
    }, 30000);
  });

  check(res, { 'WS status 101': (r) => r && r.status === 101 });
  sleep(1);
}
```

### WebSocket Testing Strategy

1. **Connection ramp**: Gradually open connections — don't blast 10k connections at once.
2. **Separate connection test from message test**: First, find max stable connections; then test message throughput at a stable connection count.
3. **Measure server-side**: File descriptor count, memory per connection, event loop lag.
4. **Test reconnection storms**: Kill server, observe client reconnection behavior and server recovery.

---

## Message Queue / Event Streaming Testing

Testing Kafka, RabbitMQ, SQS, and similar systems requires measuring producer throughput, consumer lag, and end-to-end latency.

### Key Challenges

| Challenge | Why It Matters |
|---|---|
| **Producer throughput** | Messages/sec the system can ingest |
| **Consumer lag** | How far behind consumers fall under load |
| **End-to-end latency** | Time from produce to consume — the real user-facing metric |
| **Partition scaling** | Kafka throughput scales with partitions; test with realistic partition counts |
| **Message ordering** | Under load, verify ordering guarantees still hold |
| **Dead letter queues** | Verify failed messages route correctly under pressure |

### Tool Support

| Tool | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| **k6 (xk6-kafka)** | Yes | No (use HTTP management API) | No |
| **JMeter** | Kafka plugin / JMS Sampler | JMS Sampler | AWS SDK Sampler |
| **kafka-producer-perf-test** | Built-in Kafka tool | N/A | N/A |
| **rabbitmq-perf-test** | N/A | Official RabbitMQ tool | N/A |

### k6 Kafka Example (xk6-kafka)

```javascript
import { Writer, Reader, Connection } from 'k6/x/kafka';

const writer = new Writer({ brokers: ['kafka:9092'], topic: 'perf-test' });
const reader = new Reader({ brokers: ['kafka:9092'], topic: 'perf-test', groupID: 'perf-group' });

export default function () {
  // Produce
  writer.produce({
    messages: [{
      key: `key-${__VU}-${__ITER}`,
      value: JSON.stringify({ orderId: `${__VU}-${__ITER}`, timestamp: Date.now() }),
    }],
  });

  // Consume (in a separate scenario ideally)
  const messages = reader.consume({ limit: 1 });
  // Validate message content
}

export function teardown() {
  writer.close();
  reader.close();
}
```

### Message Queue Testing Strategy

1. **Test producers and consumers separately first**, then together.
2. **Measure consumer lag over time** — a growing lag under steady load indicates a bottleneck.
3. **Test with realistic message sizes** — a 100-byte message vs a 1MB payload have very different throughput ceilings.
4. **Verify idempotency** — under load with retries, duplicate messages should not corrupt state.
5. **Test partition rebalancing** — add/remove consumers during a test to verify rebalance behavior.

---

## Protocol Testing Checklist

- [ ] Proto definitions / schemas compiled and available to test tool
- [ ] Connection reuse strategy defined (persistent vs per-request)
- [ ] Streaming scenarios identified and scripted separately from unary/request-response
- [ ] Message throughput targets defined (messages/sec, not just RPS)
- [ ] Backpressure / flow control behavior validated
- [ ] Error codes and retry behavior tested (gRPC status codes, AMQP nacks, etc.)
- [ ] End-to-end latency measured (not just request latency)
