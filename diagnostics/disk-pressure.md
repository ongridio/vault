---
title: Disk Pressure Diagnosis
tags: [disk, filesystem, io, inodes]
applies_to: [edge, manager]
---

# Disk Pressure Diagnosis

Use when disk usage is high, inodes are exhausted, or IO is slow. The
fix-tree branches early on **blocks vs. inodes vs. IO** — name which one
you're hitting before you start clearing anything.

## Step 1 — Establish what's exhausted

```bash
df -h
df -i
```

Blocks and inodes can fail independently. `df -h` shows 100% but `df -i`
shows 5% → you've got too many small files, not a space problem. Inverse
is also common (one giant log, plenty of inodes).

## Step 2 — Find the big consumers

```bash
# Largest subdirs (skips other mounts with -x)
du -xh / 2>/dev/null | sort -rh | head -20
# Top files by size
find / -xdev -type f -size +100M 2>/dev/null \
  | xargs du -sh 2>/dev/null \
  | sort -rh | head -20
```

If you suspect inodes, count files instead:

```bash
find / -xdev -type f 2>/dev/null | awk -F/ '{print $2"/"$3}' | sort | uniq -c | sort -rn | head -20
```

## Step 3 — Container fileystems (Docker / k8s)

```bash
docker system df
docker image prune -af --filter "until=720h"   # images older than 30 days
docker volume ls -qf dangling=true | xargs -r docker volume rm
```

For k8s nodes, also check `/var/lib/kubelet/pods/` — orphaned pod volumes
can survive a kubelet restart.

## Step 4 — Logs / journald

```bash
journalctl --disk-usage
journalctl --vacuum-time=7d
du -xh /var/log | sort -rh | head -20
```

If `journalctl --disk-usage` is multi-gigabyte, your unit files are
probably missing `RateLimitBurst` — fix at the source rather than just
vacuuming.

## Step 5 — TSDB and database storage

```bash
# Prometheus
du -sh /var/lib/prometheus/data/
# MySQL (per-schema rollup)
mysql -e "SELECT table_schema, ROUND(SUM(data_length+index_length)/1024/1024,1) AS mb \
  FROM information_schema.tables GROUP BY table_schema ORDER BY mb DESC LIMIT 10"
```

Prometheus growth is almost always cardinality. If retention is fine but
storage keeps climbing, find the high-churn labels with `count by (__name__) ({__name__=~".+"})`.

## Step 6 — IO performance (when usage is fine but apps are slow)

```bash
iostat -xz 1 5
iotop -oPa
```

- `await` high + `svctm` high → hardware or driver
- `await` high + `svctm` low → too many concurrent requests, queue
- `%util` near 100% on a single device → that device is the bottleneck

## Decision tree

- **Blocks exhausted** → identify top consumers → purge logs / images, or
  expand the volume
- **Inodes exhausted** → look for many-small-files producers (often
  `/var/lib/docker/overlay2`, mail spools, npm caches)
- **Slow IO** → use `iostat await + svctm` to split hardware vs.
  workload, then act on the right side of that split
