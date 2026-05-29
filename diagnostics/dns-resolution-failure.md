---
title: DNS Resolution Failures and Slowness
kind: howto
tags: [dns, network, resolv, systemd-resolved, ndots, latency]
applies_to: [edge, manager]
---

# DNS Resolution Failures and Slowness

Use when something "can't connect" but the IP works directly, when name
lookups intermittently hang for ~5s, or when latency has a suspicious
5000ms / 2500ms floor (the classic resolver timeout/retry signature).
**Resolve the name yourself first** — half of "DNS is down" turns out to
be one bad upstream or a `search`-domain explosion, not a dead resolver.

| Symptom | Probable cause class |
|---|---|
| `Name or service not known` immediately | no resolver reachable / empty `resolv.conf` |
| Lookup hangs ~5s then succeeds | first upstream dead, falling back to second |
| Every lookup +5s in containers | `ndots:5` + missing `search` domain → many failed queries |
| Works for A, fails for AAAA (or vice-versa) | upstream/firewall drops one record type |
| Intermittent SERVFAIL under load | upstream rate-limit / UDP conntrack saturation |

## Step 1 — Resolve the name the same way the app does

```bash
getent hosts api.example.com        # uses nsswitch (what glibc apps see)
resolvectl query api.example.com    # systemd-resolved view + which server answered
dig +short api.example.com          # talks to resolv.conf directly, bypasses nss cache
```

`getent` and `dig` disagreeing means the cache / `nsswitch.conf` / a
`hosts:` override is in play — not the upstream. Note the latency `dig`
prints (`Query time: N msec`).

## Step 2 — Inspect the resolver config

```bash
cat /etc/resolv.conf                 # nameserver / search / options ndots
resolvectl status                    # per-link DNS, DNSSEC, fallback (systemd)
cat /etc/nsswitch.conf | grep hosts  # order: files dns ...
```

Red flags:
- Empty / `nameserver 127.0.0.53` but `systemd-resolved` not running
- `options ndots:5` (k8s default) with no matching `search` domain →
  every short name tries N suffixes before the bare name
- A first `nameserver` that is unreachable (each query eats its timeout)

## Step 3 — Test each upstream directly

```bash
# Pull the configured servers and query each in turn
for ns in $(awk '/^nameserver/{print $2}' /etc/resolv.conf); do
  echo "== $ns =="; dig @"$ns" +time=2 +tries=1 +short api.example.com
done
```

A server that times out here is your culprit. If ALL time out but the
host has connectivity, suspect a firewall/conntrack issue on UDP/53
(see Step 5).

## Step 4 — systemd-resolved cache + stats (if used)

```bash
resolvectl statistics                # cache hit rate, transactions, failures
resolvectl flush-caches              # clear a poisoned/stale negative cache
journalctl -u systemd-resolved --since '10 min ago' | tail -40
```

A high failure-transaction count with low cache hits = upstream trouble,
not a local cache problem.

## Step 5 — UDP/53 path + conntrack (the load-related failures)

```bash
# DNS is UDP-first; a full conntrack table silently drops new UDP flows
conntrack -C 2>/dev/null; sysctl net.netfilter.nf_conntrack_count net.netfilter.nf_conntrack_max
# Watch for SERVFAIL/timeout bursts correlating with traffic
sudo tcpdump -nn -i any 'udp port 53' -c 40
```

If conntrack is near max, DNS (lots of short UDP flows) is the first
thing to break — cross-reference `diagnostics/conntrack-table-full.md`.

## Step 6 — Containers / Kubernetes specifics

```bash
# Inside the pod/container
cat /etc/resolv.conf                 # ndots:5, search <ns>.svc.cluster.local ...
nslookup kubernetes.default          # cluster DNS reachable?
# On the node
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50
```

The `ndots:5` + per-query 5s timeout combination is the single most
common "everything is slow in k8s" root cause — fully-qualify hot names
(trailing dot) or lower `ndots` for the workload.

## Decision tree

| Signal | Action |
|---|---|
| `resolv.conf` empty / resolver down | restore nameservers; start `systemd-resolved` |
| First upstream times out, second answers | remove/replace dead upstream; reorder |
| +5s floor only in containers | fix `ndots`/`search`; FQDN the hot lookups |
| All upstreams time out, host has net | check UDP/53 firewall + conntrack saturation |
| SERVFAIL bursts under load | upstream rate-limit; add caching resolver locally |
| AAAA-only failures | disable IPv6 lookups or fix upstream record set |

## References

- [systemd-resolved(8) — freedesktop](https://www.freedesktop.org/software/systemd/man/latest/systemd-resolved.html)
- [Kubernetes DNS for Services and Pods](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [Racy conntrack and DNS 5s timeouts — Weave blog](https://www.weave.works/blog/racy-conntrack-and-dns-lookup-timeouts)
- vault: `diagnostics/conntrack-table-full.md`, `diagnostics/network-connectivity.md`, `systems/network/conntrack.md`
