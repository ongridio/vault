---
title: Capturing & Reading Core Dumps (Crash Triage Basics)
kind: howto
tags: [process, coredump, crash, segfault, gdb, coredumpctl, debugging]
applies_to: [edge, manager]
---

# Capturing & Reading Core Dumps (Crash Triage Basics)

Use after a process crashed (SIGSEGV/SIGABRT, exit 139/134) and you need
to know *where* and *why*. A core dump is a snapshot of the crashed
process's memory + registers; with the matching binary + symbols you get
the crashing stack. **The two failure modes are: no core was captured at
all (config), or you have a core but can't symbolize it (missing debug
info).**

| Symptom | Probable cause |
|---|---|
| Crash but no core file | `ulimit -c 0`, or core_pattern routes elsewhere |
| Core exists, `??` frames in gdb | missing debug symbols / wrong binary |
| Core huge / fills disk | no size limit; huge RSS dumped |
| Container crash, no core on host | core_pattern is host-wide; container nuances |

## Step 1 — Is core capture even enabled?

```bash
ulimit -c                              # 0 = disabled (no core will be written)
cat /proc/sys/kernel/core_pattern      # where cores go (file path, or | pipe to handler)
# systemd-coredump installed? cores go to the journal/coredumpctl, not cwd
coredumpctl list 2>/dev/null | tail
```

`core_pattern` starting with `|` (e.g. `|/lib/systemd/systemd-coredump`)
means cores are managed by `coredumpctl`, not dropped as `core` in the
cwd — look there, don't conclude "no core".

## Step 2 — Enable capture (if it was off)

```bash
ulimit -c unlimited                    # for the shell launching the process
# Persist for a service (systemd):  [Service] LimitCORE=infinity
# Pick a sane pattern with metadata:
sysctl -w kernel.core_pattern='/var/lib/coredumps/core.%e.%p.%t'
mkdir -p /var/lib/coredumps
```

For systemd services, `LimitCORE=infinity` in the unit is what actually
applies (the shell `ulimit` doesn't reach daemons).

## Step 3 — Read the crash stack

```bash
# With systemd-coredump:
coredumpctl gdb <pid-or-name>          # opens gdb on the captured core
# Manual core file:
gdb /path/to/binary /path/to/core -batch -ex 'bt' -ex 'thread apply all bt' 2>/dev/null | head -60
```

The backtrace at the top frame is where it crashed. `thread apply all bt`
shows every thread (the crash may be a side effect of another thread).
`??` frames = missing symbols (Step 4).

## Step 4 — Get symbols (turn `??` into function names)

- Install the matching `-dbg`/`-debuginfo` package, or build with symbols
  (`-g`), or point gdb at a separate `.debug` file.
- The binary used in gdb **must match** the one that crashed (same
  build) — a version mismatch gives garbage frames.
- For stripped release binaries, keep the unstripped build / debuginfo
  archived per release so production cores stay analyzable.

## Decision tree

| Signal | Action |
|---|---|
| no core file | enable `ulimit -c` / `LimitCORE`; check `core_pattern` |
| core_pattern is `|systemd` | use `coredumpctl gdb` — cores are in the journal |
| `??` frames in bt | install matching debug symbols / correct binary |
| core fills disk | bound size / dump dir; rotate cores |
| SIGABRT (134) | often assertion/abort — check logs + bt for the abort() caller |

## References

- [core(5) — man7](https://man7.org/linux/man-pages/man5/core.5.html)
- [coredumpctl(1) — freedesktop](https://www.freedesktop.org/software/systemd/man/latest/coredumpctl.html)
- vault: `diagnostics/crashloop-restart.md`, `diagnostics/deadlock-hang-futex.md`, `reference/linux-commands.md`
