---
title: Database Connection Pool Exhaustion
kind: howto
tags: [database, connection-pool, max-connections, latency, timeout, db]
applies_to: [manager]
---

# Database Connection Pool Exhaustion

Use when requests fail or hang with "too many connections", "pool
timeout / timed out getting connection", or DB latency spikes under load
while the DB's CPU is fine. The bottleneck is **connection slots**, not
the database engine. **Decide which limit you hit**: the client-side
pool (`max_open`) or the server-side ceiling (`max_connections`).

| Symptom | Probable cause |
|---|---|
| App: "timeout acquiring connection" | client pool too small / connections held too long |
| DB: "too many connections" (server) | sum of all clients' pools > server `max_connections` |
| Latency cliff at a load threshold | requests queue waiting for a free connection |
| Connections climb, never released | leak — code path not returning conns to the pool |

## Step 1 — Server side: connections vs ceiling

```sql
-- MySQL
SHOW STATUS LIKE 'Threads_connected'; SHOW VARIABLES LIKE 'max_connections';
SHOW PROCESSLIST;                 -- who holds connections, in what state
-- PostgreSQL
SELECT count(*), state FROM pg_stat_activity GROUP BY state;
SHOW max_connections;
```

`Threads_connected` (or `pg_stat_activity` count) near `max_connections`
= server ceiling hit. A pile of `Sleep`/`idle in transaction`
connections = clients holding slots without using them (often a leak or
a transaction left open).

## Step 2 — Client side: pool config vs concurrency

Check the app's pool settings against reality:
- `max_open_conns` (cap) — if requests > this and each holds a conn for
  a slow query, the rest queue and time out.
- `max_idle_conns` — too low causes churn (open/close per request);
  too high pins server slots.
- `conn_max_lifetime` — recycles conns so a dead DB behind a proxy is
  noticed.

Rule of thumb: total connections = (instances × per-instance max_open).
That sum must stay under the server's `max_connections` with headroom —
a common outage is "scaled to N replicas, N × pool > server limit".

## Step 3 — Leak vs genuine saturation

```sql
-- PostgreSQL: idle-in-transaction is the classic leak signature
SELECT pid, state, now()-state_change AS age, query
FROM pg_stat_activity WHERE state='idle in transaction' ORDER BY age DESC;
```

Connections in `idle in transaction` for a long time = code opened a
transaction and didn't commit/rollback (missing defer/finally). That's a
leak — fix the code path; raising the pool only delays the wall.

## Step 4 — Fixes (in order)

1. **Fix leaks** — ensure every borrow returns (defer Close / finally),
   no long-lived idle transactions.
2. **Right-size pools** — set per-instance `max_open` so the fleet total
   < server `max_connections` with headroom.
3. **Add a connection proxy** (PgBouncer / ProxySQL) to multiplex many
   app connections onto few server connections when you have many small
   clients.
4. Only then consider raising server `max_connections` (each costs RAM).

## Decision tree

| Signal | Action |
|---|---|
| Server at `max_connections`, many idle | clients hold too many — shrink pools / add proxy |
| `idle in transaction` piling up | transaction leak — fix commit/rollback path |
| App pool timeouts, server fine | pool too small OR slow queries holding conns long |
| Broke right after scaling replicas | fleet pool sum > server limit — cap per-instance |
| Conns climb monotonically | leak — borrowed conns not returned |

## References

- [PostgreSQL — Monitoring with pg_stat_activity](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [About Pool Sizing — HikariCP wiki](https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing)
- [PgBouncer — lightweight connection pooler](https://www.pgbouncer.org/)
- vault: `diagnostics/error-rate-5xx.md`, `diagnostics/high-latency-p99.md`, `diagnostics/fd-exhaustion.md`
