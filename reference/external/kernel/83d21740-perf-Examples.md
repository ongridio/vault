---
title: perf Examples
source: http://www.brendangregg.com/perf.html
kind: external
domain: kernel
author: Brendan Gregg
original_date: 2014-06-22
fetched_at: 2026-05-16
bookmark_title: Linux perf Examples
tags: [external, kernel]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.brendangregg.com](http://www.brendangregg.com/perf.html)
> 作者：Brendan Gregg
> 原始日期：2014-06-22
> 抓取日期：2026-05-16

# perf Examples

# perf Examples

These are some examples of using the perf Linux profiler, which has also been called Performance Counters for Linux (PCL), Linux perf events (LPE), or perf_events. Like Vince Weaver, I'll call it perf_events so that you can search on that term later. Searching for just "perf" finds sites on the police, petroleum, weed control, and a T-shirt. This is not an official perf page, for either perf_events or the T-shirt.

perf_events is an event-oriented observability tool, which can help you solve advanced performance and troubleshooting functions. Questions that can be answered include:

- Why is the kernel on-CPU so much? What code-paths?
- Which code-paths are causing CPU level 2 cache misses?
- Are the CPUs stalled on memory I/O?
- Which code-paths are allocating memory, and how much?
- What is triggering TCP retransmits?
- Is a certain kernel function being called, and how often?
- What reasons are threads leaving the CPU?

perf_events is part of the Linux kernel, under tools/perf. While it uses many Linux tracing features, some are not yet exposed via the perf command, and need to be used via the ftrace interface instead. My perf-tools collection (github) uses both perf_events and ftrace as needed.

This page includes my examples of perf_events. A table of contents:


Key sections to start with are: Events, One-Liners, Presentations, Prerequisites, CPU statistics, Timed Profiling, and Flame Graphs. Also see my Posts about perf_events, and Links for the main (official) perf_events page, awesome tutorial, and other links. The next sections introduce perf_events further, starting with a screenshot, one-liners, and then background.

This page is under construction, and there's a lot more to perf_events that I'd like to add. Hopefully this is useful so far.

## 1. Screenshot

Starting with a screenshot, here's perf version 3.9.3 tracing disk I/O:

#perf record -e block:block_rq_issue -ag^C #ls -l perf.data-rw------- 1 root root 3458162 Jan 26 03:03 perf.data #perf report[...] # Samples: 2K of event 'block:block_rq_issue' # Event count (approx.): 2216 # # Overhead Command Shared Object Symbol # ........ ............ ................. .................... # 32.13% dd [kernel.kallsyms] [k] blk_peek_request | --- blk_peek_request virtblk_request __blk_run_queue | |--98.31%-- queue_unplugged | blk_flush_plug_list | | | |--91.00%-- blk_queue_bio | | generic_make_request | | submit_bio | | ext4_io_submit | | | | | |--58.71%-- ext4_bio_write_page | | | mpage_da_submit_io | | | mpage_da_map_and_submit | | | write_cache_pages_da | | | ext4_da_writepages | | | do_writepages | | | __filemap_fdatawrite_range | | | filemap_flush | | | ext4_alloc_da_blocks | | | ext4_release_file | | | __fput | | | ____fput | | | task_work_run | | | do_notify_resume | | | int_signal | | | close | | | 0x0 | | | | | --41.29%-- mpage_da_submit_io [...]

A `perf record` command was used to trace the block:block_rq_issue probe, which fires when a block device I/O request is issued (disk I/O). Options included -a to trace all CPUs, and -g to capture call graphs (stack traces). Trace data is written to a perf.data file, and tracing ended when Ctrl-C was hit. A summary of the perf.data file was printed using `perf report`, which builds a tree from the stack traces, coalescing common paths, and showing percentages for each path.

The perf report output shows that 2,216 events were traced (disk I/O), 32% of which from a `dd` command. These were issued by the kernel function blk_peek_request(), and walking down the stacks, about half of these 32% were from the close() system call.

Note that I use the "#" prompt to signify that these commands were run as root, and I'll use "$" for user commands. Use sudo as needed.

## 2. One-Liners

Some useful one-liners I've gathered or written. Terminology I'm using, from lowest to highest overhead:

**statistics**/**count**: increment an integer counter on events**sample**: collect details (eg, instruction pointer or stack) from a subset of events (once every ...)**trace**: collect details from every event

### Listing Events

# Listing all currently known events: perf list # Listing sched tracepoints: perf list 'sched:*'

### Counting Events

# CPU counter statistics for the specified command: perf statcommand# Detailed CPU counter statistics (includes extras) for the specified command: perf stat -dcommand# CPU counter statistics for the specified PID, until Ctrl-C: perf stat -pPID# CPU counter statistics for the entire system, for 5 seconds: perf stat -a sleep 5 # Various basic CPU statistics, system wide, for 10 seconds: perf stat -e cycles,instructions,cache-references,cache-misses,bus-cycles -a sleep 10 # Various CPU level 1 data cache statistics for the specified command: perf stat -e L1-dcache-loads,L1-dcache-load-misses,L1-dcache-storescommand# Various CPU data TLB statistics for the specified command: perf stat -e dTLB-loads,dTLB-load-misses,dTLB-prefetch-missescommand# Various CPU last level cache statistics for the specified command: perf stat -e LLC-loads,LLC-load-misses,LLC-stores,LLC-prefetchescommand# Using raw PMC counters, eg, counting unhalted core cycles: perf stat -e r003c -a sleep 5 # PMCs: counting cycles and frontend stalls via raw specification: perf stat -e cycles -e cpu/event=0x0e,umask=0x01,inv,cmask=0x01/ -a sleep 5 # Count syscalls per-second system-wide: perf stat -e raw_syscalls:sys_enter -I 1000 -a # Count system calls by type for the specified PID, until Ctrl-C: perf stat -e 'syscalls:sys_enter_*' -pPID# Count system calls by type for the entire system, for 5 seconds: perf stat -e 'syscalls:sys_enter_*' -a sleep 5 # Count scheduler events for the specified PID, until Ctrl-C: perf stat -e 'sched:*' -pPID# Count scheduler events for the specified PID, for 10 seconds: perf stat -e 'sched:*' -pPIDsleep 10 # Count ext4 events for the entire system, for 10 seconds: perf stat -e 'ext4:*' -a sleep 10 # Count block device I/O events for the entire system, for 10 seconds: perf stat -e 'block:*' -a sleep 10 # Count all vmscan events, printing a report every second: perf stat -e 'vmscan:*' -a -I 1000

### Profiling

# Sample on-CPU functions for the specified command, at 99 Hertz: perf record -F 99command# Sample on-CPU functions for the specified PID, at 99 Hertz, until Ctrl-C: perf record -F 99 -pPID# Sample on-CPU functions for the specified PID, at 99 Hertz, for 10 seconds: perf record -F 99 -pPIDsleep 10 # Sample CPU stack traces (via frame pointers) for the specified PID, at 99 Hertz, for 10 seconds: perf record -F 99 -pPID-g -- sleep 10 # Sample CPU stack traces for the PID, using dwarf (dbg info) to unwind stacks, at 99 Hertz, for 10 seconds: perf record -F 99 -pPID--call-graph dwarf sleep 10 # Sample CPU stack traces for the entire system, at 99 Hertz, for 10 seconds (< Linux 4.11): perf record -F 99 -ag -- sleep 10 # Sample CPU stack traces for the entire system, at 99 Hertz, for 10 seconds (>= Linux 4.11): perf record -F 99 -g -- sleep 10 # If the previous command didn't work, try forcing perf to use the cpu-clock event: perf record -F 99 -e cpu-clock -ag -- sleep 10 # Sample CPU stack traces for a container identified by its /sys/fs/cgroup/perf_event cgroup: perf record -F 99 -e cpu-clock --cgroup=docker/1d567f4393190204...etc...-a -- sleep 10 # Sample CPU stack traces for the entire system, with dwarf stacks, at 99 Hertz, for 10 seconds: perf record -F 99 -a --call-graph dwarf sleep 10 # Sample CPU stack traces for the entire system, using last branch record for stacks, ... (>= Linux 4.?): perf record -F 99 -a --call-graph lbr sleep 10 # Sample CPU stack traces, once every 10,000 Level 1 data cache misses, for 5 seconds: perf record -e L1-dcache-load-misses -c 10000 -ag -- sleep 5 # Sample CPU stack traces, once every 100 last level cache misses, for 5 seconds: perf record -e LLC-load-misses -c 100 -ag -- sleep 5 # Sample on-CPU kernel instructions, for 5 seconds: perf record -e cycles:k -a -- sleep 5 # Sample on-CPU user instructions, for 5 seconds: perf record -e cycles:u -a -- sleep 5 # Sample on-CPU user instructions precisely (using PEBS), for 5 seconds: perf record -e cycles:up -a -- sleep 5 # Perform branch tracing (needs HW support), for 1 second: perf record -b -a sleep 1 # Sample CPUs at 49 Hertz, and show top addresses and symbols, live (no perf.data file): perf top -F 49 # Sample CPUs at 49 Hertz, and show top process names and segments, live: perf top -F 49 -ns comm,dso

### Static Tracing

# Trace new processes, until Ctrl-C: perf record -e sched:sched_process_exec -a # Sample (take a subset of) context-switches, until Ctrl-C: perf record -e context-switches -a # Trace all context-switches, until Ctrl-C: perf record -e context-switches -c 1 -a # Include raw settings used (see: man perf_event_open): perf record -vv -e context-switches -a # Trace all context-switches via sched tracepoint, until Ctrl-C: perf record -e sched:sched_switch -a # Sample context-switches with stack traces, until Ctrl-C: perf record -e context-switches -ag # Sample context-switches with stack traces, for 10 seconds: perf record -e context-switches -ag -- sleep 10 # Sample CS, stack traces, and with timestamps (< Linux 3.17, -T now default): perf record -e context-switches -ag -T # Sample CPU migrations, for 10 seconds: perf record -e migrations -a -- sleep 10 # Trace all connect()s with stack traces (outbound connections), until Ctrl-C: perf record -e syscalls:sys_enter_connect -ag # Trace all accepts()s with stack traces (inbound connections), until Ctrl-C: perf record -e syscalls:sys_enter_accept* -ag # Trace all block device (disk I/O) requests with stack traces, until Ctrl-C: perf record -e block:block_rq_insert -ag # Sample at most 100 block device requests per second, until Ctrl-C: perf record -F 100 -e block:block_rq_insert -a # Trace all block device issues and completions (has timestamps), until Ctrl-C: perf record -e block:block_rq_issue -e block:block_rq_complete -a # Trace all block completions, of size at least 100 Kbytes, until Ctrl-C: perf record -e block:block_rq_complete --filter 'nr_sector > 200' # Trace all block completions, synchronous writes only, until Ctrl-C: perf record -e block:block_rq_complete --filter 'rwbs == "WS"' # Trace all block completions, all types of writes, until Ctrl-C: perf record -e block:block_rq_complete --filter 'rwbs ~ "*W*"' # Sample minor faults (RSS growth) with stack traces, until Ctrl-C: perf record -e minor-faults -ag # Trace all minor faults with stack traces, until Ctrl-C: perf record -e minor-faults -c 1 -ag # Sample page faults with stack traces, until Ctrl-C: perf record -e page-faults -ag # Trace all ext4 calls, and write to a non-ext4 location, until Ctrl-C: perf record -e 'ext4:*' -o /tmp/perf.data -a # Trace kswapd wakeup events, until Ctrl-C: perf record -e vmscan:mm_vmscan_wakeup_kswapd -ag # Add Node.js USDT probes (Linux 4.10+): perf buildid-cache --add `which node` # Trace the node http__server__request USDT event (Linux 4.10+): perf record -e sdt_node:http__server__request -a

### Dynamic Tracing

# Add a tracepoint for the kernel tcp_sendmsg() function entry ("--add" is optional): perf probe --add tcp_sendmsg # Remove the tcp_sendmsg() tracepoint (or use "--del"): perf probe -d tcp_sendmsg # Add a tracepoint for the kernel tcp_sendmsg() function return: perf probe 'tcp_sendmsg%return' # Show available variables for the kernel tcp_sendmsg() function (needs debuginfo): perf probe -V tcp_sendmsg # Show available variables for the kernel tcp_sendmsg() function, plus external vars (needs debuginfo): perf probe -V tcp_sendmsg --externs # Show available line probes for tcp_sendmsg() (needs debuginfo): perf probe -L tcp_sendmsg # Show available variables for tcp_sendmsg() at line number 81 (needs debuginfo): perf probe -V tcp_sendmsg:81 # Add a tracepoint for tcp_sendmsg(), with three entry argument registers (platform specific): perf probe 'tcp_sendmsg %ax %dx %cx' # Add a tracepoint for tcp_sendmsg(), with an alias ("bytes") for the %cx register (platform specific): perf probe 'tcp_sendmsg bytes=%cx' # Trace previously created probe when the bytes (alias) variable is greater than 100: perf record -e probe:tcp_sendmsg --filter 'bytes > 100' # Add a tracepoint for tcp_sendmsg() return, and capture the return value: perf probe 'tcp_sendmsg%return $retval' # Add a tracepoint for tcp_sendmsg(), and "size" entry argument (reliable, but needs debuginfo): perf probe 'tcp_sendmsg size' # Add a tracepoint for tcp_sendmsg(), with size and socket state (needs debuginfo): perf probe 'tcp_sendmsg size sk->__sk_common.skc_state' # Tell me how on Earth you would do this, but don't actually do it (needs debuginfo): perf probe -nv 'tcp_sendmsg size sk->__sk_common.skc_state' # Trace previous probe when size is non-zero, and state is not TCP_ESTABLISHED(1) (needs debuginfo): perf record -e probe:tcp_sendmsg --filter 'size > 0 && skc_state != 1' -a # Add a tracepoint for tcp_sendmsg() line 81 with local variable seglen (needs debuginfo): perf probe 'tcp_sendmsg:81 seglen' # Add a tracepoint for do_sys_open() with the filename as a string (needs debuginfo): perf probe 'do_sys_open filename:string' # Add a tracepoint for myfunc() return, and include the retval as a string: perf probe 'myfunc%return +0($retval):string' # Add a tracepoint for the user-level malloc() function from libc: perf probe -x /lib64/libc.so.6 malloc # Add a tracepoint for this user-level static probe (USDT, aka SDT event): perf probe -x /usr/lib64/libpthread-2.24.so %sdt_libpthread:mutex_entry # List currently available dynamic probes: perf probe -l

### Mixed

# Trace system calls by process, showing a summary refreshing every 2 seconds: perf top -e raw_syscalls:sys_enter -ns comm # Trace sent network packets by on-CPU process, rolling output (no clear): stdbuf -oL perf top -e net:net_dev_xmit -ns comm | strings # Sample stacks at 99 Hertz, and, context switches: perf record -F99 -e cpu-clock -e cs -a -g # Sample stacks to 2 levels deep, and, context switch stacks to 5 levels (needs 4.8): perf record -F99 -e cpu-clock/max-stack=2/ -e cs/max-stack=5/ -a -g

### Special

# Record cacheline events (Linux 4.10+): perf c2c record -a -- sleep 10 # Report cacheline events from previous recording (Linux 4.10+): perf c2c report

### Reporting

# Show perf.data in an ncurses browser (TUI) if possible: perf report # Show perf.data with a column for sample count: perf report -n # Show perf.data as a text report, with data coalesced and percentages: perf report --stdio # Report, with stacks in folded format: one line per stack (needs 4.4): perf report --stdio -n -g folded # List all events from perf.data: perf script # List all perf.data events, with data header (newer kernels; was previously default): perf script --header # List all perf.data events, with customized fields (< Linux 4.1): perf script -f time,event,trace # List all perf.data events, with customized fields (>= Linux 4.1): perf script -F time,event,trace # List all perf.data events, with my recommended fields (needs record -a; newer kernels): perf script --header -F comm,pid,tid,cpu,time,event,ip,sym,dso # List all perf.data events, with my recommended fields (needs record -a; older kernels): perf script -f comm,pid,tid,cpu,time,event,ip,sym,dso # Dump raw contents from perf.data as hex (for debugging): perf script -D # Disassemble and annotate instructions with percentages (needs some debuginfo): perf annotate --stdio

These one-liners serve to illustrate the capabilities of perf_events, and can also be used a bite-sized tutorial: learn perf_events one line at a time. You can also print these out as a perf_events cheatsheet.

## 3. Presentations

### Kernel Recipes (2017)

At Kernel Recipes 2017 I gave an updated talk on Linux perf at Netflix, focusing on getting CPU profiling and flame graphs to work. This talk includes a crash course on perf_events, plus gotchas such as fixing stack traces and symbols when profiling Java, Node.js, VMs, and containers.

A video of the talk is on youtube and the slides are on slideshare:

There's also an older version of this talk from 2015, which I've posted about.

# 4. Background

The following sections provide some background for understanding perf_events and how to use it. I'll describe the prerequisites, audience, usage, events, and tracepoints.

## 4.1. Prerequisites

The `perf` tool is in the **linux-tools-common** package. Start by adding that, then running "`perf`" to see if you get the USAGE message. It may tell you to install another related package (linux-tools-*kernelversion*).

You can also build and add `perf` from the Linux kernel source. See the Building section.

To get the most out `perf`, you'll want symbols and stack traces. These may work by default in your Linux distribution, or they may require the addition of packages, or recompilation of the kernel with additional config options.

## 4.2. Symbols

perf_events, like other debug tools, needs symbol information (symbols). These are used to translate memory addresses into function and variable names, so that they can be read by us humans. Without symbols, you'll see hexadecimal numbers representing the memory addresses profiled.

The following `perf report` output shows stack traces, however, only hexadecimal numbers can be seen:

57.14% sshd libc-2.15.so [.] connect | --- connect | |--25.00%-- 0x7ff3c1cddf29 | |--25.00%-- 0x7ff3bfe82761 | 0x7ff3bfe82b7c | |--25.00%-- 0x7ff3bfe82dfc --25.00%-- [...]

If the software was added by packages, you may find debug packages (often "-dbgsym") which provide the symbols. Sometimes `perf report` will tell you to install these, eg: "no symbols found in /bin/dd, maybe install a debug package?".

Here's the same `perf report` output seen earlier, after adding openssh-server-dbgsym and libc6-dbgsym (this is on ubuntu 12.04):

57.14% sshd libc-2.15.so [.] __GI___connect_internal | --- __GI___connect_internal | |--25.00%-- add_one_listen_addr.isra.0 | |--25.00%-- __nscd_get_mapping | __nscd_get_map_ref | |--25.00%-- __nscd_open_socket --25.00%-- [...]

I find it useful to add both libc6-dbgsym and coreutils-dbgsym, to provide some symbol coverage of user-level OS codepaths.

Another way to get symbols is to compile the software yourself. For example, I just compiled node (Node.js):

# file node-v0.10.28/out/Release/node node-v0.10.28/out/Release/node: ELF 64-bit LSB executable, ...not stripped

This has not been stripped, so I can profile node and see more than just hex. If the result is stripped, configure your build system not to run strip(1) on the output binaries.

Kernel-level symbols are in the kernel debuginfo package, or when the kernel is compiled with CONFIG_KALLSYMS.

## 4.3. JIT Symbols (Java, Node.js)

Programs that have virtual machines (VMs), like Java's JVM and node's v8, execute their own virtual processor, which has its own way of executing functions and managing stacks. If you profile these using perf_events, you'll see symbols for the VM engine, which have some use (eg, to identify if time is spent in GC), but you won't see the language-level context you might be expecting. Eg, you won't see Java classes and methods.

perf_events has JIT support to solve this, which requires the VM to maintain a /tmp/perf-PID.map file for symbol translation. Java can do this with perf-map-agent, and Node.js 0.11.13+ with --perf_basic_prof. See my blog post Node.js flame graphs on Linux for the steps.

Note that Java may not show full stacks to begin with, due to hotspot on x86 omitting the frame pointer (just like gcc). On newer versions (JDK 8u60+), you can use the -XX:+PreserveFramePointer option to fix this behavior, and profile fully using perf. See my Netflix Tech Blog post, Java in Flames, for a full writeup, and my Java flame graphs section, which links to an older patch and includes an example resulting flame graph. I also summarized the latest in my JavaOne 2016 talk Java Performance Analysis on Linux with Flame Graphs.

## 4.4 Stack Traces

Always compile with frame pointers. Omitting frame pointers is an evil compiler optimization that breaks debuggers, and sadly, is often the default. Without them, you may see incomplete stacks from perf_events, like seen in the earlier sshd symbols example. There are three ways to fix this: either using dwarf data to unwind the stack, using last branch record (LBR) if available (a processor feature), or returning the frame pointers.

There are other stack walking techniques, like BTS (Branch Trace Store), and the new ORC unwinder. I'll add docs for them at some point (and as perf support arrives).

**Frame Pointers**

The earlier sshd example was a default build of OpenSSH, which uses compiler optimizations (-O2), which in this case has omitted the frame pointer. Here's how it looks after recompiling OpenSSH with **-fno-omit-frame-pointer**:

100.00% sshd libc-2.15.so [.] __GI___connect_internal | --- __GI___connect_internal | |--30.00%-- add_one_listen_addr.isra.0 | add_listen_addr | fill_default_server_options | main | __libc_start_main | |--20.00%-- __nscd_get_mapping | __nscd_get_map_ref | |--20.00%-- __nscd_open_socket --30.00%-- [...]

Now the ancestry from add_one_listen_addr() can be seen, down to main() and __libc_start_main().

The kernel can suffer the same problem. Here's an example CPU profile collected on an idle server, with stack traces (-g):

99.97% swapper [kernel.kallsyms] [k] default_idle | --- default_idle 0.03% sshd [kernel.kallsyms] [k] iowrite16 | --- iowrite16 __write_nocancel (nil)

The kernel stack traces are incomplete. Now a similar profile with **CONFIG_FRAME_POINTER=y**:

99.97% swapper [kernel.kallsyms] [k] default_idle | --- default_idle cpu_idle | |--87.50%-- start_secondary | --12.50%-- rest_init start_kernel x86_64_start_reservations x86_64_start_kernel 0.03% sshd [kernel.kallsyms] [k] iowrite16 | --- iowrite16 vp_notify virtqueue_kick start_xmit dev_hard_start_xmit sch_direct_xmit dev_queue_xmit ip_finish_output ip_output ip_local_out ip_queue_xmit tcp_transmit_skb tcp_write_xmit __tcp_push_pending_frames tcp_sendmsg inet_sendmsg sock_aio_write do_sync_write vfs_write sys_write system_call_fastpath __write_nocancel

Much better -- the entire path from the write() syscall (__write_nocancel) to iowrite16() can be seen.

**Dwarf**

Since about the 3.9 kernel, perf_events has supported a workaround for missing frame pointers in user-level stacks: libunwind, which uses dwarf. This can be enabled using "--call-graph dwarf" (or "-g dwarf").

Also see the Building section for other notes about building perf_events, as without the right library, it may build itself without dwarf support.

**LBR**

You must have Last Branch Record access to be able to use this. It is disabled in most cloud environments, where you'll get this error:

#perf record -F 99 -a --call-graph lbrError: PMU Hardware doesn't support sampling/overflow-interrupts.

Here's an example of it working:

#perf record -F 99 -a --call-graph lbr^C[ perf record: Woken up 1 times to write data ] [ perf record: Captured and wrote 0.903 MB perf.data (163 samples) ] #perf script[...] stackcollapse-p 23867 [007] 4762187.971824: 29003297 cycles:ppp: 1430c0 Perl_re_intuit_start (/usr/bin/perl) 144118 Perl_regexec_flags (/usr/bin/perl) cfcc9 Perl_pp_match (/usr/bin/perl) cbee3 Perl_runops_standard (/usr/bin/perl) 51fb3 perl_run (/usr/bin/perl) 2b168 main (/usr/bin/perl) stackcollapse-p 23867 [007] 4762187.980184: 31532281 cycles:ppp: e3660 Perl_sv_force_normal_flags (/usr/bin/perl) 109b86 Perl_leave_scope (/usr/bin/perl) 1139db Perl_pp_leave (/usr/bin/perl) cbee3 Perl_runops_standard (/usr/bin/perl) 51fb3 perl_run (/usr/bin/perl) 2b168 main (/usr/bin/perl) stackcollapse-p 23867 [007] 4762187.989283: 32341031 cycles:ppp: cfae0 Perl_pp_match (/usr/bin/perl) cbee3 Perl_runops_standard (/usr/bin/perl) 51fb3 perl_run (/usr/bin/perl) 2b168 main (/usr/bin/perl)

Nice! Note that LBR is usually limited in stack depth (either 8, 16, or 32 frames), so it may not be suitable for deep stacks or flame graph generation, as flame graphs need to walk to the common root for merging.

Here's that same program sampled using the by-default frame pointer walk:

#perf record -F 99 -a -g^C[ perf record: Woken up 1 times to write data ] [ perf record: Captured and wrote 0.882 MB perf.data (81 samples) ] #perf script[...] stackcollapse-p 23883 [005] 4762405.747834: 35044916 cycles:ppp: 135b83 [unknown] (/usr/bin/perl) stackcollapse-p 23883 [005] 4762405.757935: 35036297 cycles:ppp: ee67d Perl_sv_gets (/usr/bin/perl) stackcollapse-p 23883 [005] 4762405.768038: 35045174 cycles:ppp: 137334 [unknown] (/usr/bin/perl)

You can recompile Perl with frame pointer support (in its ./Configure, it asks what compiler options: add -fno-omit-frame-pointer). Or you can use LBR if it's available, and you don't need very long stacks.

## 4.5. Audience

To use perf_events, you'll either:

- Develop y

[... 内容超长，已截断；完整原文见 source URL ...]
