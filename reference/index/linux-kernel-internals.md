---
title: Linux Kernel Internals — Curated Reading List
kind: index
domain: kernel
fetched_at: 2026-05-18
tags: [index, kernel, memory, scheduler, syscall]
---

> [!info] Reading-list index · canonical engineering blogs
> Articles on Linux kernel internals — memory, syscalls, scheduler, boot
> process — from sources the AIOps assistant trusts: Gustavo Duarte
> (manybutfinite.com), packagecloud, Ardan Labs, Cloudflare.

# Memory & address space

- [Anatomy of a Program in Memory — Gustavo Duarte](https://manybutfinite.com/post/anatomy-of-a-program-in-memory/)
- [How The Kernel Manages Your Memory — Gustavo Duarte](https://manybutfinite.com/post/how-the-kernel-manages-your-memory/)
- [Page Cache, the Affair Between Memory and Files — Gustavo Duarte](https://manybutfinite.com/post/page-cache-the-affair-between-memory-and-files/)
- [Getting Physical With Memory — Gustavo Duarte](https://manybutfinite.com/post/getting-physical-with-memory/)
- [Memory Translation and Segmentation — Gustavo Duarte](https://manybutfinite.com/post/memory-translation-and-segmentation/)
- [Cache: a place for concealment and safekeeping — Gustavo Duarte](https://manybutfinite.com/post/intel-cpu-caches/)
- [Motherboard Chipsets and the Memory Map — Gustavo Duarte](https://manybutfinite.com/post/motherboard-chipsets-memory-map/)
- [Getting Friendly With CPU Caches — Ardan Labs](https://ardanlabs.com/blog/2023/07/getting-friendly-with-cpu-caches.html)

# Syscalls, scheduler & boot

- [System Calls Make the World Go Round — Gustavo Duarte](https://manybutfinite.com/post/system-calls/)
- [The Definitive Guide to Linux System Calls — packagecloud](https://blog.packagecloud.io/the-definitive-guide-to-linux-system-calls/)
- [What does an idle CPU do? — Gustavo Duarte](https://manybutfinite.com/post/what-does-an-idle-cpu-do/)
- [When Does Your OS Run? — Gustavo Duarte](https://manybutfinite.com/post/when-does-your-os-run/)
- [What Your Computer Does While You Wait — Gustavo Duarte](https://manybutfinite.com/post/what-your-computer-does-while-you-wait/)
- [The Kernel Boot Process — Gustavo Duarte](https://manybutfinite.com/post/kernel-boot-process/)
- [How Computers Boot Up — Gustavo Duarte](https://manybutfinite.com/post/how-computers-boot-up/)
- [CPU Rings, Privilege, and Protection — Gustavo Duarte](https://manybutfinite.com/post/cpu-rings-privilege-and-protection/)

# Stack, security & kernel debugging

- [Journey to the Stack, Part I — Gustavo Duarte](https://manybutfinite.com/post/journey-to-the-stack/)
- [Epilogues, Canaries, and Buffer Overflows — Gustavo Duarte](https://manybutfinite.com/post/epilogues-canaries-buffer-overflows/)
- [Closures, Objects, and the Fauna of the Heap — Gustavo Duarte](https://manybutfinite.com/post/closures-objects-heap/)
- [Searching for the cause of hung tasks in the Linux kernel — Cloudflare](https://blog.cloudflare.com/searching-for-the-cause-of-hung-tasks-in-the-linux-kernel/)
- [Linux kernel security tunables everyone should consider adopting — Cloudflare](https://blog.cloudflare.com/linux-kernel-hardening/)
- [The Linux Kernel Key Retention Service — Cloudflare](https://blog.cloudflare.com/the-linux-kernel-key-retention-service-and-why-you-should-use-it-in-your-next-application/)
- [The Linux Crypto API for user applications — Cloudflare](https://blog.cloudflare.com/the-linux-crypto-api-for-user-applications/)
- [A gentle introduction to namespaces in Linux — packagecloud](https://blog.packagecloud.io/a-gentle-introduction-to-namespaces-in-linux/)

# Go runtime × Linux

- [Kubernetes Memory Limits and Go — Ardan Labs](https://ardanlabs.com/blog/2024/02/kubernetes-memory-limits-go.html)
- [Kubernetes CPU Limits and Go — Ardan Labs](https://ardanlabs.com/blog/2024/02/kubernetes-cpu-limits-go.html)
- [Garbage Collection In Go: Part II — GC Traces (Ardan Labs)](https://ardanlabs.com/blog/2019/05/garbage-collection-in-go-part2-gctraces.html)
- [Garbage Collection In Go: Part III — GC Pacing (Ardan Labs)](https://ardanlabs.com/blog/2019/07/garbage-collection-in-go-part3-gcpacing.html)
- [Range-Over Functions in Go — Ardan Labs](https://ardanlabs.com/blog/2024/04/range-over-functions-in-go.html)
