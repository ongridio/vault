---
title: Service CrashLoop / Restart Storm
kind: howto
tags: [crashloop, restart, systemd, kubernetes, exit-code, flapping]
applies_to: [edge, manager]
---

# Service CrashLoop / Restart Storm

Use when a service keeps dying and restarting — systemd `auto-restart`
churning, a container `Restarting`, or k8s `CrashLoopBackOff`. **The exit
code + the last few log lines before each death answer 80% of it.** A
restart *storm* also masks the real error because logs scroll fast and
the back-off delay grows — slow it down to read it.

| Exit / reason | Probable cause |
|---|---|
| 1 / 2 + stack at startup | config/env error, missing dependency, bad migration |
| 137 (128+9, SIGKILL) | OOMKilled (or liveness probe kill) — see oom playbook |
| 139 (128+11, SIGSEGV) | native crash / bad binary / lib mismatch |
| 0 then restart | clean exit but `Restart=always` — supervises a one-shot |
| healthy then dies after Ns | liveness/readiness probe failing |

## Step 1 — Get the exit code + restart cadence

```bash
# systemd
systemctl status <svc> | grep -E 'Active|Main PID|status='
journalctl -u <svc> -n 80 --no-pager     # logs around the deaths
# container / k8s
docker inspect <ctr> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
kubectl describe pod <pod> | grep -A6 'Last State'   # reason + exit code + message
kubectl logs <pod> --previous            # logs from the CRASHED instance, not the new one
```

`--previous` / `kubectl logs -p` is the key move: the *current* container
is fresh and quiet; the error lives in the *previous* one's logs.

## Step 2 — Slow the loop so you can read it

```bash
# systemd churning too fast — stop it auto-restarting while you debug
systemctl stop <svc>
# then run the binary in the foreground with the same env to see the error live
# k8s: kubectl get pod -w  to watch the back-off grow (10s,20s,40s...)
```

A storm with sub-second restarts thrashes CPU/IO and floods logs —
pausing the supervisor turns a blur into a single readable failure.

## Step 3 — Classify the failure

- **Dies immediately at startup** → config/env/dependency: missing file,
  bad DSN, unreachable dependency, failed DB migration. Check the very
  first error line.
- **137 OOMKilled** → memory limit too low / leak → `diagnostics/oom-killed.md`.
- **Runs then killed after N seconds** → liveness probe failing (endpoint
  slow/wrong) — the platform is killing a *healthy* app; fix the probe.
- **SIGSEGV/139** → binary/lib/arch mismatch or a real crash bug; get a
  core dump or stack.

## Step 4 — Dependency-ordering crashloops

A service that crashes because a dependency (DB, broker, config service)
isn't up yet will loop until the dependency arrives. Add proper
readiness gating / retry-with-backoff on the dependency rather than
relying on the restart loop to eventually win — the loop wastes
resources and pollutes alerting.

## Decision tree

| Signal | Action |
|---|---|
| Exit 1/2 + startup stack | fix config/env/dependency; read first error line |
| Exit 137 / OOMKilled | raise memory limit / fix leak — `diagnostics/oom-killed.md` |
| Healthy then killed after Ns | liveness probe wrong/slow — fix the probe, not the app |
| SIGSEGV (139) | binary/lib/arch mismatch or crash bug — core dump |
| Crashes until dependency up | add readiness gating + backoff on the dependency |

## References

- [systemd.service Restart= — freedesktop](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html#Restart=)
- [Kubernetes — Debug Running Pods (CrashLoopBackOff)](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/)
- vault: `diagnostics/oom-killed.md`, `diagnostics/service-down-cn.md`, `systems/container/kubernetes.md`
