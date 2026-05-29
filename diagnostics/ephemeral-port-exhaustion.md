---
title: Ephemeral Port Exhaustion (Outbound connect() Fails)
kind: howto
tags: [network, ports, ephemeral, time-wait, snat, connect, eaddrnotavail]
applies_to: [edge, manager]
---

# Ephemeral Port Exhaustion (Outbound connect() Fails)

Use when a client/proxy intermittently fails to open *outbound*
connections — `EADDRNOTAVAIL`, `cannot assign requested address`, or
connects that hang — while inbound and the network are fine. A host has
~28k ephemeral ports per (src-ip, dst-ip, dst-port) tuple; burn through
them (usually via piles of `TIME_WAIT`) and new connects fail.

| Symptom | Probable cause |
|---|---|
| `cannot assign requested address` on connect | local ephemeral pool exhausted |
| Fails only to ONE destination | per-tuple exhaustion (all conns to same ip:port) |
| Behind a NAT/LB, intermittent | SNAT source-port exhaustion on the gateway |
| Thousands of `TIME_WAIT` | short-lived connections not being reused |

## Step 1 — Count sockets by state

```bash
ss -s                                       # totals: estab / timewait / etc
ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
# Per-destination concentration (the per-tuple case)
ss -tan state time-wait | awk '{print $NF}' | sort | uniq -c | sort -rn | head
```

A huge `TIME_WAIT` count concentrated on one `dst:port` is per-tuple
exhaustion — you're hammering one upstream with new short connections.

## Step 2 — Check the pool size + headroom

```bash
sysctl net.ipv4.ip_local_port_range        # e.g. 32768 60999 ≈ 28k ports
# How close to the wall (rough): time_wait count vs pool size
```

If `TIME_WAIT` ≈ pool size for a tuple, you've hit it. Widening the range
buys headroom but doesn't fix churn.

## Step 3 — The real fix: stop churning connections

- **Connection reuse / keep-alive**: a pooled HTTP client that reuses
  connections eliminates the churn entirely — this is the right fix, not
  a sysctl. Verify the client's pool size > 0 and idle timeout is sane.
- **`tcp_tw_reuse`** (outbound only): lets the kernel reuse `TIME_WAIT`
  sockets for new *outbound* connections — safe for clients:
  ```bash
  sysctl -w net.ipv4.tcp_tw_reuse=1
  ```
  Do **not** reach for the old `tcp_tw_recycle` (removed in modern
  kernels; it broke NAT'd clients).

## Step 4 — Behind NAT/LB (SNAT port exhaustion)

If the host is fine but a shared NAT gateway is the bottleneck, the gateway
runs out of SNAT source ports across all clients. Spread load across more
source IPs, enable connection reuse end-to-end, or raise the gateway's
SNAT port range. In Kubernetes this is the classic node-SNAT limit.

## Decision tree

| Signal | Action |
|---|---|
| `TIME_WAIT` piled on one dst:port | enable client connection pooling/keep-alive |
| Broad ephemeral exhaustion | widen `ip_local_port_range` + `tcp_tw_reuse=1` (outbound) |
| Intermittent only behind NAT/LB | SNAT port exhaustion — scale source IPs / reuse conns |
| Client lib opens a conn per request | fix the client to reuse a pool — root cause |

## References

- [Coping with the TCP TIME-WAIT state — Vincent Bernat](https://vincent.bernat.ch/en/blog/2014-tcp-time-wait-state-linux)
- [ip-sysctl: tcp_tw_reuse — kernel.org](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- vault: `diagnostics/conntrack-table-full.md`, `diagnostics/fd-exhaustion.md`, `diagnostics/tcp-retransmit-loss.md`
