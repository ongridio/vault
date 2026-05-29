---
title: Kubernetes Readiness / Liveness Probe Failures
kind: howto
tags: [kubernetes, probe, readiness, liveness, healthcheck, restart, endpoints]
applies_to: [edge, manager]
---

# Kubernetes Readiness / Liveness Probe Failures

Use when a pod is `Running` but not `Ready` (0/1), traffic isn't routed
to it, or it's being restarted on a cycle. **Readiness controls traffic
(removed from Service endpoints when failing); liveness controls
restarts (kubelet kills the container).** Misconfigured probes cause two
classic self-inflicted outages: no pod ever becomes Ready (outage with
healthy apps), or a slow-but-healthy app gets liveness-killed in a loop.

| Symptom | Probable cause |
|---|---|
| Running, 0/1 Ready, no traffic | readiness probe failing |
| Restart loop, exit by SIGTERM after Ns | liveness probe failing → kubelet kills |
| Ready flaps under load | probe timeout too tight for loaded app |
| Never Ready at startup | no `startupProbe`; readiness fails during slow init |

## Step 1 — Read the probe result + config

```bash
kubectl describe pod <pod> | grep -A2 -iE 'Readiness|Liveness|Startup'
kubectl describe pod <pod> | grep -iE 'Unhealthy|probe failed'   # Events
kubectl get pod <pod> -o jsonpath='{range .spec.containers[*]}{.livenessProbe}{"\n"}{.readinessProbe}{"\n"}{end}'
```

The `Unhealthy` event shows the probe type + the failure (HTTP code,
timeout, connection refused). Note the probe's `path`, `port`,
`timeoutSeconds`, `periodSeconds`, `failureThreshold`.

## Step 2 — Probe the endpoint exactly as the kubelet does

```bash
kubectl exec <pod> -- curl -fsS -o /dev/null -w '%{http_code} %{time_total}s\n' \
  http://localhost:<port><path>
```

- `connection refused` → app not listening on that port yet (init too
  slow → add a `startupProbe`) or wrong port.
- 503/500 → the app's own health endpoint says unhealthy (a real
  dependency problem — chase that).
- slow (`time_total` > `timeoutSeconds`) → probe timing too aggressive
  for a loaded app.

## Step 3 — Liveness-kill vs readiness-gate

```bash
kubectl describe pod <pod> | grep -A6 'Last State'   # Reason: Error, exit by signal?
```

If the container is being **killed** (restart count climbing, killed by
the platform), it's the *liveness* probe. A liveness probe pointing at a
slow/heavy endpoint, or with `timeoutSeconds`/`failureThreshold` too
tight, turns transient slowness into a restart storm — the app was fine.

## Step 4 — Fix the probe, not the app

- **Slow startup** → add a `startupProbe` (generous failureThreshold) so
  liveness/readiness don't fire until init completes.
- **Liveness too aggressive** → raise `timeoutSeconds`/`failureThreshold`,
  point it at a *cheap* liveness endpoint (not one that hits the DB).
- **Readiness should gate on dependencies** (DB reachable), liveness
  should NOT (else a brief dependency blip restarts a healthy app).

## Decision tree

| Signal | Action |
|---|---|
| 0/1 Ready, connection refused | wrong port / slow init — add startupProbe |
| Readiness 503 from app | real dependency problem — chase the dependency |
| Liveness restart loop, app healthy | loosen liveness timeout/threshold; cheap endpoint |
| Flaps under load | probe timeout too tight — raise it / separate liveness from work |
| Slow init always not-Ready | add startupProbe with generous budget |

## References

- [Configure Liveness, Readiness and Startup Probes — Kubernetes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- vault: `diagnostics/crashloop-restart.md`, `diagnostics/high-latency-p99.md`, `systems/container/kubernetes.md`
