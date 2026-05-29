---
title: Log Flood / journald & Log Disk Full
kind: howto
tags: [logs, journald, disk, log-rotation, flood, observability]
applies_to: [edge, manager]
---

# Log Flood / journald & Log Disk Full

Use when a disk fills up with logs, journald grows unbounded, or a log
flood is both filling storage and drowning useful signal. A crash-looping
or misconfigured service can spew millions of lines, filling the FS
(→ read-only remounts, write failures) and saturating IO. **Find the
source of the flood first — capping rotation without stopping the spammer
just delays the next fill.**

| Symptom | Probable cause |
|---|---|
| Disk full, huge `/var/log` or journal | log flood + no/large rotation cap |
| One service dominates the journal | that service spamming (crashloop, debug log left on) |
| journald using GBs | `SystemMaxUse` unset/large; persistent storage unbounded |
| IO saturated by logging | sync logging of a flood |

## Step 1 — Where is the space + who's spamming

```bash
du -xhd1 /var/log | sort -h | tail              # biggest log dirs/files
journalctl --disk-usage                          # how much journald holds
# Top log producers in the journal (recent)
journalctl --since '10 min ago' -o json --no-pager 2>/dev/null \
  | grep -o '"_SYSTEMD_UNIT":"[^"]*"' | sort | uniq -c | sort -rn | head
```

The unit producing the most lines is the flood source — usually a
crash-looping service (see `diagnostics/crashloop-restart.md`) or one left
on debug/trace level.

## Step 2 — Reclaim space now (stop the bleeding)

```bash
# Vacuum the journal to a size / age
journalctl --vacuum-size=500M
journalctl --vacuum-time=2d
# Truncate a runaway plain log file WITHOUT breaking the writer's fd:
: > /var/log/<huge.log>        # truncate in place (do NOT rm an open file)
```

Don't `rm` a log file an active process holds open — the space isn't
freed until the process closes the fd (it keeps writing to the unlinked
inode). Truncate (`: >`) instead, or restart the writer.

## Step 3 — Stop the source

Fix the actual spammer: resolve the crashloop, turn off debug logging
left enabled, or rate-limit a chatty code path. A flood also blows past
downstream limits (Loki 429s — `diagnostics/loki-ingestion-backpressure.md`).

## Step 4 — Cap it going forward

```ini
# /etc/systemd/journald.conf
[Journal]
SystemMaxUse=1G
SystemMaxFileSize=128M
MaxRetentionSec=1week
RateLimitIntervalSec=30s
RateLimitBurst=10000
```

And ensure `logrotate` covers plain log files (size + count caps,
compression). journald `RateLimit*` drops bursts from a single unit to
protect the system.

## Decision tree

| Signal | Action |
|---|---|
| disk full from logs | vacuum journal / truncate (don't rm open files); free space |
| one unit floods | fix the spammer (crashloop / debug-level left on) |
| journald unbounded | set `SystemMaxUse`/retention + RateLimit in journald.conf |
| plain logs unbounded | configure logrotate (size/age/compress) |
| flood also 429s Loki | cap source — `diagnostics/loki-ingestion-backpressure.md` |

## References

- [journald.conf(5) — freedesktop](https://www.freedesktop.org/software/systemd/man/latest/journald.conf.html)
- [logrotate(8) — man7](https://man7.org/linux/man-pages/man8/logrotate.8.html)
- vault: `diagnostics/disk-full-cn.md`, `diagnostics/crashloop-restart.md`, `diagnostics/loki-ingestion-backpressure.md`
