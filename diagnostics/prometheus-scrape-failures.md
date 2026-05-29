---
title: Prometheus Scrape Failures / Targets Down
kind: howto
tags: [prometheus, scrape, targets, up, exporter, observability]
applies_to: [manager]
---

# Prometheus Scrape Failures / Targets Down

Use when metrics for a target stop appearing, panels go blank, or
`up == 0`. Prometheus polls each target on an interval; a scrape can fail
at connect, at HTTP, at parse, or by exceeding the timeout/sample limit.
**The Targets page (`/targets`) shows the exact `lastError` per target —
read it first.**

| `up`/error | Probable cause |
|---|---|
| `up == 0`, connection refused | exporter down / wrong port |
| `up == 0`, context deadline exceeded | scrape slower than `scrape_timeout` |
| `up == 0`, 401/403 | auth/TLS required by the target |
| `up == 1` but metric missing | metric renamed/removed, or relabel dropped it |
| `sample limit exceeded` | target exposes more series than `sample_limit` |

## Step 1 — Read the target state

```promql
up == 0                                   # which targets are down
# Scrape duration vs timeout (the slow-scrape case)
scrape_duration_seconds > 1
```

In the UI, `/targets` (or `/api/v1/targets`) shows `health`, `lastError`,
and `lastScrape` per endpoint — the `lastError` string is the diagnosis.

## Step 2 — Reproduce the scrape by hand

```bash
# From the Prometheus host/pod, hit the target's metrics endpoint exactly
curl -sS -m 5 -o /dev/null -w '%{http_code} %{time_total}s\n' http://<target>:<port>/metrics
curl -sS http://<target>:<port>/metrics | head        # does it parse? right format?
```

- connection refused / timeout → target down or unreachable from
  Prometheus's network (see `diagnostics/network-connectivity.md`).
- slow (`time_total` > `scrape_timeout`) → the exporter is slow (heavy
  collectors, big `/metrics`); raise timeout or lighten the exporter.
- 401/403 → the endpoint needs auth/TLS the scrape config lacks.

## Step 3 — Config / discovery / relabel

```bash
# Reload after edits and check config validity
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload      # if --web.enable-lifecycle
```

A target that never appears at all is usually service-discovery or a
`relabel_configs` rule dropping it (`action: drop`/`keep` mismatch). A
metric present at `/metrics` but missing in Prometheus is a
`metric_relabel_configs` drop.

## Step 4 — Limits

`sample_limit exceeded` means the target now exposes more series than the
configured cap (often cardinality growth — see
`diagnostics/prometheus-high-cardinality.md`). Fix the target's
cardinality rather than just raising the limit.

## Decision tree

| Signal | Action |
|---|---|
| `up==0` refused | start/fix exporter; correct port; check firewall |
| `up==0` deadline exceeded | raise `scrape_timeout` or lighten exporter |
| `up==0` 401/403 | add auth/TLS to the scrape config |
| target never appears | fix service discovery / relabel keep-drop rules |
| `up==1`, metric missing | metric renamed or dropped by metric_relabel |
| `sample_limit exceeded` | cut target cardinality — high-cardinality playbook |

## References

- [Prometheus — Configuration & relabeling](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)
- [promtool — Prometheus docs](https://prometheus.io/docs/prometheus/latest/command-line/promtool/)
- vault: `diagnostics/prometheus-high-cardinality.md`, `diagnostics/network-connectivity.md`, `systems/observability-stack/prometheus.md`
