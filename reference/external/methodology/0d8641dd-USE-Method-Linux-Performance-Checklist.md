---
title: USE Method: Linux Performance Checklist
source: https://www.brendangregg.com/USEmethod/use-linux.html
kind: external
domain: methodology
author: Brendan Gregg
license: fair-use
fetched_at: 2026-05-18
tags: [external, methodology]
---

> [!info] External article · imported reference
> Source: [www.brendangregg.com](https://www.brendangregg.com/USEmethod/use-linux.html)
> Author: Brendan Gregg
> License: fair-use
> Fetched: 2026-05-18

# USE Method: Linux Performance Checklist

This is an example USE-based metric list for Linux operating systems (eg, Ubuntu, CentOS, Fedora). This is primarily intended for system administrators of the physical systems, who are using command line tools. Some of these metrics can be found in remote monitoring tools.

| component | type | metric |
| --- | --- | --- |
| CPU | utilization | system-wide: vmstat 1, "us" + "sy" + "st"; sar -u, sum fields except "%idle" and "%iowait"; dstat -c, sum fields except "idl" and "wai"; per-cpu: mpstat -P ALL 1, sum fields except "%idle" and "%iowait"; sar -P ALL, same as mpstat; per-process: top, "%CPU"; htop, "CPU%"; ps -o pcpu; pidstat 1, "%CPU"; per-kernel-thread: top/htop ("K" to toggle), where VIRT == 0 (heuristic). [1] |
| CPU | saturation | system-wide: vmstat 1, "r" > CPU count [2]; sar -q, "runq-sz" > CPU count; dstat -p, "run" > CPU count; per-process: /proc/PID/schedstat 2nd field (sched_info.run_delay); perf sched latency (shows "Average" and "Maximum" delay per-schedule); dynamic tracing, eg, SystemTap schedtimes.stp "queued(us)" [3] |
| CPU | errors | perf (LPE) if processor specific error events (CPC) are available; eg, AMD64's "04Ah Single-bit ECC Errors Recorded by Scrubber" [4] |
| Memory capacity | utilization | system-wide: free -m, "Mem:" (main memory), "Swap:" (virtual memory); vmstat 1, "free" (main memory), "swap" (virtual memory); sar -r, "%memused"; dstat -m, "free"; slabtop -s c for kmem slab usage; per-process: top/htop, "RES" (resident main memory), "VIRT" (virtual memory), "Mem" for system-wide summary |
| Memory capacity | saturation | system-wide: vmstat 1, "si"/"so" (swapping); sar -B, "pgscank" + "pgscand" (scanning); sar -W; per-process: 10th field (min_flt) from /proc/PID/stat for minor-fault rate, or dynamic tracing [5]; OOM killer: dmesg | grep killed |
| Memory capacity | errors | dmesg for physical failures; dynamic tracing, eg, SystemTap uprobes for failed malloc()s |
| Network Interfaces | utilization | sar -n DEV 1, "rxKB/s"/max "txKB/s"/max; ip -s link, RX/TX tput / max bandwidth; /proc/net/dev, "bytes" RX/TX tput/max; nicstat "%Util" [6] |
| Network Interfaces | saturation | ifconfig, "overruns", "dropped"; netstat -s, "segments retransmited"; sar -n EDEV, *drop and *fifo metrics; /proc/net/dev, RX/TX "drop"; nicstat "Sat" [6]; dynamic tracing for other TCP/IP stack queueing [7] |
| Network Interfaces | errors | ifconfig, "errors", "dropped"; netstat -i, "RX-ERR"/"TX-ERR"; ip -s link, "errors"; sar -n EDEV, "rxerr/s" "txerr/s"; /proc/net/dev, "errs", "drop"; extra counters may be under /sys/class/net/...; dynamic tracing of driver function returns 76] |
| Storage device I/O | utilization | system-wide: iostat -xz 1, "%util"; sar -d, "%util"; per-process: iotop; pidstat -d; /proc/PID/sched "se.statistics.iowait_sum" |
| Storage device I/O | saturation | iostat -xnz 1, "avgqu-sz" > 1, or high "await"; sar -d same; LPE block probes for queue length/latency; dynamic/static tracing of I/O subsystem (incl. LPE block probes) |
| Storage device I/O | errors | /sys/devices/.../ioerr_cnt; smartctl; dynamic/static tracing of I/O subsystem response codes [8] |
| Storage capacity | utilization | swap: swapon -s; free; /proc/meminfo "SwapFree"/"SwapTotal"; file systems: "df -h" |
| Storage capacity | saturation | not sure this one makes sense - once it's full, ENOSPC |
| Storage capacity | errors | strace for ENOSPC; dynamic tracing for ENOSPC; /var/log/messages errs, depending on FS |
| Storage controller | utilization | iostat -xz 1, sum devices and compare to known IOPS/tput limits per-card |
| Storage controller | saturation | see storage device saturation, ... |
| Storage controller | errors | see storage device errors, ... |
| Network controller | utilization | infer from ip -s link (or /proc/net/dev) and known controller max tput for its interfaces |
| Network controller | saturation | see network interface saturation, ... |
| Network controller | errors | see network interface errors, ... |
| CPU interconnect | utilization | LPE (CPC) for CPU interconnect ports, tput / max |
| CPU interconnect | saturation | LPE (CPC) for stall cycles |
| CPU interconnect | errors | LPE (CPC) for whatever is available |
| Memory interconnect | utilization | LPE (CPC) for memory busses, tput / max; or CPI greater than, say, 5; CPC may also have local vs remote counters |
| Memory interconnect | saturation | LPE (CPC) for stall cycles |
| Memory interconnect | errors | LPE (CPC) for whatever is available |
| I/O interconnect | utilization | LPE (CPC) for tput / max if available; inference via known tput from iostat/ip/... |
| I/O interconnect | saturation | LPE (CPC) for stall cycles |
| I/O interconnect | errors | LPE (CPC) for whatever is available |

| component | type | metric |
| --- | --- | --- |
| Kernel mutex | utilization | With CONFIG_LOCK_STATS=y, /proc/lock_stat "holdtime-totat" / "acquisitions" (also see "holdtime-min", "holdtime-max") [8]; dynamic tracing of lock functions or instructions (maybe) |
| Kernel mutex | saturation | With CONFIG_LOCK_STATS=y, /proc/lock_stat "waittime-total" / "contentions" (also see "waittime-min", "waittime-max"); dynamic tracing of lock functions or instructions (maybe); spinning shows up with profiling (perf record -a -g -F 997 ..., oprofile, dynamic tracing) |
| Kernel mutex | errors | dynamic tracing (eg, recusive mutex enter); other errors can cause kernel lockup/panic, debug with kdump/crash |
| User mutex | utilization | valgrind --tool=drd --exclusive-threshold=... (held time); dynamic tracing of lock to unlock function time |
| User mutex | saturation | valgrind --tool=drd to infer contention from held time; dynamic tracing of synchronization functions for wait time; profiling (oprofile, PEL, ...) user stacks for spins |
| User mutex | errors | valgrind --tool=drd various errors; dynamic tracing of pthread_mutex_lock() for EAGAIN, EINVAL, EPERM, EDEADLK, ENOMEM, EOWNERDEAD, ... |
| Task capacity | utilization | top/htop, "Tasks" (current); sysctl kernel.threads-max, /proc/sys/kernel/threads-max (max) |
| Task capacity | saturation | threads blocking on memory allocation; at this point the page scanner should be running (sar -B "pgscan*"), else examine using dynamic tracing |
| Task capacity | errors | "can't fork()" errors; user-level threads: pthread_create() failures with EAGAIN, EINVAL, ...; kernel: dynamic tracing of kernel_thread() ENOMEM |
| File descriptors | utilization | system-wide: sar -v, "file-nr" vs /proc/sys/fs/file-max; dstat --fs, "files"; or just /proc/sys/fs/file-nr; per-process: ls /proc/PID/fd | wc -l vs ulimit -n |
| File descriptors | saturation | does this make sense? I don't think there is any queueing or blocking, other than on memory allocation. |
| File descriptors | errors | strace errno == EMFILE on syscalls returning fds (eg, open(), accept(), ...). |

for the follow-up strategies after identifying a possible bottleneck. If you complete this checklist but still have a performance issue, move onto other strategies: drill-down analysis and latency analysis.

Filling this this checklist has required a lot of research, testing and experimentation. Please reference back to this post if it helps you develop related material.

It's quite possible I've missed something or included the wrong metric somewhere (sorry); I'll update the post to fix these up as they are understood.
