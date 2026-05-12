---
title: Tempo — How It Actually Works
tags: [tempo, traces, observability]
---

# Tempo — How It Actually Works

Tempo stores distributed traces. The big design choice: **no index on
span attributes**. You look up traces by `trace_id` (cheap) or use
TraceQL (which scans blocks). Trade-off: ingest is cheap; broad
attribute search is expensive without supporting metrics.

## What a trace is

```
Trace = tree of Spans
  Span = (name, start, duration, attributes, parent_span_id, trace_id)
```

A request enters the system, generates a `trace_id`, and every service
it touches emits spans tagged with that `trace_id`. The tree is
reconstructed from `parent_span_id` linkages at query time.

## Architecture

- **distributor** — accepts spans (OTLP, Jaeger, Zipkin); forwards to
  ingester
- **ingester** — buffers spans, builds blocks, flushes to object store
- **compactor** — merges small blocks
- **querier** — pulls blocks by `trace_id`; executes TraceQL
- **query-frontend** — splits TraceQL across shards; caches results
- **metrics-generator** (optional) — derives RED metrics from spans,
  remote-writes to Prometheus

## The lookup-vs-search distinction

| Operation | Cost | When |
|-----------|------|------|
| Get a trace by `trace_id` | O(1) — direct block read | You have a trace ID from logs / a metric exemplar |
| TraceQL across attributes | O(N) — scans blocks | Cross-trace investigation; expensive |

This is why the canonical flow is **alert → metric exemplar → trace ID
→ Tempo lookup → drill into spans**, not "browse Tempo for slow
requests". Use Tempo's `metrics-generator` to derive
`traces_spanmetrics_*` counters, alert on those, and pivot via
exemplars.

## Sampling

100% sampling above a few hundred requests/sec is rarely affordable.
Strategies:

- **Head sampling** — decide at the root span. Cheap; loses tail interest.
- **Tail sampling** — buffer the whole trace, decide based on the
  finished tree (errors, latency, slow path). More expensive; far more
  useful for incidents.
- **Hybrid** — head sample at low base rate, tail-sample for "always
  keep errors / slow".

Tail sampling lives at the **OpenTelemetry Collector** in front of
Tempo, not inside Tempo itself.

## TraceQL — the query language

```traceql
{ resource.service.name="api" && duration > 500ms }
{ status = error }
{ span.http.status_code = 500 }
{ resource.service.name="api" } && { span.kind = client && duration > 1s }
```

Patterns:

- Selectors `{ ... }` filter spans
- `&&` between selectors filters trace-scoped: both selectors must
  match *somewhere in the same trace*
- Aggregate with `count()`, `avg(duration)`, etc.

Limitations vs. PromQL/LogQL: less mature, fewer aggregation operators.
For most "what's slow on this route" questions, derive RED metrics in
Prometheus via the generator and query there.

## Correlation glue

The whole point of running Tempo:

- **Metric exemplars** — a Prometheus sample tagged with a `trace_id`
  ("the slow request behind this p99 spike")
- **Log labels** — every log line includes `trace_id` so Loki queries
  pivot directly into Tempo
- **Service map** — generator emits `traces_service_graph_request_*`
  metrics; Grafana renders the dependency graph from those

Wire all three and incident investigation feels qualitatively
different.

## Operational signals

- `tempo_distributor_spans_received_total` — ingest volume
- `tempo_ingester_blocks_flushed_total` — block flush rate
- `tempo_request_duration_seconds` p99 by route — query latency
- `tempo_metrics_generator_processor_*` — generator health (it's a
  common silent-failure point)
