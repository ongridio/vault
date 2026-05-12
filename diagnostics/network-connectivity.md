---
title: Network Connectivity Diagnosis
tags: [network, tcp, dns, firewall]
applies_to: [edge, manager]
---

# Network Connectivity Diagnosis

Use this when a service reports "can't connect", "timing out", or
"intermittent". Walk the stack bottom-up — each step rules out a layer
before you move on.

## Step 1 — NIC / link layer

```bash
ip -s link show
ethtool <iface>
tc -s qdisc show
```

If `RX errors` / `TX errors` / `dropped` keep climbing, you're in
physical / driver / buffer territory. Check cable, switch port LEDs, and
driver version. Anything climbing under load that's flat at idle usually
points at a duplex mismatch or qdisc drops.

## Step 2 — Routing and addressing

```bash
ip addr
ip route
ip -6 route
ip neigh
```

Confirm the target subnet has a route, and that the next-hop gateway is
reachable in the neighbor table. A `FAILED` entry in `ip neigh` for the
gateway is usually L2 (ARP) trouble.

## Step 3 — DNS

```bash
dig +short <hostname>
dig +trace <hostname>
cat /etc/resolv.conf
```

Rule out: upstream resolver down, TTL flap, glibc cache, or
`nsswitch.conf` redirecting through a broken provider (e.g. `mdns` before
`dns`).

## Step 4 — TCP / TLS

```bash
nc -zv <host> <port>
curl -sv --max-time 5 https://<host> 2>&1 | head -30
openssl s_client -connect <host>:443 -servername <host> </dev/null 2>&1 | head -20
```

Watch the three-way handshake, certificate chain, SNI, and ALPN. If you
see `SSL_ERROR_SYSCALL` mid-handshake, look upstream of the load balancer
(it's typically aborting before the cert is sent).

## Step 5 — Firewall / NAT

```bash
iptables -S
nft list ruleset
conntrack -L | wc -l
sysctl net.netfilter.nf_conntrack_max
```

Many `SYN_SENT` or `TIME_WAIT` entries usually mean either:
- the conntrack table is full (compare count vs. `nf_conntrack_max`), or
- the backend is rejecting `SYN` (check upstream `ss -ant 'state syn-recv'`).

## Step 6 — Application

```bash
ss -tnp 'state established'
journalctl -u <service> --since '15 min ago' | tail
```

## Decision tree

- **Physical** → driver / cable / switch port / qdisc drops
- **Routing or addressing** → route table / cloud VPC peering / SG
- **DNS** → upstream resolver / glibc / `resolv.conf`
- **TCP or TLS** → cert / port / firewall / SNI mismatch
- **conntrack full** → bump `nf_conntrack_max`, lengthen keepalive on
  short-lived clients, or move to direct-server-return
