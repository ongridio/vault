---
title: Tempo — Missing Traces / Broken Span Trees
kind: howto
tags: [tempo, traces, otlp, spans, sampling, context-propagation, observability]
applies_to: [manager]
---

# Tempo — Missing Traces / Broken Span Trees

Use when traces don't show up in Tempo, a trace has gaps (a service's
spans missing), or trace IDs don't stitch across services. **Trace data
fails at three places: the app didn't emit spans, the collector dropped
them, or context wasn't propagated across the hop.** A "missing service"
in a trace is almost always propagation, not Tempo.

| Symptom | Probable cause |
|---|---|
| No traces at all | exporter misconfigured / OTLP endpoint wrong / collector down |
| Trace exists but a service is absent | context (traceparent) not propagated to it |
| Only some traces appear | head sampling dropping them (expected) |
| Spans arrive but query empty | wrong time range / trace ID / tenant |

## Step 1 — Is anything arriving at the collector/Tempo?

```bash
# Tempo / OTel collector ingest metrics
curl -sS http://<tempo>:3200/metrics | grep -E 'tempo_distributor_spans_received_total|tempo_discarded_spans_total'
# Collector (otelcol) receiver/exporter counts
curl -sS http://<otelcol>:8888/metrics | grep -E 'receiver_accepted_spans|exporter_sent_spans|refused'
```

`spans_received_total` flat at zero = nothing is being sent (app/exporter
problem). `discarded`/`refused` rising = Tempo/collector is dropping
(limits, bad data, backpressure).

## Step 2 — App export path

Check the app's OTLP exporter config: endpoint host:port (4317 gRPC /
4318 HTTP), TLS/plaintext match, and that the SDK is actually installed
and started. A common miss: exporter points at the wrong collector
address, or the collector isn't reachable from the app's network.

```bash
# From the app's network, can it reach the OTLP endpoint?
nc -vz <collector> 4317
```

## Step 3 — The "missing middle service" = propagation

If service A → B → C and B's spans are missing from the trace, B isn't
propagating the `traceparent` (W3C Trace Context) header — it started a
*new* trace instead of continuing A's. Causes: B's framework
instrumentation missing, a proxy/gateway stripping headers, or mixed
propagation formats (B3 vs W3C). Verify the inbound request to B carries
`traceparent` and B's SDK reads it.

## Step 4 — Sampling vs loss

```text
Head sampling (e.g. 10%) intentionally keeps 1 in N traces — "missing"
traces may simply be unsampled. Tail sampling keeps error/slow traces.
Confirm the sampling policy before chasing a "drop".
```

If you need a specific trace, ensure the policy keeps errors/slow, or
raise the sample rate temporarily to reproduce.

## Decision tree

| Signal | Action |
|---|---|
| `spans_received`==0 | app exporter wrong/off; fix OTLP endpoint, verify SDK started |
| received but `discarded` rising | Tempo limits/backpressure — scale, raise limits |
| a service missing from trace | context propagation — fix instrumentation / header passthrough |
| only some traces present | sampling (expected) — adjust policy / sample rate |
| spans sent but query empty | wrong time range / trace ID / tenant in query |

## References

- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Grafana Tempo — documentation](https://grafana.com/docs/tempo/latest/)
- [OpenTelemetry Collector — troubleshooting](https://opentelemetry.io/docs/collector/troubleshooting/)
- vault: `diagnostics/high-latency-p99.md`, `diagnostics/network-connectivity.md`, `systems/observability-stack/tempo.md`
