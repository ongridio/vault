---
title: Zombie Process Accumulation
kind: howto
tags: [process, zombie, defunct, reaping, pid, init, containers]
applies_to: [edge, manager]
---

# Zombie Process Accumulation

Use when `ps` shows growing `<defunct>` / `Z` processes, or the PID table
fills. A zombie is a finished child whose exit status the parent never
`wait()`ed for — it holds a PID slot but no resources otherwise. A few
transient zombies are normal; **accumulation means a parent isn't reaping,
and unbounded growth exhausts PIDs** (fork failures across the box).

| Symptom | Probable cause |
|---|---|
| Growing `Z`/`<defunct>` under one parent | that parent never calls wait()/reaps |
| Zombies in a container, PID 1 = app | app isn't a proper init; orphans reparent to it |
| `fork: Cannot allocate memory` / PID limit | zombies (+ live procs) exhausted the PID space |
| Zombies clear when parent restarts | confirms parent reaping bug |

## Step 1 — Count + find the parent

```bash
ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/'      # all zombies + their PPID
ps -eo stat | grep -c Z                          # how many
# Who is the negligent parent?
ps -eo pid,ppid,stat,comm | awk '$3 ~ /Z/ {print $2}' | sort | uniq -c | sort -rn
```

The PPID that owns most zombies is the buggy reaper. A zombie's *only*
fix is its parent reaping it (or the parent dying — then init reaps).

## Step 2 — Why the parent isn't reaping

- The parent forks children but never `wait()`/`waitpid()`s (or ignores
  `SIGCHLD` incorrectly).
- The parent is itself stuck/blocked (D-state, deadlocked) and can't run
  its reap loop — check the parent's state too.

```bash
ps -o pid,stat,wchan,comm -p <parent-pid>        # is the parent itself stuck?
```

## Step 3 — Containers: the PID 1 reaping trap

In a container, the app often runs as **PID 1**, which has special
signal/reaping semantics. If the app isn't designed as an init, orphaned
grandchildren reparent to it and pile up as zombies. Run a tiny init
(`tini`, `--init` in Docker, or the runtime's init) as PID 1 to reap
orphans.

## Step 4 — Clear + prevent

```bash
# You can't kill a zombie (it's already dead). Options:
#   - signal the parent to reap: kill -CHLD <parent>   (if it just missed SIGCHLD)
#   - restart the parent (orphans reparent to init, which reaps)
# PID exhaustion check:
cat /proc/sys/kernel/pid_max ; ps -e | wc -l
```

Prevention is in code: parents must reap (`wait()` loop / `SIGCHLD`
handler), and containers should use a real init as PID 1.

## Decision tree

| Signal | Action |
|---|---|
| zombies under one parent | fix parent to wait()/reap; restart it to clear |
| parent stuck (D/blocked) | unblock the parent (it can't reap while stuck) |
| container app = PID 1, zombies | add an init (`tini`/`--init`) to reap orphans |
| PID exhaustion / fork failures | clear zombies + raise `pid_max`; fix the leak |

## References

- [wait(2) / zombie processes — man7](https://man7.org/linux/man-pages/man2/wait.2.html)
- [Docker — specify an init process (--init / tini)](https://docs.docker.com/engine/reference/run/#specify-an-init-process)
- vault: `diagnostics/crashloop-restart.md`, `diagnostics/high-load-low-cpu.md`, `reference/linux-commands.md`
