---
title: Prometheus High Cardinality / TSDB Memory Blowup
kind: howto
tags: [prometheus, cardinality, tsdb, memory, labels, observability]
applies_to: [manager]
---

# Prometheus High Cardinality / TSDB Memory Blowup

Use when Prometheus RAM balloons, ingestion lags, queries time out, or it
OOMs. The usual root is **cardinality**: a label with unbounded values
(user id, request id, full URL, pod hash) multiplies into millions of
series. **Find the offending metric+label before adding RAM — RAM only
delays the wall.**

| Symptom | Probable cause |
|---|---|
| RSS climbs steadily, then OOM | active-series growth (cardinality explosion) |
| One target/job dominates series | a metric with a high-cardinality label |
| Spikes right after a deploy | new label added (id/uuid/path) to a hot metric |
| Slow queries + high churn | rapid series churn (labels that change every scrape) |

## Step 1 — Total series + worst offenders

```promql
# Total active series (the number that matters)
prometheus_tsdb_head_series
# Top metrics by series count
topk(20, count by (__name__)({__name__=~".+"}))
# Series per job/target — find the noisy source
topk(20, count by (job) ({__name__=~".+"}))
```

(The `/api/v1/status/tsdb` endpoint also lists top series by metric and
by label — fastest way to see the offender.)

## Step 2 — Find the exploding label

```promql
# For the suspect metric, which label has the most values?
count(count by (label_name) (suspect_metric))
# e.g. a `path` or `user_id` label with thousands of distinct values
```

A label whose distinct-value count is in the thousands+ (and grows with
traffic) is the explosion. URLs with IDs, raw user/request identifiers,
and unbounded enum-ish labels are the classic culprits.

## Step 3 — Churn (high turnover even at modest count)

```promql
rate(prometheus_tsdb_head_series_created_total[5m])   # new series/sec
```

High creation rate = series churn (labels that change every scrape, like
a timestamp or pod-restart hash). Churn hurts as much as raw count —
it bloats the head block and the index.

## Step 4 — Fix at the source, then reclaim

- **Drop/relabel the bad label** at scrape time (`metric_relabel_configs`
  → `labeldrop`/`drop`) so the high-cardinality dimension never enters
  the TSDB.
- **Fix the exporter/app** to not put unbounded values in labels (put
  them in logs/traces instead — that's what Loki/Tempo are for).
- Only after capping cardinality, size RAM/retention. Adding RAM to an
  uncapped explosion just moves the OOM later.

## Decision tree

| Signal | Action |
|---|---|
| One metric dominates series | relabel/drop its high-cardinality label at scrape |
| One job dominates | fix that exporter/app; scope its labels |
| High series-created rate | kill churny labels (ids/hashes/timestamps in labels) |
| Bounded but huge legitimately | shard Prometheus / use recording rules + downsample |
| OOM with capped cardinality | now size RAM/retention appropriately |

## References

- [Prometheus — Cardinality & label best practices](https://prometheus.io/docs/practices/naming/#labels)
- [Cardinality is key — Robust Perception](https://www.robustperception.io/cardinality-is-key/)
- [metric_relabel_configs — Prometheus config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#metric_relabel_configs)
- vault: `diagnostics/oom-killed.md`, `systems/observability-stack/prometheus.md`
