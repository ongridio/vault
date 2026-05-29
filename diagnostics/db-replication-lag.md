---
title: Database Replication Lag / Stale Replicas
kind: howto
tags: [database, replication, lag, replica, read-after-write, mysql, postgres]
applies_to: [manager]
---

# Database Replication Lag / Stale Replicas

Use when reads return stale data, "I just saved it but it's gone" bugs
appear, or a replica falls behind / breaks. Replicas apply the primary's
changes asynchronously; under load or with a slow replica, they fall
behind (lag). **Decide: is the replica *lagging* (catching up slowly) or
*broken* (replication stopped)?** And separately, is the app reading a
replica when it needs read-after-write consistency?

| Symptom | Probable cause |
|---|---|
| Reads stale by seconds, grows under load | replication lag (replica can't keep up) |
| "saved then missing" right after write | read-after-write hitting a lagging replica |
| Lag stuck/growing unbounded | replication broken / single-threaded apply bottleneck |
| Replica errored / stopped | conflicting row, disk full, broken position/GTID |

## Step 1 — Measure the lag + replication health

```sql
-- PostgreSQL (on replica)
SELECT now()-pg_last_xact_replay_timestamp() AS replay_lag;
SELECT status, last_msg_receipt_time FROM pg_stat_wal_receiver;
-- MySQL (on replica)
SHOW REPLICA STATUS\G    -- Seconds_Behind_Source, Replica_IO/SQL_Running, Last_Error
```

`Seconds_Behind_Source` (MySQL) / `replay_lag` (PG) quantifies it.
`Replica_IO_Running`/`Replica_SQL_Running` = No, or a `Last_Error`, means
replication is **broken**, not merely lagging (Step 3).

## Step 2 — Lagging: find the bottleneck

Async apply falls behind when: the primary's write rate exceeds the
replica's single-threaded apply speed, the replica is IO/CPU saturated,
a long-running query on the replica blocks apply (PG: `max_standby_*`
conflicts), or the network between them is slow. Check the replica's USE
(CPU/IO — `diagnostics/disk-io-saturation.md`) and whether parallel apply
/ multi-threaded replication is enabled.

## Step 3 — Broken replication

```text
- MySQL Last_Error: a row/DDL that can't apply (duplicate key, missing
  table) — fix the conflict or skip carefully, then resume.
- Disk full on the replica halts apply — free space, resume.
- Lost/!valid binlog position or GTID gap — may require re-seeding the
  replica from a fresh backup.
```

Don't blindly skip errors (it diverges the replica). Understand the
conflicting statement first; re-seed if divergence is suspected.

## Step 4 — App-side consistency (the real bug source)

Read-after-write bugs are usually an **app** issue: it writes to the
primary then immediately reads from a replica that hasn't caught up.
Fixes: route reads that need consistency to the primary, use
"read-your-writes" routing (sticky to primary for N seconds after a
write), or wait for the replica to reach the write's LSN/GTID before
reading.

## Decision tree

| Signal | Action |
|---|---|
| lag grows under write load | replica undersized / single-thread apply — parallelize, scale replica |
| replica IO/CPU saturated | fix replica resources — `diagnostics/disk-io-saturation.md` |
| Replica_*_Running=No / Last_Error | broken — fix conflict / disk / re-seed; then resume |
| "saved then missing" | app routing — read from primary for read-after-write |
| disk full on replica | free space, resume replication |

## References

- [PostgreSQL — Hot Standby & monitoring lag](https://www.postgresql.org/docs/current/hot-standby.html)
- [MySQL — Replication & SHOW REPLICA STATUS](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- vault: `diagnostics/db-slow-queries-locks.md`, `diagnostics/disk-io-saturation.md`, `diagnostics/disk-full-cn.md`
