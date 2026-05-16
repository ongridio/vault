---
title: Conntrack Table Full — Short-Connection Storms and Drops
kind: howto
tags: [conntrack, netfilter, nat, packet-drop, short-conn]
applies_to: [edge, manager]
---

# Conntrack Table Full

Use when `dmesg` shows `nf_conntrack: table full, dropping packet`, OR
when TCP connections randomly fail / stall under load, OR when
`nf_conntrack` slab is growing without bound (see
`diagnostics/memory-leak-hunt.md` Step 5).

The kernel tracks every connection in a hash table. Once full it drops
**new** connections (existing ones keep working). Three failure modes:
table too small, entries not aging out, or short-connection storm.

## Step 1 — Confirm the symptom

```bash
# Are we actually full?
cat /proc/sys/net/netfilter/nf_conntrack_count       # current entries
cat /proc/sys/net/netfilter/nf_conntrack_max         # cap
cat /proc/net/stat/nf_conntrack | head -2            # cumulative stats
dmesg -T | grep -i conntrack | tail -20              # recent drops
```

`nf_conntrack_count > 0.8 × nf_conntrack_max` is the danger zone. Drops
begin around 100%, but performance degrades earlier from hash collisions.

Inspect the rate of churn (snapshot twice, 10s apart):

```bash
cat /proc/net/stat/nf_conntrack
```

Columns: `entries searched found new invalid ignore delete delete_list
insert insert_failed drop early_drop icmp_error expect_new expect_create
expect_delete search_restart`. `insert_failed` rising = the symptom in
our face.

## Step 2 — What's filling the table

```bash
conntrack -L 2>/dev/null | wc -l
conntrack -L -p tcp 2>/dev/null | awk '{print $4}' | sort | uniq -c | sort -rn
# -- counts by state: ESTABLISHED, TIME_WAIT, SYN_SENT, ...

conntrack -L -p tcp 2>/dev/null \
  | awk '{ for (i=1;i<=NF;i++) if ($i ~ /dport=/) print $i }' \
  | sort | uniq -c | sort -rn | head -20
# -- top destination ports
```

Three diagnostic patterns:

| Pattern | Diagnosis |
|---|---|
| 80%+ entries in `TIME_WAIT` | Short-connection storm; entries waiting 2*MSL (default 120s) to be reaped |
| 80%+ in `ESTABLISHED` to one dport | Real connection count grew beyond table size; bump cap |
| Many `SYN_SENT` to varied IPs | Outbound port scan / health-check spray; could be benign |
| Many `INVALID` state | Asymmetric routing — packets going one direction only |

## Step 3 — Tunables (use carefully, default values are conservative)

```bash
# Raise the cap. Each entry is ~300 bytes, so 1M entries = ~300 MB RAM.
sysctl -w net.netfilter.nf_conntrack_max=1048576
# Match hash table size for O(1) lookup (default = max/8)
echo 131072 > /sys/module/nf_conntrack/parameters/hashsize

# Drop TIME_WAIT timeout — only safe in low-RTT controlled networks.
# Default 120s; halving to 60s OK in most LAN; below 30s risks
# delayed-ACK confusing the next handshake.
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_time_wait=60

# Aggressive ESTABLISHED timeout makes idle long-lived connections die.
# Default 432000s (5 days). Don't lower below your typical TCP keepalive
# interval or you'll start dropping live connections.
sysctl -w net.netfilter.nf_conntrack_tcp_timeout_established=86400
```

Persist via `/etc/sysctl.d/*.conf` after testing.

## Step 4 — Eliminate the source of churn

If the table is full because of a short-connection storm (Step 2 showed
TIME_WAIT dominance), fixing the application is better than tuning:

- **Connection pooling** — every HTTP client / DB client should reuse
  connections. Per-request TCP setup is the leading cause.
- **Keep-Alive on HTTP** — make sure both client and server agree
  (`Connection: keep-alive`, no premature close).
- **Persistent agents** — replace `curl` in cron with a long-lived
  daemon if hit frequency > 1/s.

## Step 5 — Bypass conntrack for traffic that doesn't need it

```bash
# Mark specific flows as NOTRACK — they skip conntrack entirely.
# Example: bypass conntrack for traffic to internal service mesh.
iptables -t raw -A PREROUTING -d 10.10.0.0/16 -j CT --notrack
iptables -t raw -A OUTPUT -s 10.10.0.0/16 -j CT --notrack
```

Use this for:
- LAN-internal traffic that doesn't need NAT
- High-volume health checks
- DNS to internal resolvers

Don't use for: anything that hits NAT, anything you want to filter
stateful.

## Step 6 — Verify after change

```bash
# Watch the count for 5 minutes after tuning
watch -n 5 'echo count: $(cat /proc/sys/net/netfilter/nf_conntrack_count); \
            echo failed: $(grep -o "insert_failed [0-9]*" /proc/net/stat/nf_conntrack | head -1)'
```

If count plateaus below `max × 0.7` and `insert_failed` stops growing,
you've reached steady state. If count keeps climbing, raise max
further OR fix the underlying churn.

## Decision tree

| Signal | Action |
|---|---|
| `insert_failed` growing, count near max, TIME_WAIT dominant | Lower `time_wait` timeout OR fix app pooling — Step 3/4 |
| Count near max, all ESTABLISHED, all legit | Raise `nf_conntrack_max` — Step 3 |
| Count low but many INVALID | Asymmetric routing — escalate to network, not a conntrack problem |
| LAN-only traffic dominates table | Apply NOTRACK rules — Step 5 |
| `SUnreclaim` (slab) growing with `nf_conntrack` | Kernel leak — check kernel version, file bug |

## References

- [Netfilter conntrack tuning — RHEL docs](https://access.redhat.com/solutions/8721)
- [`conntrack-tools` user guide](https://conntrack-tools.netfilter.org/manual.html)
- [TIME_WAIT and the sins of premature optimisation — Vincent Bernat](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
- vault: `systems/network/conntrack.md`, `diagnostics/tcp-retransmit-loss.md`
