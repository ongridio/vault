---
title: SSH Auth Failures & Brute-Force Floods
kind: howto
tags: [ssh, sshd, auth, brute-force, fail2ban, keys, security]
applies_to: [edge, manager]
---

# SSH Auth Failures & Brute-Force Floods

Use when you can't SSH in ("Permission denied (publickey)"), a flood of
auth failures fills logs / loads the box, or sshd rejects connections.
**Separate your own legitimate auth failing (key/permission/config) from
an external brute-force flood (many failed logins from many IPs).** The
auth log distinguishes them instantly.

| Symptom | Probable cause |
|---|---|
| `Permission denied (publickey)` for you | key not offered/accepted, perms, wrong user |
| Many `Failed password` from varied IPs | brute-force scan (internet noise) |
| `Connection refused` | sshd not running / firewall / wrong port |
| Locked out after failures | fail2ban/sshd banned your IP |
| `Too many authentication failures` | client offering too many keys before the right one |

## Step 1 — Read the auth log (server side)

```bash
journalctl -u ssh -u sshd --since '20 min ago' 2>/dev/null | tail -50
#   or: tail -50 /var/log/auth.log   (Debian)  /  /var/log/secure  (RHEL)
sshd -T 2>/dev/null | grep -iE 'permitrootlogin|passwordauth|pubkeyauth|port|allowusers'
```

`Failed password for invalid user X from <ip>` repeating from many IPs =
brute-force noise (Step 3). A specific `publickey` rejection for *your*
user = your auth problem (Step 2).

## Step 2 — Your own publickey failing

```bash
# Client: see what's actually happening
ssh -vvv user@host 2>&1 | grep -iE 'offer|accept|deny|publickey|permission'
# Server-side perms (sshd refuses keys if these are too open):
ls -ld ~ ~/.ssh; ls -l ~/.ssh/authorized_keys   # ~ not group/world-writable; .ssh 700; authorized_keys 600
```

Top causes: `authorized_keys`/`.ssh`/home with too-loose permissions
(sshd silently ignores the key), the right key not offered (use `-i` /
ssh-agent), wrong username, or `PubkeyAuthentication`/`AllowUsers`
excluding you. `Too many authentication failures` = agent offering many
keys — use `IdentitiesOnly=yes -i <key>`.

## Step 3 — Brute-force flood

A flood of failed logins is mostly internet background scanning. It's
noise unless it loads the box or precedes a breach. Harden:
- Disable password auth (`PasswordAuthentication no`) — keys only kills
  brute force outright.
- `PermitRootLogin no` / `prohibit-password`.
- `fail2ban` to ban offending IPs after N failures; rate-limit/move the
  port; restrict source IPs at the firewall.

```bash
fail2ban-client status sshd 2>/dev/null    # banned IPs (if fail2ban present)
fail2ban-client set sshd unbanip <your-ip> # if YOU got banned
```

## Step 4 — Locked out / refused

```bash
# Refused, not rejected: is sshd up + reachable?
systemctl status sshd; ss -tlnp | grep :22
```

`Connection refused` = sshd down / firewall / wrong port (not an auth
issue). If fail2ban banned your own IP after fat-fingering, unban it
(Step 3). Keep a console/out-of-band path so a lockout isn't terminal.

## Decision tree

| Signal | Action |
|---|---|
| your publickey denied | fix ~/.ssh perms (700/600, home not writable); offer right key |
| `Too many auth failures` | `IdentitiesOnly=yes -i <key>` (stop offering all keys) |
| flood of failed passwords | disable password auth; fail2ban; restrict source IPs |
| connection refused | sshd down / firewall / port — not auth |
| you got banned | `fail2ban-client unbanip`; keep OOB access |

## References

- [sshd_config(5) — man7](https://man7.org/linux/man-pages/man5/sshd_config.5.html)
- [OpenSSH — key-based auth & permissions](https://www.openssh.com/manual.html)
- vault: `diagnostics/network-connectivity.md`, `diagnostics/service-down-cn.md`, `reference/linux-commands.md`
