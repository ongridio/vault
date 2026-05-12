---
title: Host Metric Alerts Runbook
tags: [host, cpu, memory, disk, load]
alert_keys: [cpu_high, cpu_high_default, mem_high, swap_high, disk_high, disk_full_warning, load1_high, fd_exhaustion]
applies_to: [edge, manager]
---

# Host Metric Alerts Runbook

Per-alert handler for the host-metric rules seeded by Ongrid. Each section
matches a `rule_key` so the rule's `runbook_url` can deep-link with a
fragment, e.g. `#cpu_high`.

## How to use this page

1. Read **What this means** to confirm the signal interpretation
2. Run **Immediate checks** — these are safe, read-only
3. If the cause is obvious, follow the **Likely causes** branch
4. Otherwise escalate using the **Escalation criteria** at the bottom

---

## cpu_high  / cpu_high_default <a id="cpu_high"></a>

**What this means.** CPU utilization on the host exceeded the configured
threshold for the configured window. The default rule fires at >90% for
5 minutes. `cpu_high_default` is the platform default; `cpu_high` is the
per-edge override key.

**Immediate checks.**

```bash
top -bn1 | head -20
mpstat -P ALL 1 3            # per-core utilization
pidstat -u 1 3 | head -20    # per-process
```

**Likely causes.**

- Runaway process (compile job, ML training, broken loop) → `pidstat` will
  show the culprit
- Kernel/IO wait pegged → `mpstat` shows `%iowait` high → jump to disk
  pressure
- Steal time on a VM → `%steal` non-zero → talk to the hypervisor owner

---

## mem_high <a id="mem_high"></a>

**What this means.** Resident memory usage crossed the alert threshold.
Note: `MemAvailable` is the right number for "can the system allocate?",
not `MemFree`.

**Immediate checks.**

```bash
free -h
cat /proc/meminfo | head -20
ps aux --sort=-rss | head -10
```

**Likely causes.**

- Leak in a long-running process → RSS climbs monotonically → restart +
  capture heap profile if available
- Page cache pressure (normal under heavy IO) → check `Cached` vs.
  `MemAvailable`; this is usually fine
- OOM imminent → check `dmesg -T | tail` and `/var/log/messages` for
  prior OOM kills

---

## swap_high <a id="swap_high"></a>

**What this means.** Swap utilization is high — the system is moving
pages off RAM. Often a leading indicator of memory pressure.

**Immediate checks.**

```bash
swapon --show
vmstat 1 5                   # si/so columns = swap-in/out activity
```

If `si`/`so` are sustained non-zero, you're actively thrashing.
Persistent swap > 50% is almost always worth a restart of the heaviest
RSS consumer.

---

## disk_high  / disk_full_warning <a id="disk_high"></a>

See the dedicated [Disk Pressure diagnostic](../diagnostics/disk-pressure.md)
for the full walkthrough. Short version:

```bash
df -h
df -i
du -xh / 2>/dev/null | sort -rh | head -20
```

Default thresholds: warning at 80%, critical at 90%.

---

## load1_high <a id="load1_high"></a>

**What this means.** 1-minute load average exceeds the threshold. Load
counts running + runnable + uninterruptible-sleep tasks, so a load spike
without CPU spike usually means IO wait.

**Immediate checks.**

```bash
uptime
ps -eo stat,pid,comm | awk '$1 ~ /^D/ {print}'   # D-state (IO wait)
iostat -xz 1 3
```

A load value of N with a Y-core machine is "saturated" once N > Y.

---

## fd_exhaustion <a id="fd_exhaustion"></a>

**What this means.** Process or system-wide file descriptor count is
approaching the limit.

**Immediate checks.**

```bash
cat /proc/sys/fs/file-nr             # used, allocated, max
lsof | awk '{print $2}' | sort | uniq -c | sort -rn | head -10
ulimit -n
```

**Common culprits.**

- Connection leak in an HTTP client (missing `defer conn.Close()`)
- A process holding deleted files (`lsof | grep deleted`) — restart
  releases them
- ulimit too low for the workload — raise `LimitNOFILE` in the unit file
  and `daemon-reload`

---

## Escalation criteria

Escalate to on-call (page severity P2) if **any**:

- Resource still climbing after 10 minutes of remediation
- Customer-visible impact reported in `#incidents`
- More than one host on the same edge cluster showing the same alert
  inside 5 minutes (cluster-wide cause, not a single bad actor)
- The alert is `disk_full_warning` and free space is < 5%
