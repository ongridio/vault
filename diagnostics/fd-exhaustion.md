---
title: File Descriptor Exhaustion ("Too Many Open Files")
kind: howto
tags: [fd, file-descriptors, ulimit, sockets, epoll, limits]
applies_to: [edge, manager]
---

# File Descriptor Exhaustion ("Too Many Open Files")

Use when logs show `EMFILE` / "too many open files", `accept()` starts
failing, or a service stops taking new connections while existing ones
work. **Find which limit you hit**: the per-process soft limit
(`RLIMIT_NOFILE`), the system-wide `file-max`, or — for sockets — the
conntrack/port range, which masquerades as an fd problem.

| Symptom | Probable cause class |
|---|---|
| `accept: too many open files`, others fine | per-process soft limit reached |
| Many processes failing at once | system-wide `fs.file-max` reached |
| FD count climbs forever | descriptor leak (unclosed conns/files) |
| Plateaus near a round number (1024/65536) | hit a configured ulimit ceiling |

## Step 1 — Count open FDs vs the limit

```bash
pid=$(pgrep -n <svc>)
ls /proc/$pid/fd | wc -l                         # current open FDs
cat /proc/$pid/limits | grep 'open files'        # soft / hard RLIMIT_NOFILE
# System-wide
cat /proc/sys/fs/file-nr                          # allocated  unused  max
sysctl fs.file-max
```

If `ls /proc/$pid/fd | wc -l` is at (or just under) the soft limit,
that's the wall. If `file-nr`'s first column nears `file-max`, it's
system-wide.

## Step 2 — What are the FDs (leak triage)

```bash
# Break down by type — sockets vs regular files vs pipes
ls -l /proc/$pid/fd | awk '{print $11}' | sed -E 's/[0-9]+$//' | sort | uniq -c | sort -rn
# Sockets dominating? Break down by TCP state
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
```

A pile of `CLOSE_WAIT` sockets = **the app isn't calling close()** on
peer-closed connections (classic leak). A pile of `TIME_WAIT` is usually
benign churn, not a leak. Many regular-file FDs to the same path = a
file handle leak.

## Step 3 — Watch it move

```bash
# Is it leaking (monotonic) or just spiky under load?
while :; do echo "$(date +%T) $(ls /proc/$pid/fd | wc -l)"; sleep 5; done
```

Monotonic rise that never drops, even when traffic dips = leak → fix the
code path that opens without closing. Rises with load and recedes = you
just need a higher limit.

## Step 3.5 — Sockets: is it really port/conntrack, not FDs?

```bash
sysctl net.ipv4.ip_local_port_range              # ephemeral port pool size
ss -s                                            # total sockets by state
conntrack -C; sysctl net.netfilter.nf_conntrack_max
```

Outbound-heavy clients can exhaust ephemeral ports or conntrack while
FDs look fine — see `diagnostics/conntrack-table-full.md`.

## Step 4 — Raise the limit correctly (if not a leak)

```bash
# systemd service — the ulimit that actually applies to daemons
#   add to the unit (NOT /etc/security/limits.conf, which PAM-only):
#   [Service]
#   LimitNOFILE=1048576
systemctl show <svc> -p LimitNOFILE
# Container: docker run --ulimit nofile=1048576:1048576 ...
# System ceiling if file-max is the wall:
sysctl -w fs.file-max=2097152
```

For systemd-managed daemons, `limits.conf` is ignored — the unit's
`LimitNOFILE` is what counts. This is the most common "I raised the
ulimit and it didn't help" trap.

## Decision tree

| Signal | Action |
|---|---|
| At per-process soft limit, not leaking | raise `LimitNOFILE` in the systemd unit / container |
| `file-nr` near `file-max` | raise `fs.file-max` (system-wide) |
| FD count monotonic, many `CLOSE_WAIT` | descriptor leak — fix missing close()/defer |
| Sockets fine but connects fail | ephemeral ports / conntrack — `diagnostics/conntrack-table-full.md` |
| Raised ulimit, no effect, systemd daemon | set it in the unit, not `limits.conf` |

## References

- [setrlimit(2) — RLIMIT_NOFILE](https://man7.org/linux/man-pages/man2/getrlimit.2.html)
- [systemd LimitNOFILE — systemd.exec(5)](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#LimitCPU=)
- [CLOSE_WAIT vs TIME_WAIT explained](https://blog.cloudflare.com/this-is-strictly-a-violation-of-the-tcp-specification/)
- vault: `diagnostics/conntrack-table-full.md`, `diagnostics/error-rate-5xx.md`, `diagnostics/tcp-retransmit-loss.md`, `reference/linux-commands.md`
