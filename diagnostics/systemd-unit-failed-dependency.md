---
title: systemd Unit Failed / Dependency & Ordering Issues
kind: howto
tags: [systemd, unit, dependency, ordering, boot, service, failed]
applies_to: [edge, manager]
---

# systemd Unit Failed / Dependency & Ordering Issues

Use when a service won't start at boot but starts fine manually, fails
with a dependency error, or a whole target is degraded. systemd starts
units per declared dependencies (`Requires`/`Wants`) and ordering
(`After`/`Before`). **Boot-only failures are usually ordering: the unit
started before something it needs (network, mount, another service) was
actually ready.**

| Symptom | Probable cause |
|---|---|
| Fails at boot, starts fine manually | ordering — needed dep not ready yet at boot |
| `dependency failed` / not started | a `Requires=` dep failed → this unit skipped |
| Starts before network/DB up | missing `After=`/`Wants=` ordering |
| `start request repeated too quickly` | crash-looping faster than `StartLimit` |

## Step 1 — Read the failure + dependency chain

```bash
systemctl status <unit> -l
journalctl -u <unit> -b --no-pager | tail -40       # this boot's logs
systemctl list-dependencies <unit>                  # what it requires/wants
systemd-analyze verify <unit>                        # config sanity
```

`status` shows the exit reason and which dependency (if any) failed.
`Active: failed (Result: dependency)` means a `Requires=` dep failed and
systemd refused to start this unit.

## Step 2 — Ordering: ready vs started

A frequent trap: `After=network.target` only means "network stack
started", **not** "network is up / has an IP". For services needing real
connectivity, order after `network-online.target` and pull it in:

```ini
[Unit]
After=network-online.target
Wants=network-online.target
# Needs a DB/mount? order after it AND require it:
After=postgresql.service
Requires=postgresql.service
```

`Requires=` (hard, fails this unit if dep fails) vs `Wants=` (soft) vs
`After=` (ordering only, no dependency) — pick deliberately.

## Step 3 — Crash-loop / start limits

```bash
systemctl show <unit> -p StartLimitBurst -p StartLimitIntervalSec -p Restart
# "start request repeated too quickly" = hit the rate limit; reset it:
systemctl reset-failed <unit>
```

If the unit crashes (not a dependency issue), it's a crashloop — see
`diagnostics/crashloop-restart.md`; fix the crash, don't just raise the
limit.

## Step 4 — Boot-time analysis

```bash
systemd-analyze blame | head                         # slowest units at boot
systemd-analyze critical-chain <unit>                # what gated this unit
```

`critical-chain` shows the ordering path that delayed/blocked the unit —
useful when a slow/failed upstream unit cascades.

## Decision tree

| Signal | Action |
|---|---|
| boot-only failure, needs network | `After/Wants=network-online.target` |
| `Result: dependency` | the `Requires=` dep failed — fix that dep |
| needs DB/mount not ready | add `After=`+`Requires=` for it |
| start request too quickly | `reset-failed`; fix the crash (crashloop playbook) |
| slow boot / cascade | `systemd-analyze critical-chain` to find the gate |

## References

- [systemd.unit(5) — dependencies & ordering](https://www.freedesktop.org/software/systemd/man/latest/systemd.unit.html)
- [network-online.target — systemd](https://www.freedesktop.org/software/systemd/man/latest/systemd.special.html#network-online.target)
- vault: `diagnostics/crashloop-restart.md`, `diagnostics/service-down-cn.md`, `diagnostics/journal-disk-full.md`
