---
title: Observability Primer
tags: [observability, metrics, logs, traces]
---

# Observability Primer

Ongrid's observability story is built on three signal types and a fourth
relationship layer. Understanding what each is good for — and what each
is *bad* for — is the difference between a fast investigation and a
shotgun debug session.

## The three signals

### Metrics

Numeric samples over time. Cheap to store, cheap to aggregate, expensive
to gain cardinality on.

- **Good for:** trend dashboards, alerting (PromQL), capacity planning
- **Bad for:** "why did *this specific request* fail" — metrics aggregate
  away identity
- **Stack:** Prometheus (TSDB) — see `systems/observability-stack/prometheus.md`

The cost knob is **cardinality**: every unique label combination is a
new time series. A `user_id` label is almost always a mistake.

### Logs

Discrete events with text payload. Identity-preserving (each line has
a timestamp and context), but expensive to search at scale.

- **Good for:** "what did this service do at 14:03", error patterns,
  forensic reconstruction
- **Bad for:** trend questions, alerting on volume changes (do that
  with metrics over a log-derived counter)
- **Stack:** Loki — label-indexed, full-text via `|=` / `|~` filters

The cost knob is **label cardinality on the index**. Body text is cheap;
high-cardinality labels (`trace_id`, `user_id`) blow up the index.

### Traces

Per-request causality graphs. Each request is a tree of spans showing
service-to-service calls, with timing and attributes.

- **Good for:** latency root cause, dependency mapping, fan-out problems
- **Bad for:** baseline behavior — you only see what was sampled
- **Stack:** Tempo — see `systems/observability-stack/tempo.md`

The cost knob is **sampling rate**. 100% sampling is rarely affordable
above a few hundred req/s; tail-sampling on error / latency is the
production pattern.

## The fourth dimension: correlation

Signals on their own are useful; signals **correlated** are diagnostic.
Ongrid's manager links them by:

- **`trace_id` on logs** — emit it in every log line so `trace_id` from
  Tempo can pivot directly to Loki
- **`exemplars` on metrics** — Prometheus exemplars stamp a `trace_id`
  on a metric sample, letting a slow request alert link to its trace
- **`edge_id` / `host` everywhere** — the topology dimension joins
  signals back to the device that produced them

When an alert fires, the agent's first move is to widen from the metric
that triggered the rule to the matching trace + logs around the same
window. The `correlate_incident` skill does this automatically.

## When to reach for which

| Question shape | Start with |
|----------------|-----------|
| "Is X getting worse?" | Metrics |
| "Why did the 14:03 request fail?" | Logs filtered by `trace_id` |
| "Which dependency is slow?" | Traces |
| "How are these related?" | All three, joined on `trace_id` / `edge_id` |

## See also

- `concepts/alerting.md` — how alert rules use metrics
- `concepts/incident-response.md` — the workflow that consumes correlation
- `systems/observability-stack/*` — how each stack actually works
