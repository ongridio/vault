---
title: SELinux / AppArmor Denials (Permission Denied With Correct Perms)
kind: howto
tags: [selinux, apparmor, mac, permission-denied, audit, security]
applies_to: [edge, manager]
---

# SELinux / AppArmor Denials (Permission Denied With Correct Perms)

Use for the "but the file permissions are correct!" class of failure: a
process gets `Permission denied` (EACCES) opening a file, binding a port,
or connecting — yet `ls -l`/ownership look fine. A Mandatory Access
Control layer (SELinux on RHEL-family, AppArmor on Debian/Ubuntu) is
denying it by policy, independent of Unix perms. **Confirm MAC is the
cause via the audit log before changing anything.**

| Symptom | Probable cause |
|---|---|
| EACCES with correct Unix perms | SELinux/AppArmor policy denial |
| Works after `setenforce 0` | confirmed SELinux denial |
| Fails only on RHEL/Fedora | SELinux enforcing |
| Fails after moving a file | wrong SELinux context on the new path |
| Service can't bind a non-standard port | SELinux port type not allowed |

## Step 1 — Is MAC enforcing, and is it denying?

```bash
# SELinux
getenforce                          # Enforcing / Permissive / Disabled
ausearch -m AVC,USER_AVC -ts recent 2>/dev/null | tail   # the denials
#   or: journalctl -t setroubleshoot / grep AVC /var/log/audit/audit.log
# AppArmor
aa-status 2>/dev/null                # which profiles are enforcing
journalctl -k | grep -i 'apparmor=.*DENIED' | tail
```

An `AVC ... denied { read }` (SELinux) or `apparmor="DENIED"` (AppArmor)
line naming your process + the target = MAC is the cause. No such line =
it's a real Unix-perm / other issue, not MAC.

## Step 2 — Quick confirm (temporary, not the fix)

```bash
# SELinux: flip to permissive briefly — if it now works, SELinux was it
setenforce 0      # ... test ... then setenforce 1 to re-enable
```

Use this only to *confirm*. **Leaving MAC disabled is not a fix** — it
removes a security layer. Re-enable and craft the right policy/context.

## Step 3 — SELinux: fix context or boolean (don't disable)

```bash
# Wrong file context (common after moving files into a managed path):
ls -Z /path/to/file                          # current context
restorecon -Rv /path                         # reset to policy default
semanage fcontext -a -t <type> '/custom(/.*)?' && restorecon -Rv /custom  # custom path
# Non-standard port for a service:
semanage port -a -t http_port_t -p tcp 8888  # allow nginx/httpd on 8888
# A boolean toggles common allowances (e.g. httpd network connect):
getsebool -a | grep <service>; setsebool -P <bool> on
```

The `ausearch` denial + `sealert`/`audit2why` often suggest the exact
fix. Prefer the narrow fix (context/boolean/port) over disabling.

## Step 4 — AppArmor: adjust the profile

```bash
aa-logprof 2>/dev/null               # interactively update the profile from denials
# Put one profile in complain mode (logs, doesn't block) to gather rules:
aa-complain /etc/apparmor.d/<profile>
# After fixing, return to enforce:
aa-enforce /etc/apparmor.d/<profile>
```

## Decision tree

| Signal | Action |
|---|---|
| AVC denied (SELinux) | fix context (`restorecon`/`semanage fcontext`) or boolean/port |
| works in permissive only | SELinux confirmed — craft policy; re-enable enforcing |
| apparmor DENIED | update profile (`aa-logprof`); enforce after |
| denial after moving files | wrong context — `restorecon -Rv` |
| service can't bind port | `semanage port -a -t <type>` for that port |

## References

- [SELinux User's and Administrator's Guide — RHEL](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/using_selinux/)
- [AppArmor — Ubuntu Server docs](https://ubuntu.com/server/docs/security-apparmor)
- vault: `diagnostics/service-down-cn.md`, `diagnostics/network-connectivity.md`, `reference/linux-commands.md`
