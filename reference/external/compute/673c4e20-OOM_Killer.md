---
title: OOM_Killer
source: https://linux-mm.org/OOM_Killer
kind: external
domain: compute
original_date: 2017-12-30
fetched_at: 2026-05-16
bookmark_title: OOM_Killer - linux-mm.org Wiki
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[linux-mm.org](https://linux-mm.org/OOM_Killer)
> 原始日期：2017-12-30
> 抓取日期：2026-05-16

# OOM_Killer

The functions, code excerpts and comments discussed below here are from mm/oom_kill.c unless otherwise noted.

It is the job of the linux 'oom killer' to sacrifice one or more processes in order to free up memory for the system when all else fails. It will also kill any process sharing the same mm_struct as the selected process, for obvious reasons. Any particular process leader may be immunized against the oom killer if the value of its /proc/<pid>/oomadj is set to the constant OOM_DISABLE (currently defined as -17).

The function which does the actual scoring of a process in the effort to find the best candidate for elimination is called badness(), which results from the following call chain:

_alloc_pages -> out_of_memory() -> select_bad_process() -> badness()


*The comments to badness() pretty well speak for themselves:*

/* * oom_badness - calculate a numeric value for how bad this task has been * @p: task struct of which task we should calculate * @p: current uptime in seconds * * The formula used is relatively simple and documented inline in the * function. The main rationale is that we want to select a good task * to kill when we run out of memory. * * Good in this context means that: * 1) we lose the minimum amount of work done * 2) we recover a large amount of memory * 3) we don't kill anything innocent of eating tons of memory * 4) we want to kill the minimum amount of processes (one) * 5) we try to kill the process the user expects us to kill, this * algorithm has been meticulously tuned to meet the principle * of least surprise ... (be careful when you change it) */

badness() works by accumulating 'points' for each process it examines and returning them to select_bad_process(). The process with the highest number of points, loses and is ultimately eliminated, unless it is already in the midst of freeing up memory on its own.

The scoring of a process starts with the size of its resident memory:

/* * The memory size of the process is the basis for the badness. */ points = p->mm->total_vm;

The independent memory size of any child (except a kernel thread) is added to the score:

/* * Processes which fork a lot of child processes are likely * a good choice. We add the vmsize of the childs if they * have an own mm. This prevents forking servers to flood the * machine with an endless amount of childs */ ... if (chld->mm != p->mm && chld->mm) points += chld->mm->total_vm;

'Niced' processes have their scores increased, and long running processes have their scores decreased:

s = int_sqrt(cpu_time); if (s) points /= s; s = int_sqrt(int_sqrt(run_time)); if (s) points /= s; /* * Niced processes are most likely less important, so double * their badness points. */ if (task_nice(p) > 0) points *= 2;

Processes with CAP_SYS_ADMIN and CAP_SYS_RAWIO, respectively, each have their scores reduced:

/* * Superuser processes are usually more important, so we make it * less likely that we kill those. */ if (cap_t(p->cap_effective) & CAP_TO_MASK(CAP_SYS_ADMIN) || p->uid == 0 || p->euid == 0) points /= 4; /* * We don't want to kill a process with direct hardware access. * Not only could that mess up the hardware, but usually users * tend to only have this flag set on applications they think * of as important. */ if (cap_t(p->cap_effective) & CAP_TO_MASK(CAP_SYS_RAWIO)) points /= 4;

Finally the accumulated score is bitshifted by the user-settable value of /proc/<pid>/oomadj:

/* * Adjust the score by oomkilladj. */ if (p->oomkilladj) { if (p->oomkilladj > 0) points <<= p->oomkilladj; else points >>= -(p->oomkilladj); }

So the ideal candidate for liquidation is a recently started, non privileged process which together with its children uses lots of memory, has been nice'd, and does no raw I/O. Something like a nohup'd parallel kernel build (which is not a bad choice since all results are saved to disk and very little work is lost when a 'make' is terminated).