---
title: nftables / iptables and conntrack
tags: [network, firewall, conntrack, nat]
---

# nftables / iptables and conntrack

The Linux firewall + NAT engine. Modern hosts run **nftables**; many
still use **iptables** front-ends translated by `iptables-nft`. The
**connection tracker (conntrack)** sits underneath both and is where
most "weird intermittent" network problems live.

## The hook chain

Every packet on the host traverses Netfilter hooks in this order:

```
RX driver ─► prerouting ─► [routing] ─► forward ─► postrouting ─► TX driver
                            │                       ▲
                            ▼                       │
                          local ── input ──► local app ── output
```

Tables and what they typically attach to:

- **filter** — packet accept / drop (input / forward / output hooks)
- **nat** — SNAT / DNAT / masquerade (prerouting / postrouting)
- **mangle** — packet rewrite (TOS, marks)
- **raw** — `NOTRACK` opt-out of conntrack

## Reading rules

```bash
nft list ruleset
nft list table inet filter
iptables -S            # legacy view, also works on nftables hosts
iptables -t nat -S
```

The chain a packet hits is deterministic given source / destination /
interface. If you can't figure out *why* a rule is or isn't applying,
trace it:

```bash
nft monitor trace                    # nftables tracing
iptables -t raw -A PREROUTING -p tcp --dport 443 -j TRACE   # legacy
```

## conntrack — the part that bites

conntrack tracks the state of every flow on the host. NAT, return-path
routing for masqueraded traffic, stateful firewall rules, and
DNAT-on-reply all depend on conntrack having the right entry.

```bash
conntrack -L | head
conntrack -S                        # stats (insert_failed, drop, etc.)
sysctl net.netfilter.nf_conntrack_max
sysctl net.netfilter.nf_conntrack_count   # current entry count
```

The two failure modes:

### Table full

`conntrack -S` shows `drop` climbing.
`nf_conntrack_count` near `nf_conntrack_max`. Symptom: new connections
fail randomly, existing ones fine. Fix:

```bash
sysctl -w net.netfilter.nf_conntrack_max=524288
# Persist via /etc/sysctl.d/
```

The default is sized for desktop use. Servers under load **always**
need to raise this.

### Stuck TIME_WAIT entries

NAT'd outbound short-lived connections (curl loops, sidecar probes) at
high rate pile up conntrack entries in TIME_WAIT and starve the table.
Mitigations:

- Lower `nf_conntrack_tcp_timeout_time_wait` (default 120s, often safe
  at 30s)
- Use connection pooling on the client
- Switch to direct-server-return where possible (skip NAT entirely)

## NAT — the SNAT / DNAT pair

- **DNAT** — rewrite destination on the way in (port forwarding, k8s
  service VIPs implemented in iptables mode)
- **SNAT** — rewrite source on the way out (egress masquerade,
  multi-homed routing)

The pair is conntrack-symmetric: a reply packet's conntrack entry was
created on the original direction, and the reply rewrite is computed
from that entry. Lose the entry (TTL, table flush, daemon restart that
drops `--zone-id`) and the reply is dropped.

## Useful tools

```bash
# Show what nftables thinks the rule for this packet is
nft monitor trace

# Track conntrack state changes for a specific flow
conntrack -E -p tcp --dport 443 --src <addr>

# Top sources by conntrack entries
conntrack -L | awk '{print $5}' | sort | uniq -c | sort -rn | head

# Quick legacy <-> nft swap test
iptables-save | head     # are these legacy or nft-shim rules?
```

## How container runtimes interact

- Docker injects an iptables chain on startup. If you wipe `iptables` or
  use a tool that rebuilds rules from scratch (`firewalld --reload`),
  Docker's chain is gone and container networking dies.
- K8s `kube-proxy` in iptables mode adds chains per Service. In ipvs
  mode, the iptables table stays small and IPVS does the load
  balancing. In nftables mode (newer), rules live under `inet kube-proxy`.

When something breaks after a firewall change: it's almost always
about the order in which the container runtime's chains relate to the
new rules. Restart the runtime to repopulate.
