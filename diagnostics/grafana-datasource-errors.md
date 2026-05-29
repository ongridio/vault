---
title: Grafana — Datasource Errors & Empty Panels
kind: howto
tags: [grafana, datasource, panels, dashboards, proxy, observability]
applies_to: [manager]
---

# Grafana — Datasource Errors & Empty Panels

Use when dashboards show "No data", a red datasource error, or panels
error after they worked yesterday. **Grafana is rarely the problem — it's
a thin query layer.** Split: can Grafana reach the datasource (Prom/Loki/
Tempo) at all, vs. can it reach it but the query/time-range returns
nothing.

| Symptom | Probable cause |
|---|---|
| Red "datasource error" / `bad gateway` | Grafana can't reach the backend (URL/network/auth) |
| Panel "No data", others fine | query/label/time-range returns nothing |
| All panels empty after a change | datasource URL/UID changed; dashboard points at a dead UID |
| Intermittent timeouts | backend slow (high-cardinality query, overloaded TSDB) |

## Step 1 — Test the datasource itself

In Grafana: the datasource Settings page has a **Save & Test** button —
it does a health probe. Or via API:

```bash
# Datasource health (needs admin auth)
curl -sS -u admin:<pw> http://<grafana>:3000/api/datasources
# Then test the backend the same way Grafana does:
curl -sS http://<prometheus>:9090/-/healthy
curl -sS http://<loki>:3100/ready
```

If the backend health is fine but Grafana's test fails, it's the
datasource *config* (URL, auth, TLS) — Grafana often runs server-side
(proxy) so it must reach the backend over the *server's* network, not
your browser's.

## Step 2 — URL / network / auth from Grafana's perspective

```bash
# From the Grafana container/host:
curl -sS -o /dev/null -w '%{http_code}\n' http://<backend>:<port>/<health-path>
```

Common breakages: datasource URL uses a hostname only resolvable in the
browser (not in the Grafana container), missing/expired auth token, or
TLS verify failing on a self-signed backend. Provisioned datasources
(YAML) overwrite UI edits on restart — check the provisioning file.

## Step 3 — "No data" with a healthy datasource

```text
- Time range: the panel's range has no data (shifted clock? no recent
  samples? — check the raw query in Explore with a wide range).
- The metric/label/stream was renamed or dropped (see the Prom/Loki
  scrape/cardinality playbooks).
- A template variable resolved to empty / a wrong value.
```

Run the panel's exact query in **Explore** to separate "query returns
nothing" from "panel/visualization misconfigured".

## Step 4 — Provisioning / dashboard UID drift

Provisioned dashboards reference a datasource by **UID**. If the
datasource was recreated (new UID), every panel points at a dead UID →
all empty. Re-point the dashboards or pin a stable UID in provisioning.

## Decision tree

| Signal | Action |
|---|---|
| datasource test fails | fix URL/auth/TLS reachable from Grafana's server network |
| backend healthy, Grafana can't reach | container DNS/network/proxy — use in-cluster address |
| one panel empty | run query in Explore; check time range / renamed metric |
| all panels empty after change | datasource UID drift — re-point dashboards/provisioning |
| intermittent timeouts | backend slow — high-cardinality query / overloaded TSDB |

## References

- [Grafana — Data sources](https://grafana.com/docs/grafana/latest/datasources/)
- [Grafana — Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- vault: `diagnostics/prometheus-scrape-failures.md`, `diagnostics/prometheus-high-cardinality.md`, `systems/observability-stack/prometheus.md`
