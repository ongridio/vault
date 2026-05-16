---
title: 链接了dpdk的进程启动core在 Illegal instruction
source: https://www.cnblogs.com/cobbliu/p/7777688.html
kind: external
domain: compute
original_date: 2017-11-03
fetched_at: 2026-05-16
bookmark_title: 链接了dpdk的进程启动core在 Illegal instruction - CobbLiu - 博客园
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/cobbliu/p/7777688.html)
> 原始日期：2017-11-03
> 抓取日期：2026-05-16

# 链接了dpdk的进程启动core在 Illegal instruction

# 链接了dpdk的进程启动core在 Illegal instruction

失败后的core栈像下面这样：

Program terminated with signal SIGILL, Illegal instruction. #0 0x00000000036a3fdd in rte_cpu_get_flag_enabled () [Current thread is 1 (Thread 0x7fc26fda21a0 (LWP 10988))] (gdb) bt #0 0x00000000036a3fdd in rte_cpu_get_flag_enabled () #1 0x0000000003694f4e in rte_hash_crc_init_alg () #2 0x000000000388074f in __libc_csu_init () #3 0x00007fc26df6092e in __libc_start_main () from /lib64/libc.so.6 #4 0x0000000000bec929 in _start ()

core的原因很显然："Illegal instruction"，指令非法，查看core处的汇编代码：

shrx指令属于bmi2指令集，查看运行该binary的机器上有无bmi2指令集：

cat /proc/cpuinfo | grep flags

发现该CPU上没有bmi2指令集，所以core掉了。


所以根源是：编译dpdk library的机器的CPU版本更高，支持了bmi2指令，但是运行dpdk的机器的CPU版本更低，不支持bmi2指令。

网上有博文说在编译dpdk的时候将CONFIG_RTE_MACHINE设置成default能解，我们尝试了无果，最后找了一个CPU版本较低的机器编译了整套dpdk library，core未出现。不过这个解法只是权宜之计，长期来看，还是要做好不同CPU和机型的适配工作。