---
title: Service Latency / p99 Spike
kind: howto
tags: [latency, p99, tail-latency, traces, tempo, promql, performance]
applies_to: [edge, manager]
---

# Service Latency / p99 Spike

Use when a service's tail latency (p95/p99) climbs while the median
barely moves — the classic "most requests are fine, some are awful"
shape. **Separate "everything got slower" (median moves too) from
"the tail got fatter" (only p99 moves)**: they have different causes.

| Shape | Probable cause class |
|---|---|
| Median + p99 both up uniformly | global saturation (CPU/IO/DB) or a slow dependency |
| Only p99 up, median flat | GC pauses, lock contention, a slow shard/instance, retries |
| p99 up in steps tied to deploys | regression in new code path |
| p99 up only under load | queueing / thread-pool / connection-pool exhaustion |

## Step 1 — Quantify it from metrics (histogram, not average)

Averages hide tails. Use the request-duration histogram:

```promql
# p99 over 5m, by route
histogram_quantile(0.99, sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))
# Compare p50 vs p99 to see if it's tail-only
histogram_quantile(0.50, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))
```

If the histogram's top finite bucket is saturating (`+Inf` bucket
growing), your real p99 is *above* the largest bucket — the number is a
floor, the truth is worse.

## Step 2 — Localize with traces (which span owns the time)

The platform ships Tempo. Pull slow exemplars and read the span tree:

- Filter traces by service + `duration > <p99 threshold>`.
- In the slow trace, find the span with the largest *self* time (not
  just total) — that's where the latency is born, not just passed
  through.

Common span verdicts:
- One downstream call dominates → dependency problem (DB, upstream API)
- Gap *between* spans (no child running) → queueing / GC / scheduler
- Many small serial calls → N+1 / fan-out without concurrency

## Step 3 — Saturation check on the slow node (USE)

```bash
mpstat -P ALL 1 3                    # CPU run vs iowait
vmstat 1 5                           # r (runqueue), b (blocked), si/so (swap)
ss -tin state established | wc -l    # connection count
```

A high run-queue (`r` > cores) with the service pinned = CPU saturation
→ p99 inflates from scheduling delay. High `b`/iowait → see
`diagnostics/high-load-low-cpu.md`.

## Step 4 — The usual tail-latency culprits

```bash
# GC pauses (JVM example) — long pauses == p99 cliffs
jstat -gcutil <pid> 1000 5
# Thread/connection pool exhaustion — requests waiting for a worker
ss -tlnp | grep <svc-port>           # Recv-Q growing = accept backlog filling
# A single bad instance dragging the aggregate
#   compare p99 per instance label in PromQL before blaming the service
histogram_quantile(0.99, sum by (le, instance) (rate(http_request_duration_seconds_bucket[5m])))
```

If one `instance` is far worse than its peers, evict/restart it and the
aggregate p99 recovers — don't redesign the service for one sick pod.

## Step 5 — Retries amplifying the tail

A timeout + retry policy turns one slow dependency into 2–3× load right
when it's already struggling (retry storm). Check client retry/timeout
config and whether the latency rises *with* the dependency's error rate
(cross-reference `diagnostics/error-rate-5xx.md`).

## Decision tree

| Signal | Action |
|---|---|
| Median moves with p99 | global saturation — find the saturated resource (USE) |
| One downstream span dominates | fix/scale the dependency; add a circuit breaker |
| Inter-span gaps, no child running | GC / lock / runqueue — profile the process |
| One instance worse than peers | drain/restart that instance |
| p99 tracks dependency error rate | cap retries, add backoff + jitter, circuit-break |
| `+Inf` bucket growing | true p99 above largest bucket; add finer buckets |

## References

- [Latency: The 99 Percentile — useful percentiles](https://igor.io/latency/)
- [The Tail at Scale — Dean & Barroso (CACM)](https://research.google/pubs/the-tail-at-scale/)
- [histogram_quantile — Prometheus docs](https://prometheus.io/docs/prometheus/latest/querying/functions/#histogram_quantile)
- vault: `diagnostics/error-rate-5xx.md`, `diagnostics/high-load-low-cpu.md`, `systems/observability-stack/tempo.md`, `systems/observability-stack/prometheus.md`
