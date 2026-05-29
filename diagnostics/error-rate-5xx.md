---
title: HTTP 5xx Error-Rate Spike
kind: howto
tags: [http, 5xx, errors, error-rate, logs, loki, traces, tempo]
applies_to: [edge, manager]
---

# HTTP 5xx Error-Rate Spike

Use when the 5xx ratio for a service jumps. **First split 5xx by code
and by endpoint** — 500 (app bug / unhandled exception), 502/504 (the
proxy can't reach or times out on the upstream), and 503 (overload /
shedding) point at completely different layers.

| Code | Who emitted it | Probable cause class |
|---|---|---|
| 500 | the app itself | unhandled exception, bad deploy, dependency error |
| 502 Bad Gateway | reverse proxy | upstream crashed / refused / spoke garbage |
| 503 Service Unavailable | proxy or app | overload, no healthy backends, deliberate shed |
| 504 Gateway Timeout | reverse proxy | upstream too slow (see latency playbook) |

## Step 1 — Quantify and split the rate

```promql
# Overall 5xx ratio
sum(rate(http_requests_total{code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
# By code + route — find the concentration
sum by (code, route) (rate(http_requests_total{code=~"5.."}[5m]))
```

A spike concentrated on one route/code is a targeted bug; a broad rise
across all routes is infrastructure (dependency, saturation, deploy).

## Step 2 — Read the actual errors (logs / Loki)

The platform ships Loki. Pull the error lines for the window:

```logql
{service="checkout"} |= "level=error" | json | line_format "{{.msg}}"
# 502/504 live in the proxy log, not the app log
{service="nginx"} | json | status=~"5.."
```

What to extract: the exception/stack class, the downstream host/error
("connection refused", "context deadline exceeded", "pool exhausted"),
and whether it started at a sharp edge (deploy) or ramped (saturation).

## Step 3 — Correlate with the timeline

```bash
# Did it start exactly at a deploy / config change / restart?
#   compare the spike's start to release + container-start times
docker ps --format '{{.Names}} {{.Status}}' | grep -i <svc>
```

A 5xx spike whose start aligns to the second with a deploy is a
regression — roll back first, diagnose after. A ramp that tracks traffic
or a resource metric is saturation.

## Step 4 — Follow one failing request end-to-end (traces / Tempo)

Pull a trace for a failed request (error tag / 5xx status) and find the
span that errored. The failing span's attributes usually name the root:
DB error, upstream 5xx, timeout, or a panic. This converts "service X
returns 500" into "service X's call to Y times out at 1s".

## Step 5 — Dependency + saturation checks

```bash
# Is a downstream dependency itself erroring? (cascade)
#   check its own 5xx/error metrics + health endpoint
curl -sS -o /dev/null -w '%{http_code}\n' http://<dep>/healthz
# Connection-pool / FD exhaustion surfaces as 5xx under load
ss -s; cat /proc/<pid>/limits | grep 'open files'
```

If the dependency is the one actually failing, fix there and add a
circuit breaker here so its outage doesn't take you down with it.

## Decision tree

| Signal | Action |
|---|---|
| Spike starts at a deploy | roll back the release, then root-cause |
| 500 on one route, stack in logs | app bug — patch / disable that path |
| 502 + "connection refused" upstream | upstream is down/crashing — restart/scale it |
| 504 + slow upstream spans | latency problem — see `diagnostics/high-latency-p99.md` |
| 503 + high load / shed logs | overload — scale out or shed earlier; check pools |
| Downstream dep erroring too | cascade — circuit-break + fix the dependency |

## References

- [Google SRE Workbook — Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)
- [LogQL — Loki query language](https://grafana.com/docs/loki/latest/query/)
- [Circuit Breaker pattern — Fowler](https://martinfowler.com/bliki/CircuitBreaker.html)
- vault: `diagnostics/high-latency-p99.md`, `diagnostics/service-down-cn.md`, `diagnostics/fd-exhaustion.md`, `systems/observability-stack/loki.md`, `systems/observability-stack/tempo.md`
