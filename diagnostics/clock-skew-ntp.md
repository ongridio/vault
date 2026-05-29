---
title: Clock Skew / NTP Drift (Auth, Cert & Replication Failures)
kind: howto
tags: [ntp, clock, time, skew, chrony, tls, jwt, replication]
applies_to: [edge, manager]
---

# Clock Skew / NTP Drift (Auth, Cert & Replication Failures)

Use when failures smell like "time": TLS certs reported expired/not-yet-
valid on one host only, JWT/OAuth tokens rejected as expired/future,
Kerberos auth failing, MFA/TOTP codes wrong, replication or distributed
locks behaving oddly, or logs across hosts that don't line up. Many
security protocols allow only a small clock window; **a drifted clock
breaks them while everything else looks healthy.**

| Symptom | Probable cause |
|---|---|
| Cert "expired"/"not yet valid" on one host | that host's clock is off |
| JWT/token rejected (exp/nbf) | clock skew vs the issuer/validator |
| Kerberos/MFA failures | skew beyond the allowed window (often ±5 min) |
| Cross-host logs misordered | unsynced clocks |

## Step 1 — Is the clock synced + how far off?

```bash
timedatectl                          # System clock synchronized: yes/no, NTP service active
chronyc tracking 2>/dev/null         # System time offset, last sync, stratum
#   or: ntpq -p   (ntpd)
date -u                              # compare to a known-good source mentally
```

`synchronized: no` or a large offset in `chronyc tracking` (`System time
... seconds fast/slow`) is the smoking gun. Compare two hosts' `date -u`
to see relative skew.

## Step 2 — Why sync is failing

```bash
systemctl status chronyd 2>/dev/null || systemctl status systemd-timesyncd
chronyc sources -v 2>/dev/null       # are upstream NTP servers reachable + selected?
journalctl -u chronyd --since '1 hour ago' | tail
```

Common causes: NTP service not running, UDP/123 blocked by a firewall, no
reachable upstream (air-gapped without an internal NTP server), or a VM
whose host clock drifts and guest sync is off.

## Step 3 — Correct it

```bash
# Force an immediate step + verify
chronyc makestep 2>/dev/null         # step the clock now (chrony)
systemctl enable --now chronyd       # ensure it runs on boot
# Air-gapped: point at an internal NTP server (or a host with a good clock)
```

A large step can disrupt apps that assume monotonic wall-clock — prefer
fixing sync and letting it slew, but for auth-breaking skew an immediate
`makestep` restores service fastest.

## Step 4 — Prevent + detect

- Run NTP (chrony) on every host; in air-gapped sites stand up an
  internal NTP server (one host syncs externally / uses GPS, others sync
  to it).
- Open UDP/123 where NTP must reach upstreams.
- Alert on `chronyc tracking` offset / `timedatectl` not-synchronized so
  skew is caught before it breaks TLS/auth.

## Decision tree

| Signal | Action |
|---|---|
| `synchronized: no` | start/enable chronyd/timesyncd; check upstreams |
| large offset | `chronyc makestep` to restore auth now; fix sync |
| NTP can't reach upstream | open UDP/123; or internal NTP server (air-gapped) |
| only VMs drift | enable guest time sync; fix host clock |
| cert/JWT failures clear after sync | confirmed clock skew — add offset alerting |

## References

- [chrony documentation](https://chrony-project.org/documentation.html)
- [timedatectl / systemd-timesyncd — freedesktop](https://www.freedesktop.org/software/systemd/man/latest/timedatectl.html)
- vault: `diagnostics/tls-handshake-failure.md`, `diagnostics/jwt-auth-failures.md`, `diagnostics/k8s-node-notready.md`
