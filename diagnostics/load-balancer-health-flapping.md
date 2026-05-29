---
title: Load Balancer Health-Check Flapping / Backends Pulled
kind: howto
tags: [load-balancer, health-check, flapping, backend, 502, draining]
applies_to: [edge, manager]
---

# Load Balancer Health-Check Flapping / Backends Pulled

Use when an LB (cloud LB, nginx/HAProxy, k8s Service) marks healthy
backends down — capacity drops, 502/503s appear, or backends flap
in/out. **Decide whether the backends are actually unhealthy or the
health check itself is wrong/too strict** — a bad check turns a healthy
fleet into an outage.

| Symptom | Probable cause |
|---|---|
| All backends marked down at once | health-check path/port wrong, or a shared dependency |
| Backends flap in/out periodically | check interval/timeout too tight; GC/latency spikes |
| 502/503 right after a deploy | new pods failing the check; or no draining on old ones |
| Down only under load | check times out when backend is busy |

## Step 1 — What is the LB checking, and what does the backend return

```bash
# Reproduce the LB's exact check against a backend
curl -sS -m 2 -o /dev/null -w '%{http_code} %{time_total}s\n' http://<backend>:<port><health-path>
# Compare to what the LB expects (port, path, method, expected code, timeout)
```

If the manual check returns the expected code quickly but the LB still
marks it down, the LB's configured port/path/expected-code/timeout is
wrong (or it checks from a network the backend doesn't allow).

## Step 2 — Flapping = timing too tight

LB health checks have `interval`, `timeout`, and `healthy/unhealthy
threshold`. If `timeout` < the backend's p99 under load, every latency
blip fails the check and the backend flaps. Loosen the timeout / raise
the unhealthy threshold so a single slow response doesn't eject a healthy
backend. (Cross-reference `diagnostics/high-latency-p99.md`.)

## Step 3 — Shallow vs deep checks

- A **shallow** check (`/healthz` returns 200 if the process is up) keeps
  the backend in rotation during transient dependency blips — good for
  liveness-style LB checks.
- A **deep** check (hits DB/dependencies) ejects the whole fleet when a
  shared dependency hiccups — turning a dependency blip into a full
  outage. Use deep checks sparingly and with generous thresholds.

## Step 4 — Deploy interplay (draining + readiness)

502s at deploy usually mean: new backends added before they're ready
(no readiness gating) and/or old backends killed without **connection
draining**. Ensure the LB drains in-flight requests on removal and only
adds backends once they pass readiness (k8s: readiness probe gates
Service endpoints — see `diagnostics/k8s-probe-failures.md`).

## Decision tree

| Signal | Action |
|---|---|
| manual check OK, LB says down | fix LB check config (port/path/code) or source network |
| flapping under load | loosen check timeout / raise unhealthy threshold |
| whole fleet ejected on dep blip | use shallow check; don't gate LB on shared deps |
| 502 at deploy | enable connection draining + readiness gating |
| genuinely unhealthy backends | chase the backend error (5xx/latency playbooks) |

## References

- [HAProxy — health checks](https://www.haproxy.com/documentation/haproxy-configuration-tutorials/service-reliability/health-checks/)
- [Google SRE — Load balancing & graceful degradation](https://sre.google/sre-book/load-balancing-datacenter/)
- vault: `diagnostics/error-rate-5xx.md`, `diagnostics/high-latency-p99.md`, `diagnostics/k8s-probe-failures.md`
