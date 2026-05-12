---
title: Linux TCP Stack
tags: [network, tcp, sockets]
---

# Linux TCP Stack

The state machine, the queues, and the timers — knowing these names
turns "connection randomly failed" into a specific question.

## TCP state machine (operator subset)

```
                 CLOSED
                   │  active open
                   ▼
   SYN_SENT  ─────────►  ESTABLISHED  ─────────► FIN_WAIT_1
                                                     │
   passive open                                      ▼
   LISTEN  ──► SYN_RECV ──► ESTABLISHED ──► CLOSE_WAIT ──► LAST_ACK ──► CLOSED
                                                                          │
   active close ──► FIN_WAIT_1 ──► FIN_WAIT_2 ──► TIME_WAIT ──────────────┘
```

States that show up in `ss -ant` and what they mean operationally:

| State | What it means | Trouble shape |
|-------|---------------|---------------|
| LISTEN | Server bound, waiting for SYN | None unless absent |
| SYN_SENT | Client sent SYN, awaiting SYN-ACK | Many = remote down / firewall |
| SYN_RECV | Got SYN, sent SYN-ACK, awaiting ACK | Many = SYN flood, or queue full |
| ESTABLISHED | Connection open | Normal |
| CLOSE_WAIT | Remote closed, our app hasn't called close() | Many = app leaking fds |
| FIN_WAIT_2 | We closed, remote hasn't | Many = remote app holding |
| TIME_WAIT | We closed last, holding port for 2*MSL (60s default) | Many = short-lived connection storm; fix at client (keepalive / pooling) |

## Two queues per listen socket

```
client SYN ──► [ SYN queue ] ──► three-way handshake ──► [ accept queue ] ──► accept()
```

- **SYN queue** size: `min(backlog, net.core.somaxconn,
  net.ipv4.tcp_max_syn_backlog)`
- **Accept queue** size: same `backlog` argument to `listen(2)`

`ss -lnt` shows the current queue depths in `Recv-Q` (pending in
accept queue) and `Send-Q` (queue max). If Recv-Q sits at the max
under load, your application isn't accepting fast enough — either
the workers are blocked or there aren't enough of them. Bumping
`somaxconn` only delays the wall.

## Window and pacing

- **Receive window (rwnd)** — how much the receiver is willing to take;
  advertised in each ACK. Auto-tuned by the kernel via
  `tcp_rmem`.
- **Congestion window (cwnd)** — how much the sender thinks the
  network can hold. Controlled by the congestion algorithm.
- **`tcp_wmem`** / `tcp_rmem` — kernel buffer ranges (min default max).
  Long-fat-network bulk transfers benefit from larger max.

Congestion algorithm:

```bash
sysctl net.ipv4.tcp_available_congestion_control
sysctl net.ipv4.tcp_congestion_control
```

- **`cubic`** — default, fine for most LAN / WAN
- **`bbr`** — Google's; often dramatically better on lossy or
  bufferbloated paths, sometimes worse on clean LANs
- **`reno`** / others — niche

## Timers

- **SYN retransmits** — `net.ipv4.tcp_syn_retries` (default 6, ~1 min
  total). Half-open connection to unreachable host sits in SYN_SENT
  this long.
- **Retransmit timeout (RTO)** — adapts to RTT; min `tcp_min_rto`
- **Keepalive** — off by default at the kernel level. Enabled per
  socket via `SO_KEEPALIVE`. Tunables: `tcp_keepalive_time` (default
  7200s — way too long for most apps), `tcp_keepalive_intvl`,
  `tcp_keepalive_probes`.
- **FIN_WAIT_2 timeout** — `tcp_fin_timeout` (default 60s)
- **TIME_WAIT** — fixed at 2*MSL ≈ 60s; not directly tunable, despite
  what the internet tells you

## Operational knobs that actually help

```bash
# Allow port reuse from TIME_WAIT for outbound connections (client side only!)
sysctl -w net.ipv4.tcp_tw_reuse=1

# Raise SYN queue when SYN_RECV piles up under load
sysctl -w net.ipv4.tcp_max_syn_backlog=8192
sysctl -w net.core.somaxconn=8192

# Defense against SYN flood without dropping legit traffic
sysctl -w net.ipv4.tcp_syncookies=1   # default on most distros

# Larger buffers for long-fat-network bulk transfer
sysctl -w net.ipv4.tcp_rmem='4096 87380 33554432'
sysctl -w net.ipv4.tcp_wmem='4096 65536 33554432'
```

Avoid `tcp_tw_recycle` — it was removed for breaking NAT.

## Useful one-liners

```bash
ss -s                                  # socket summary by state
ss -ant | awk '{print $1}' | sort | uniq -c
ss -tnp 'state established'
ss -tnli                               # listen queues
nstat -az | grep -i drop               # listen-queue + retrans drops
ip -s -s link show <iface>             # detailed driver stats
```

## See also

- `diagnostics/network-connectivity.md`
- `systems/network/conntrack.md`
- `systems/network/dns.md`
