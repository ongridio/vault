---
title: Database Slow Queries & Lock Contention
kind: howto
tags: [database, slow-query, locks, blocking, index, mysql, postgres]
applies_to: [manager]
---

# Database Slow Queries & Lock Contention

Use when DB-backed requests are slow or timing out and the DB host isn't
obviously saturated. **Separate "a query is slow" (missing index, bad
plan, big scan) from "a query is blocked" (waiting on a lock held by
another transaction).** They look identical from the app (slow) but the
DB tells them apart precisely.

| Symptom | Probable cause |
|---|---|
| One query slow, others fine | missing index / full scan / bad plan |
| Many queries slow at once | lock contention / a long-held lock blocking them |
| Slow only at peak | plan flips / cache cold / connection queueing |
| `Lock wait timeout` / `deadlock detected` | blocking / lock cycle |

## Step 1 — What's running + what's blocked

```sql
-- PostgreSQL: active queries, who is blocked, and by whom
SELECT pid, state, wait_event_type, now()-query_start AS dur, left(query,80)
FROM pg_stat_activity WHERE state!='idle' ORDER BY dur DESC;
SELECT * FROM pg_locks WHERE NOT granted;          -- waiting locks
-- MySQL
SHOW PROCESSLIST;                                   -- long-running / Locked states
SELECT * FROM performance_schema.data_lock_waits;   -- who waits on whom
```

`wait_event_type = Lock` (PG) or `Locked`/`Waiting for ... lock` (MySQL)
= a blocked query (Step 3). A long-running query with high duration but
*running* (not waiting) = a slow query (Step 2).

## Step 2 — Slow query: read the plan

```sql
EXPLAIN ANALYZE <the slow query>;     -- actual rows, timing, scan type
```

`Seq Scan`/full table scan on a big table, a huge `rows` estimate vs
actual mismatch (stale stats), or a missing index on the filter/join
column = the cause. Add the index, update statistics (`ANALYZE`), or
rewrite the query. Enable the slow-query log to catch repeat offenders.

## Step 3 — Lock contention: find the blocker

```sql
-- PostgreSQL: the blocking PID
SELECT blocked.pid AS blocked, blocking.pid AS blocking,
       left(blocking_act.query,80) AS blocking_query
FROM pg_locks blocked
JOIN pg_locks blocking ON blocking.locktype=blocked.locktype
  AND blocking.granted AND NOT blocked.granted
JOIN pg_stat_activity blocking_act ON blocking_act.pid=blocking.pid;
```

The blocker is usually a long-running transaction, an `idle in
transaction` session that never committed, or a schema change (DDL)
taking a strong lock. Kill/commit the blocker (`pg_terminate_backend`)
to release the queue, then fix why it held the lock.

## Step 4 — Structural fixes

- **Idle-in-transaction blockers**: app opened a tx and didn't commit —
  fix the code (see `diagnostics/db-connection-pool-exhaustion.md` for
  the related leak).
- **DDL under load**: run migrations with lock timeouts / online schema
  change tools so they don't block the world.
- **Hot-row contention**: shorten transactions, reduce the lock scope,
  or redesign the access pattern.

## Decision tree

| Signal | Action |
|---|---|
| one query slow, full scan | add index / update stats / rewrite query |
| query waiting on Lock | find + clear the blocker; fix why it held it |
| `idle in transaction` blocker | fix app commit/rollback path |
| deadlock detected | fix transaction lock ordering; add retries |
| DDL blocking everything | use lock_timeout + online migration tooling |

## References

- [PostgreSQL — Monitoring (pg_stat_activity, pg_locks)](https://www.postgresql.org/docs/current/monitoring-stats.html)
- [Use The Index, Luke — SQL indexing](https://use-the-index-luke.com/)
- vault: `diagnostics/db-connection-pool-exhaustion.md`, `diagnostics/high-latency-p99.md`, `diagnostics/db-replication-lag.md`
