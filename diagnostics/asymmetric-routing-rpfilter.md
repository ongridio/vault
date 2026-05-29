---
title: Asymmetric Routing & rp_filter Drops
kind: howto
tags: [network, routing, rp_filter, multihome, asymmetric, policy-routing]
applies_to: [edge, manager]
---

# Asymmetric Routing & rp_filter Drops

Use on multi-homed hosts (two NICs / two default-ish paths) where traffic
arrives but replies vanish, or connections work one direction only. The
reply leaves via a different interface than the request arrived on
(asymmetric), and Linux **reverse-path filtering** (`rp_filter`) silently
drops packets whose source wouldn't route back out the interface they
came in on. Looks like a firewall drop with no firewall rule.

| Symptom | Probable cause |
|---|---|
| SYN arrives, no reply seen by client | reply routed out the other NIC; or rp_filter dropped inbound |
| Works on NIC A, dies on NIC B | single routing table, all replies via A's default route |
| Silent inbound drops, no fw rule | `rp_filter=1` strict mode dropping asymmetric packets |
| Only some sources affected | those sources' return path differs |

## Step 1 — See the routes + which way replies go

```bash
ip route show; ip route show table all
ip rule show                          # policy routing rules (if any)
# Which source/iface would the kernel use to reach a client?
ip route get <client-ip>
```

If `ip route get <client>` returns a different interface than the one the
request came in on, you have asymmetry — the reply leaves the "wrong" NIC.

## Step 2 — Check rp_filter

```bash
sysctl net.ipv4.conf.all.rp_filter net.ipv4.conf.<iface>.rp_filter
#   1 = strict (drops asymmetric), 2 = loose, 0 = off
# Are drops happening? rp_filter drops are counted here:
nstat -z | grep -i 'IPReversePathFilter'
```

`rp_filter=1` (strict) on a multi-homed host drops perfectly legitimate
asymmetric traffic. A rising `IPReversePathFilter` counter confirms it's
the culprit.

## Step 3 — Confirm with a capture

```bash
# Inbound seen, but no matching outbound on the same iface?
sudo tcpdump -nn -i <iface-in> host <client>     # see the SYN arrive
sudo tcpdump -nn -i <iface-out> host <client>    # reply leaving elsewhere?
```

Request on iface A, reply attempting iface B (or no reply) = asymmetric
routing. Combined with strict rp_filter, the inbound is dropped before
the app even sees it.

## Step 4 — Fix: policy routing or relax rp_filter

- **Proper fix — policy (source) routing**: give each NIC its own routing
  table + `ip rule` so replies leave the same NIC the request arrived on.
  This makes paths symmetric and is the correct multi-homing setup.
- **Quick fix — loosen rp_filter**: `sysctl -w net.ipv4.conf.all.rp_filter=2`
  (loose mode) stops the drops while keeping some anti-spoof protection.
  Set `=0` only if you fully control the network.

## Decision tree

| Signal | Action |
|---|---|
| `ip route get client` → wrong NIC | set up per-NIC policy routing (symmetric replies) |
| `IPReversePathFilter` rising | rp_filter strict dropping — loosen to 2 or fix routing |
| works one NIC only | single default route — add source-based routing |
| inbound seen, no app response | rp_filter dropping before the socket — relax + fix routes |

## References

- [Linux Advanced Routing & Traffic Control (LARTC)](https://lartc.org/)
- [rp_filter — kernel.org ip-sysctl](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- vault: `diagnostics/network-connectivity.md`, `diagnostics/packet-loss-nic-errors.md`, `systems/network/tcp-stack.md`
