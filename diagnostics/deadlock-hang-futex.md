---
title: Process Hang / Deadlock (Threads Stuck on Locks)
kind: howto
tags: [process, deadlock, hang, futex, threads, mutex, strace]
applies_to: [edge, manager]
---

# Process Hang / Deadlock (Threads Stuck on Locks)

Use when a process is alive but does nothing — requests time out, no log
output, CPU near zero (or one core spinning). Either threads are blocked
on a lock that never releases (deadlock), waiting on IO/a peer that never
answers, or spinning on a contended lock. **Find what the threads are
waiting on before restarting — a restart hides the bug.**

| Symptom | Probable cause |
|---|---|
| Alive, 0% CPU, no progress | threads blocked (futex/IO/network wait) — deadlock or stuck peer |
| One core 100%, no progress | spinlock / busy-wait on a contended lock |
| Hangs only under concurrency | lock-ordering deadlock (A→B vs B→A) |
| Hangs waiting on a downstream | blocked on a dependency that never responds |

## Step 1 — What is each thread doing

```bash
# Per-thread state + where in the kernel they wait
ps -L -o pid,tid,stat,wchan:32,comm -p <pid>
cat /proc/<pid>/task/*/stack 2>/dev/null | sort | uniq -c   # kernel wait points (root)
# Live syscall view — are they all parked in futex()/read()/poll()?
sudo strace -f -p <pid> -e trace=futex,read,write,poll,epoll_wait -tt 2>&1 | head -40
```

All threads parked in `futex(...FUTEX_WAIT...)` = blocked on a userspace
lock (deadlock or a lock holder stuck). All in `read`/`recvfrom` on a
socket = waiting on a peer/dependency (network problem, not a deadlock).

## Step 2 — Get a thread dump (the real diagnosis)

```bash
# Native: full backtrace of every thread
gdb -p <pid> -batch -ex 'thread apply all bt' 2>/dev/null | head -80
# JVM: jstack <pid>   (reports "Found N deadlocks" explicitly)
# Go: send SIGQUIT to dump all goroutine stacks (if not trapped)
```

The thread dump shows the exact lock/call each thread is stuck at. A
classic deadlock: thread A holds lock 1 waiting for lock 2; thread B holds
lock 2 waiting for lock 1. jstack names it outright.

## Step 3 — Deadlock vs stuck-dependency

- **Deadlock** (internal): two+ threads in a lock cycle, all blocked, CPU
  ~0. Fix = lock ordering / timeout-on-acquire in code.
- **Stuck on dependency** (external): threads blocked in
  `read/recv/connect` to a DB/peer that hung. Fix is the dependency + add
  client timeouts (see `diagnostics/high-latency-p99.md`).
- **Livelock/spin**: one core hot, threads spinning retrying — contention,
  not a clean deadlock.

## Step 4 — Capture, then recover

```bash
# Capture a core for offline analysis BEFORE killing (so the bug is debuggable)
gcore <pid>          # writes core.<pid>; analyze with gdb later
# Then restart to restore service
```

Grab the thread dump / core first — restart loses the evidence and the
deadlock will recur.

## Decision tree

| Signal | Action |
|---|---|
| all threads in futex, CPU ~0 | userspace deadlock — thread dump → fix lock order/timeouts |
| threads in read/recv on socket | stuck on a dependency — fix dep + add client timeouts |
| one core spinning | lock contention/livelock — profile the hot lock |
| jstack "Found N deadlocks" | exact cycle reported — fix that lock ordering |
| need service back now | `gcore` first, then restart (keep the evidence) |

## References

- [strace(1) — man7](https://man7.org/linux/man-pages/man1/strace.1.html)
- [futex(2) — man7](https://man7.org/linux/man-pages/man2/futex.2.html)
- vault: `diagnostics/high-latency-p99.md`, `diagnostics/db-connection-pool-exhaustion.md`, `diagnostics/core-dump-analysis.md`
