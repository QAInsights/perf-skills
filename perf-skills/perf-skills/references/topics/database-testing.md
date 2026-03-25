# Database Performance Testing

Covers load testing databases directly (JDBC, connection pools, query concurrency) — not just testing the application layer that sits in front of them.

---

## When to Test the Database Directly

| Scenario | Why Direct DB Testing |
|---|---|
| **New schema or index changes** | Validate query performance under concurrency before deploying |
| **Connection pool tuning** | Find optimal pool size, timeout, and eviction settings |
| **Read replica lag** | Measure replication delay under write-heavy load |
| **Stored procedure performance** | Procedures with complex logic need load testing independent of the app |
| **Migration validation** | After DB engine upgrade (e.g., MySQL 5.7 → 8.0), verify no regression |
| **Deadlock detection** | Concurrent writes to overlapping rows expose locking issues |

---

## Tool Support

| Tool | DB Support | How |
|---|---|---|
| **JMeter** | JDBC Sampler + JDBC Connection Configuration | GUI-based; supports any JDBC-compatible database |
| **k6** | `xk6-sql` extension | Custom build required; supports Postgres, MySQL, SQLite |
| **Gatling** | JDBC feeder (read-only) | Feeders can read from DB; no native write support |
| **pgbench** | PostgreSQL only | Built-in PostgreSQL benchmarking tool |
| **sysbench** | MySQL, PostgreSQL | Industry-standard DB benchmark tool |
| **HammerDB** | Oracle, SQL Server, MySQL, PostgreSQL | Open-source TPC-C / TPC-H workload generator |

---

## JMeter JDBC Testing

### Setup

```
1. Add JDBC Connection Configuration (Config Element)
   Variable Name:     myDB
   Database URL:      jdbc:postgresql://db-host:5432/myapp
   JDBC Driver:       org.postgresql.Driver
   Username:          perf_test_user
   Password:          ${__P(db.password)}
   Max Connections:   50
   Connection Timeout: 10000
   Idle Timeout:      60000

2. Add JDBC Request (Sampler)
   Variable Name:     myDB
   Query Type:        Select Statement
   Query:             SELECT * FROM orders WHERE customer_id = ? AND status = ?
   Parameter Values:  ${customer_id},${status}
   Parameter Types:   INTEGER,VARCHAR
```

### Common JDBC Test Patterns

**Read-heavy workload:**
```sql
-- Simulate product catalog browsing
SELECT p.*, c.name AS category
FROM products p
JOIN categories c ON p.category_id = c.id
WHERE p.active = true
ORDER BY p.created_at DESC
LIMIT 20 OFFSET ${__Random(0,1000)};
```

**Write-heavy workload:**
```sql
-- Simulate order creation under concurrency
INSERT INTO orders (customer_id, total, status, created_at)
VALUES (${customer_id}, ${total}, 'pending', NOW())
RETURNING id;
```

**Mixed read-write (realistic):**
Use JMeter Throughput Controller to mix 80% reads / 20% writes.

---

## Connection Pool Testing

Connection pool misconfiguration is one of the most common database performance bottlenecks.

### What to Test

| Parameter | Test Approach |
|---|---|
| **Max pool size** | Ramp VUs beyond pool size; measure wait time and connection timeout errors |
| **Min idle connections** | Start test after idle period; measure cold-start latency vs pre-warmed |
| **Connection timeout** | Set aggressive timeout; verify graceful degradation when pool exhausted |
| **Idle timeout / eviction** | Run soak test; verify idle connections get recycled without errors |
| **Leak detection** | Run soak test; monitor active connection count — it should stabilize, not grow |

### Diagnosing Pool Exhaustion

```
Symptoms:
- Response times spike suddenly at a specific VU count
- Errors: "Connection pool exhausted", "Timeout waiting for idle object"
- DB shows fewer active connections than expected

Root causes:
1. Pool max size too small for concurrency
2. Long-running queries hold connections
3. Connection leak — app code doesn't close connections in finally/catch blocks
4. N+1 queries — each request opens multiple connections sequentially
```

### Key Metrics to Monitor

| Metric | Source | Warning Threshold |
|---|---|---|
| Active connections | HikariCP metrics, PgBouncer stats | > 80% of max pool size |
| Pending connection requests | Connection pool metrics | > 0 sustained |
| Connection wait time | Connection pool metrics | > 100ms |
| Connection creation rate | Pool metrics | High rate = connections not being reused |
| DB `max_connections` usage | `pg_stat_activity`, MySQL `SHOW STATUS` | > 80% of server limit |

---

## Query Performance Under Concurrency

Queries that perform well at 1 VU often degrade at 100 VUs due to lock contention, buffer pool pressure, and I/O saturation.

### Testing Approach

1. **Baseline single-query latency** — run the query once, capture execution plan.
2. **Ramp concurrent query execution** — 1, 5, 10, 25, 50, 100 concurrent threads.
3. **Monitor per-step**: query latency (p50/p95/p99), lock waits, buffer cache hit ratio, disk I/O.
4. **Identify the knee point** — the concurrency level where latency starts climbing non-linearly.

### Slow Query Detection During Load Tests

**PostgreSQL:**
```sql
-- Enable slow query logging
ALTER SYSTEM SET log_min_duration_statement = 100;  -- Log queries > 100ms
SELECT pg_reload_conf();

-- Check during test
SELECT query, calls, mean_exec_time, stddev_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 20;
```

**MySQL:**
```sql
-- Enable slow query log
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.1;  -- 100ms

-- Check during test
SELECT * FROM mysql.slow_log ORDER BY start_time DESC LIMIT 20;
```

---

## Read Replica Lag Testing

For read-replica architectures, verify that replication lag doesn't cause stale reads under write-heavy load.

### Test Pattern

1. **Write a record** with a unique marker (timestamp + test ID) to the primary.
2. **Immediately read** the same record from the replica.
3. **Measure the delay** until the record appears on the replica.
4. **Track lag over time** — it should stabilize, not grow.

### What Replication Lag Breaks

- User creates an order → immediately views "My Orders" → order is missing (stale read).
- Cache invalidation depends on DB triggers → replica lag delays invalidation.
- Consistency checks fail under load if reading from replica.

---

## Deadlock and Lock Contention Testing

### Test Pattern

1. Create a scenario where multiple VUs update overlapping rows (e.g., same account balance).
2. Ramp concurrency and monitor for deadlock errors.
3. Verify the application handles deadlocks gracefully (retry logic, not crash).

### Monitoring Lock Contention

**PostgreSQL:**
```sql
SELECT blocked.pid, blocked.query AS blocked_query,
       blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_locks bl ON bl.pid = blocked.pid
JOIN pg_locks blk ON blk.locktype = bl.locktype
  AND blk.relation = bl.relation AND blk.pid != bl.pid
JOIN pg_stat_activity blocking ON blocking.pid = blk.pid
WHERE NOT bl.granted;
```

---

## Database Testing Checklist

- [ ] Dedicated test user/schema created (not production credentials)
- [ ] Connection pool configuration documented and parameterized
- [ ] Slow query logging enabled before test start
- [ ] Read vs write ratio matches production workload
- [ ] Test data volume matches production scale (row counts, index sizes)
- [ ] Lock contention and deadlock monitoring active
- [ ] Replication lag monitoring active (if using read replicas)
- [ ] Connection pool metrics exposed and dashboarded
- [ ] Cleanup script ready for test data removal post-test
