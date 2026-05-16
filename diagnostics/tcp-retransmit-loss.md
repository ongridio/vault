---
title: TCP Retransmits, Loss, and Resets
kind: howto
tags: [tcp, network, retransmit, packet-loss, rst]
applies_to: [edge, manager]
---

# TCP Retransmits, Loss, and Resets

Use when an application complains "connection is slow / drops / resets".
The three symptoms have different causes and different evidence trails;
**name which one you're seeing first** before reaching for tcpdump.

| Symptom | Probable cause class |
|---|---|
| Slow but completes | retransmits (path loss / queue) |
| Stalls then resumes | RTO timeouts |
| `Connection reset by peer` | RST sent by other side (or middlebox) |
| Half-open / `EPIPE` after idle | NAT / firewall timeout |

## Step 1 — Snapshot of system-wide TCP health

```bash
ss -s                                 # estab / closed / orphaned
nstat -z | grep -E 'Retrans|TCPLost|RcvCollapsed|TCPAbortFailed'
cat /proc/net/snmp | awk '/^Tcp:/ {for (i=1;i<=NF;i++) print i, $i}'
```

Key counters:
- `TcpRetransSegs` — segments retransmitted
- `TcpOutRsts` — RSTs we sent
- `TcpExtListenOverflows` — accept queue full (= overload)
- `TcpExtTCPSynRetrans` — SYN retransmits (initial handshake failing)

Take a sample, wait 30s, take another, compute deltas. A rising
`TcpRetransSegs / TcpOutSegs` ratio > 1% is significant.

## Step 2 — Per-connection visibility

```bash
ss -tin                               # all TCP, internal info
ss -tin state established '( dport = :443 or sport = :443 )'
```

The line below each connection has fat fields like `rto:200 rtt:12/4 mss:1448 cwnd:10 ssthresh:7 bytes_sent:... unacked:... retrans:0/3 lost:0`.

What to look for:
- `retrans:X/Y` — X recent, Y total. Y growing fast = bad path
- `unacked` consistently > 0 = peer not ACKing
- `cwnd:1` after `ssthresh` collapse = recovering from loss
- `rcv_space` shrinking = receiver backed up

## Step 3 — Capture only what you need

```bash
# Capture for ONE target only — full capture floods the disk
sudo tcpdump -nn -i any -s 96 'host 10.0.0.5 and tcp' -w /tmp/tcp.pcap
# Stop after a representative chunk (Ctrl-C). Then offline analysis:
tshark -r /tmp/tcp.pcap -q -z io,stat,1
tshark -r /tmp/tcp.pcap -Y 'tcp.analysis.retransmission'
tshark -r /tmp/tcp.pcap -Y 'tcp.flags.reset == 1'
```

Wireshark display filters:
- `tcp.analysis.retransmission` — actual retransmits
- `tcp.analysis.fast_retransmission` — 3-dupack triggered
- `tcp.analysis.lost_segment` — receiver saw gap
- `tcp.analysis.zero_window` — receiver buffer full

If retransmits cluster around a specific peer or hop, that's your
network problem to escalate.

## Step 4 — Identify who sent the RST

```bash
# In real time, watch RSTs only
sudo tcpdump -nn -i any 'tcp[tcpflags] & tcp-rst != 0' -c 50

# Compare with conntrack — if conntrack entry is in TIME_WAIT or
# CLOSE_WAIT but the stack sent a RST, NAT may be timing out
conntrack -L -p tcp --state ESTABLISHED | head -20
```

Common RST patterns and culprits:

| Pattern | Likely culprit |
|---|---|
| RST after long idle period | NAT/firewall conntrack timeout (default ~5 min on many devices) |
| RST immediately after `Connection: close` | normal, ignore |
| RST from middle-box (TTL ≠ peer's) | IDS / inline appliance |
| RST after SYN/SYN+ACK | listen socket missing or `tcp_abort_on_overflow` |
| RST with `tcp_rst on closed` log | application closed before draining receive buffer |

## Step 5 — Path-level loss confirmation

```bash
mtr -rwn -c 100 <dest>                # 100 packets via each hop
ping -i 0.2 -c 200 <dest>             # 200 pings @ 5/sec
```

`mtr` showing hop-N loss while N+1 is clean = ICMP rate-limit, not real
loss. Real loss is visible on the **last** hop and at all subsequent.

## Tunables worth knowing

```bash
# Conservative defaults that hurt high-RTT links
sysctl net.ipv4.tcp_sack                # should be 1
sysctl net.ipv4.tcp_window_scaling      # should be 1
sysctl net.ipv4.tcp_timestamps          # should be 1
sysctl net.core.rmem_max net.core.wmem_max
sysctl net.ipv4.tcp_rmem net.ipv4.tcp_wmem

# Aggressive in some kernels — check before changing
sysctl net.ipv4.tcp_congestion_control    # bbr is better on lossy paths
```

## Decision tree

| Signal | Action |
|---|---|
| `TcpRetransSegs/OutSegs > 1%` cluster on one peer | Trace path with mtr, escalate to network |
| Many `TcpExtListenOverflows` | accept queue too small; bump backlog / fix slow accept loop |
| RSTs after idle, app NAT'd | extend conntrack timeout or send keepalive |
| `ss -tin` shows persistent `unacked` + `cwnd:1` | path quality bad, consider BBR |
| Spike of `TcpAbortFailed` | application closing aggressively, not a network bug |

## References

- [TCP Tuning Guide — kernel.org](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- [TCP Probe — instrument TCP at the kernel level](https://lwn.net/Articles/261620/)
- [How TCP Backlog Works in Linux — Veithen](http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html)
- vault: `systems/network/tcp-stack.md`, `systems/network/conntrack.md`
