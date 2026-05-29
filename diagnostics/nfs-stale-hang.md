---
title: NFS Hangs & Stale File Handles
kind: howto
tags: [nfs, storage, mount, hang, d-state, stale-handle, network-fs]
applies_to: [edge, manager]
---

# NFS Hangs & Stale File Handles

Use when processes wedge in uninterruptible `D` state touching an NFS
path, `df` itself hangs, or you see `Stale file handle`. With **hard**
mounts (the safe default) a missing NFS server makes clients block
*forever* rather than error — by design, to avoid data loss. **First
decide: server/network gone (hang) vs. the exported object changed
underneath you (stale handle).**

| Symptom | Probable cause |
|---|---|
| Processes stuck in `D`, `df` hangs | server unreachable + `hard` mount (blocking) |
| `Stale file handle` (ESTALE) | file/export recreated; the handle no longer resolves |
| Intermittent stalls under load | network loss to server / server overloaded |
| `Permission denied` after export change | `no_root_squash`/uid mapping or export options |

## Step 1 — Identify the mount + reach the server

```bash
mount -t nfs,nfs4                     # which paths, and hard vs soft, vers, server
nfsstat -m                            # per-mount options incl. hard/soft, timeo, retrans
# Can we even reach the server's NFS ports?
rpcinfo -p <server> 2>&1 | grep -E 'nfs|mountd'   # v3
nc -vz <server> 2049                  # v4 single port
```

If `rpcinfo`/`nc` to the server fails, it's a reachability problem
(server down / network / firewall), and `hard` mounts are why everything
is blocked rather than erroring.

## Step 2 — Find who is stuck (and on what)

```bash
ps -eo pid,stat,wchan:32,cmd | awk '$2 ~ /D/'    # D-state procs + kernel wait point
# wchan like nfs_* / rpc_* confirms they're blocked in the NFS client
cat /proc/<pid>/stack 2>/dev/null                # exact blocking call (root)
```

D-state on `rpc_wait_bit_killable` / `nfs_*` = blocked on the server.
These won't die on SIGKILL while truly stuck on a hard mount.

## Step 3 — Stale handle (ESTALE) path

`Stale file handle` means the inode/generation the client cached no
longer exists on the server — the directory/file was deleted+recreated,
or the export was re-set. Remount to refresh handles:

```bash
umount -f -l /mnt/nfs   &&  mount /mnt/nfs       # lazy+force, then remount
# Application must re-open files; cached fds stay stale until reopened
```

## Step 4 — Recover a wedged hard mount

```bash
# Lazy unmount detaches the tree so new access stops hanging;
# already-stuck D-state procs only clear once the server returns or reboot.
umount -f -l /mnt/nfs
# Prevention going forward, trading safety for liveness:
#   mount with soft + sane timeo/retrans for non-critical reads, OR
#   hard + intr-equivalent and robust server HA for critical data.
```

Weigh `soft` carefully: it returns errors (and risks silent data loss on
writes) instead of blocking. For critical writes keep `hard` and fix the
server's availability instead.

## Decision tree

| Signal | Action |
|---|---|
| `rpcinfo`/port unreachable, D-state procs | server/network down — restore server; lazy-umount to free new access |
| `Stale file handle` | remount to refresh handles; app must reopen fds |
| Intermittent stalls under load | network loss / server overload — see `diagnostics/network-connectivity.md` |
| Need liveness over safety on non-critical | consider `soft` + tuned `timeo/retrans` |
| Critical writes hanging | keep `hard`; invest in server HA, don't switch to soft |

## References

- [nfs(5) mount options — man7](https://man7.org/linux/man-pages/man5/nfs.5.html)
- [Linux NFS FAQ — nfs.sourceforge.net](https://nfs.sourceforge.net/)
- vault: `diagnostics/disk-io-saturation.md`, `diagnostics/network-connectivity.md`, `diagnostics/high-load-low-cpu.md`
