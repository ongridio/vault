---
title: Poor man’s memory leak detection with strace
source: https://deathandthepenguinblog.wordpress.com/2015/08/15/poor-mans-memory-leak-detection-with-strace/
kind: external
domain: compute
author: Damian Eads; PhD
original_date: 2015-08-15
fetched_at: 2026-05-16
bookmark_title: Poor man’s memory leak detection with strace | Death and the penguin
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[deathandthepenguinblog.wordpress.com](https://deathandthepenguinblog.wordpress.com/2015/08/15/poor-mans-memory-leak-detection-with-strace/)
> 作者：Damian Eads; PhD
> 原始日期：2015-08-15
> 抓取日期：2026-05-16

# Poor man’s memory leak detection with strace

When we suspect our program has a memory leak, there are many tools available to help you pinpoint offending parts of the program. **Valgrind** is popular, but slow and laborious. Sometimes I like to confirm my intuition with a quick and dirty trick, especially if it prints clues in real-time as the program runs. **strace**, a program that traces system calls, can be used for this purpose. **strace** logs system calls (API endpoints that transfer control from user space to kernel space), their input arguments, and return values. It even prints the beginning of each buffer passed via a pointer or sometimes a synopsis of a common struct (e.g. *struct stat*).

Let’s consider an example of a memory leak. In the program below, we assume each iteration of the loop needs a chunk of memory as scratch space. When the block concludes, this chunk is no longer needed and can be freed. We have purposely commented out the **free()** call to induce a memory leak.

#include <stdlib.h> #include <stdio.h> int main(int argc, char **argv) { int n = 100000; for (int i = 0; i < n; i++) { printf("step\n"); void *mem = (void*)malloc(100000); /* do some computation here... */ /* free(mem); */ } return 0; }

Compile the program:

gcc leak.c -o leak

If we run this through **strace**, we see a long pattern that looks like this:

brk(0x23812000) = 0x23812000 write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 brk(0x23842000) = 0x23842000 write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 brk(0x23873000) = 0x23873000 write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 brk(0x238a4000) = 0x238a4000 write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 brk(0x238d5000) = 0x238d5000

**printf()** is not a system call, but a user space function provided by the C standard library, which is why it does not appear in the trace. **printf()** actually calls the more general system call **write()** on file descriptor 1 (by convention, 0 is standard input, 1 is standard output, 2 is standard error). That’s why we see the string “step”, which was passed to **printf()**. In between the write calls, **brk()** are interspersed. The **brk()** is less familiar but as ubiquitous as **write()**. We can consult **man** to learn what it does.

$ man 2 brk | head -30 BRK(2) Linux Programmer's Manual BRK(2) NAME brk, sbrk - change data segment size SYNOPSIS #include <unistd.h> int brk(void *addr); void *sbrk(intptr_t increment); DESCRIPTION brk() and sbrk() change the location of the program break, which defines the end of the process's data segment (i.e., the program break is the first location after the end of the uninitialized data segment). Increasing the program break has the effect of allocating memory to the process; decreasing the break deallocates memory.

As we see, **brk()** is used by a process to request more memory allocated to it by the OS. Let’s filter out the **write()** calls. We can see that the ceiling of the process’s data segment only increases:

brk(0x23812000) = 0x23812000 brk(0x23842000) = 0x23842000 brk(0x23873000) = 0x23873000 brk(0x238a4000) = 0x238a4000 brk(0x238d5000) = 0x238d5000

We can make an educated guess that **malloc()** calls **brk()** to request more memory when needed. However, we also can infer that the chunks are not being reused. If the chunks allocated had been larger, e.g. 1 million bytes each, we would observe that malloc uses **mmap()** instead of **brk()**.

mmap(NULL, 1003520, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdd2c461000 write(1, "step\n", 5step ) = 5 mmap(NULL, 1003520, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdd2c36c000 write(1, "step\n", 5step ) = 5 mmap(NULL, 1003520, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdd2c277000 write(1, "step\n", 5step ) = 5 mmap(NULL, 1003520, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdd2c182000 write(1, "step\n", 5step ) = 5 mmap(NULL, 1003520, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fdd2c08d000 write(1, "step\n", 5step

Now, let’s free the memory as the last step in the body of the loop.

#include <stdlib.h> #include <stdio.h> int main(int argc, char **argv) { int n = 100000; for (int i = 0; i < n; i++) { printf("step\n"); void *mem = (void*)malloc(100000); /* do some computation here... */ free(mem); } return 0; }

Let’s recompile and run **strace** again to see if there is any change in output.

write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 write(1, "step\n", 5step ) = 5 ...

We see a long stream of write calls corresponding to **printf(“step\n”)** without **brk()** calls interleaved. Thus, our single-threaded program is not requesting more memory from the kernel as the loop continues beyond the first iteration. Thus, we can infer that **malloc()** has reused the memory that it had requested from the OS earlier in its execution. Of course, things are often much trickier for multi-threaded programs. As Ed Rosten points out, the compiler sometimes optimizes out the malloc, free pair altogether.

In summary, if **strace** gives us an uninterrupted stream of **brk()** calls, it may be a sign that a memory leak is present. If the cause and fix of the leak is not immediately obvious, you can use this as a cue to run your favorite memory leak tool on the program. We cannot use this trick to guarantee the absence of a leak, but it is extremely useful to get clues to see if reality matches our intuition.