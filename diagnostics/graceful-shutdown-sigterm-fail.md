---
title: Graceful Shutdown Failures (SIGTERM Ignored / 502s on Deploy)
kind: howto
tags: [process, sigterm, shutdown, draining, deploy, kubernetes, systemd]
applies_to: [edge, manager]
---

# Graceful Shutdown Failures (SIGTERM Ignored / 502s on Deploy)

Use when restarts/deploys cause errors: in-flight requests get dropped,
clients see 502/connection-reset during a rollout, or a process takes the
full kill-timeout to die (SIGKILL). The orchestrator sends `SIGTERM` and
expects the app to stop accepting new work, finish in-flight, and exit —
**if the app ignores SIGTERM or doesn't drain, the platform SIGKILLs it
and you drop requests.**

| Symptom | Probable cause |
|---|---|
| 502/reset during deploy | old pod killed before draining in-flight requests |
| Process takes ~grace-period then dies | SIGTERM not handled → SIGKILL after timeout |
| Lost messages/jobs on restart | no graceful drain of queues/connections |
| Hangs on shutdown | shutdown handler blocks (waits forever on something) |

## Step 1 — Does the app handle SIGTERM at all?

```bash
# Send SIGTERM and watch: does it exit promptly + cleanly, or hang to SIGKILL?
time kill -TERM <pid>      # if it lingers ~grace period, SIGTERM isn't handled
# What signals does it even catch? (SigCgt bitmask)
grep -E 'SigCgt|SigIgn' /proc/<pid>/status
```

If `SIGTERM` isn't in the caught set, the default action (terminate) is
abrupt — no draining. Many apps under a shell wrapper never receive
SIGTERM (the shell does) — check PID 1 / exec form.

## Step 2 — The PID 1 / wrapper trap (containers)

```bash
# In a container, signals go to PID 1. A shell-form CMD makes the shell PID 1,
# which may NOT forward SIGTERM to the app:
#   BAD:  CMD app args            (sh -c wrapper swallows signals)
#   GOOD: exec form / exec app    (app is PID 1 and gets SIGTERM)
```

A very common cause of "SIGTERM ignored" in containers is the app running
under `sh -c` so it never gets the signal — use exec form or an init.

## Step 3 — Draining order (the 502 source)

Correct graceful shutdown sequence:
1. On SIGTERM, **stop accepting new** connections/requests (close the
   listener / fail readiness so the LB stops routing).
2. **Finish in-flight** requests (with a deadline).
3. Close downstream resources, then exit 0.

In Kubernetes, also respect `terminationGracePeriodSeconds` and use a
`preStop` hook / readiness flip + a short sleep so the LB/endpoints
deregister *before* the process stops — otherwise traffic still arrives
mid-shutdown (502s).

## Step 4 — Tune the grace period to reality

```bash
# systemd: time before SIGKILL
systemctl show <svc> -p TimeoutStopSec
# k8s: terminationGracePeriodSeconds on the pod
```

If real in-flight work needs 30s to drain but the grace period is 10s,
the platform SIGKILLs mid-request. Set the grace period ≥ the longest
in-flight request, and make the handler actually finish within it.

## Decision tree

| Signal | Action |
|---|---|
| lingers to SIGKILL | app doesn't handle SIGTERM — add a handler |
| container, signal never arrives | run app as PID 1 (exec form) / add init |
| 502s at deploy | drain order: fail readiness first, then finish in-flight |
| grace period < drain time | raise TimeoutStopSec / terminationGracePeriodSeconds |
| shutdown hangs forever | shutdown handler blocks — add a deadline + force exit |

## References

- [Kubernetes — Pod termination lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)
- [systemd.kill — TimeoutStopSec](https://www.freedesktop.org/software/systemd/man/latest/systemd.kill.html)
- vault: `diagnostics/crashloop-restart.md`, `diagnostics/error-rate-5xx.md`, `diagnostics/k8s-probe-failures.md`
