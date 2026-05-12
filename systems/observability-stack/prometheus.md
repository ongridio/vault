---
title: Prometheus — How It Actually Works
tags: [prometheus, metrics, tsdb, observability]
---

# Prometheus — How It Actually Works

The mental model in one breath: **Prometheus periodically scrapes
exposition-format text endpoints over HTTP, appends each sample to a
local TSDB, and serves PromQL queries against that TSDB.** Everything
else is plumbing around that loop.

## The scrape loop

```
prometheus.yaml (scrape_configs)
   │
   ▼
service discovery  →  list of targets
   │
   ▼
HTTP GET /metrics every scrape_interval
   │
   ▼
parse exposition format  →  samples (name, labels, value, timestamp)
   │
   ▼
write to head block (in-memory)
   │
   ▼ (every 2h)
flush head → on-disk block under data/
```

`scrape_interval` (typ. 15s) is the resolution. Halving it doubles
storage. Quadrupling it loses signal on short-lived issues. 15s is the
universal default for a reason.

## TSDB shape

```
data/
├── 01HJMG.../   ← 2-hour block (immutable)
│   ├── chunks/
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01HJMH.../
├── wal/         ← write-ahead log for in-memory data
└── chunks_head/
```

Old blocks are compacted into larger ones (`block_compaction`),
eventually consolidated into 24h+ ranges. Retention (`--storage.tsdb.retention.time`)
deletes blocks past the limit; **it does not surgically delete series**
— a series sticks around until every block containing it ages out.

## Cardinality — the dominant cost

A "series" is one unique (`name`, `label1=val1`, ..., `labelN=valN`)
combination. Storage and query cost scale with **active series count**,
not with sample volume. Rules of thumb:

- 1 sample per series per scrape = ~1 byte after compression
- 100k active series at 15s = ~3 GB / day
- 1M active series at 15s = ~30 GB / day, query latency degrades sharply

Killer labels (avoid):

- `user_id`, `email`, `request_id`, `trace_id` — unbounded
- Anything derived from URL paths without normalization (e.g.
  `/api/users/123/orders/456` → use `/api/users/:id/orders/:id`)
- Container UIDs without TTL on metric expiry

Find the worst offenders:

```promql
topk(20, count by (__name__) ({__name__=~".+"}))
topk(20, count by (__name__, label) ({__name__=~".+"}))
```

Or, on the server, the `/api/v1/status/tsdb` endpoint dumps the top
series by label cardinality.

## PromQL — the four operations that cover 90%

| Op | What it does | Example |
|----|--------------|---------|
| Instant vector selector | All series matching label filters, at the current instant | `up{job="api"}` |
| Range vector selector | A window of samples per series | `http_requests_total[5m]` |
| `rate()` / `irate()` | Per-second rate of a counter | `rate(http_requests_total[5m])` |
| Aggregation (`sum by` / `avg by` / `topk`) | Collapse series | `sum by (job) (rate(http_requests_total[5m]))` |

Two traps:

- **`rate()` on a gauge** silently produces garbage — `rate` is for
  counters
- **`rate()[1m]` on a 15s scrape** has only 4 samples and is jittery;
  use `[5m]` minimum unless you really need fast response

## Recording rules vs. alerting rules

- **Recording rules** materialize an expression on each evaluation
  interval, so dashboards / repeated alerts don't re-compute the same
  thing. Conventionally namespaced `level:metric:operations`.
- **Alerting rules** evaluate an expression; if it produces samples for
  `for:` duration, fire an alert.

Recording rules cost storage but save query CPU; for any expression
that more than two alerts or dashboards depend on, prefer the recording
rule.

## Federation vs. remote-write

Two scaling patterns:

- **Hierarchical federation** — an upper Prometheus scrapes
  `/federate` from a lower one to pull a subset. Simple; loses
  high-resolution data.
- **Remote-write** — a Prometheus pushes samples to a long-term store
  (Cortex, Mimir, Thanos receive, VictoriaMetrics). Cleaner for
  multi-cluster + long retention.

Ongrid's reference architecture: edge Prometheus scrapes locally,
remote-writes to a central Prometheus on the manager. See ADR-009.

## High-leverage operational metrics

Every Prometheus deploy should alert on at least:

- `up == 0` for any production target → scrape failed
- `prometheus_tsdb_head_series` → cardinality growth
- `rate(prometheus_tsdb_compactions_failed_total[5m]) > 0`
- `prometheus_remote_storage_samples_pending` rising → remote-write
  backlog
- `process_resident_memory_bytes` near the container memory limit
