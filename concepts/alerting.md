---
title: Alerting Model
tags: [alerting, rules, severities]
---

# Alerting Model

Ongrid uses a two-dimensional taxonomy for alert rules, plus a small set
of operational primitives (severity, suppression, runbook). Understanding
the taxonomy is what stops your alert tree from becoming spaghetti.

## The 8 × 6 taxonomy

Every rule is described by **(signal source × trigger mode)**:

**Signal sources (rows).** Where the rule's input data comes from:
metric, log_match, log_volume, trace_latency, trace_error_rate, event,
heartbeat, synthetic. Each has its own evaluator (see `internal/alert/eval/`).

**Trigger modes (columns).** How the input is compared to fire:
threshold, anomaly, change, absence, ratio, predicate.

Most production rules end up in two cells: `metric × threshold` (the
classic PromQL alert) and `log_match × predicate` (regex on a log
stream). The rest exist so you don't have to bend a square peg into a
round hole when a real use case shows up.

## Severities

| Severity | Meaning | Default routing |
|----------|---------|-----------------|
| `critical` | Customer impact happening now | Page + Slack + incident |
| `warning`  | Customer impact imminent | Slack + ticket |
| `info`     | Anomaly worth noting; no action | Slack channel only |

Severity is a routing decision, not a truth claim. A rule firing at
`warning` should still be high signal — if it pages no one and no one
ever looks, delete the rule.

## Suppression and grouping

- **Inhibition** — a downstream rule suppresses an upstream rule when
  the cause already alerts. Example: `device_offline` inhibits
  `cpu_high` for the same edge.
- **Grouping** — alerts on the same dimension (edge, service) collapse
  into one notification within a window.
- **Maintenance windows** — operator-defined silences scoped by label
  selector + time window.

## Runbook linkage

Every rule should set `runbook_url` to a doc in `alerts/`. The link uses
a fragment matching the `rule_key`, e.g.:

```
https://github.com/ongridio/vault/blob/main/alerts/host-metrics.md#cpu_high
```

If a rule has no runbook, the on-call experience is "alert fired, now
read someone's git history to guess what to do." Don't ship that.

## Lifecycle states

- **Pending** — condition holds but `for:` window not yet elapsed
- **Firing** — condition has held through `for:`; notification sent
- **Resolved** — condition no longer holds; resolution notification sent

`for:` exists to filter transients. Tune it long enough to skip blips,
short enough that real incidents still surface in time to act.
