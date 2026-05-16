---
title: ThreadSanitizerAlgorithm
source: https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm
kind: external
domain: compute
author: Google
original_date: 2015-08-31
fetched_at: 2026-05-16
bookmark_title: ThreadSanitizerAlgorithm · google/sanitizers Wiki
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[github.com](https://github.com/google/sanitizers/wiki/ThreadSanitizerAlgorithm)
> 作者：Google
> 原始日期：2015-08-31
> 抓取日期：2026-05-16

# ThreadSanitizerAlgorithm

-
Notifications
You must be signed in to change notification settings - Fork 1.1k

# ThreadSanitizerAlgorithm

Here we provide a high-level description of `ThreadSanitizer`

(version 2) algorithm.
The tool is work-in-progress, so some details may change in future.
This is the second version of `ThreadSanitizer`

,
the first (slow and deprecated) version can be found here.

`ThreadSanitizer`

consists of two parts:
instrumentation module and a run-time library.

We instrument every memory access in the program unless it can be proven
to be race-free or redundant.
Memory accesses are simply prepended with a function call like `__tsan_read4(addr)`

.

Examples of race-free access:

- Reads from constant globals (including vtables).
- Accesses to memory that does not escape the current function (and therefore the current thread).

Examples of redundant accesses:

- Read that happens before a write to the same location.

Atomic memory accesses are instrumented using specialized `__tsan_atomic_`

callbacks.
Reads from vtable pointer are instrumented using `__tsan_vptr_update`

to
deal with benign vptr races.
Function entry and exit are instrumented with `__tsan_func_entry(caller_pc)`

and `__tsan_func_exit`

.
A call to `__tsan_init`

is inserted before all initializers.

**Shadow State** is **N** Shadow Words (described below); **N** is one of 2, 4, 8 (configurable).
Every aligned 8-byte word of application memory is mapped into **N** Shadow Words
using direct address mapping (no memory accesses required to compute the shadow address).

**Shadow Word** is a 64-bit object that contains the following fields:

| TID (Thread Id) | 16 bits (configurable) |
|---|---|
| Scalar Clock | 42 bits (configurable) |
| IsWrite | 1 bit |
| Access Size (1, 2, 4 or 8) | 2 bits |
| Address Offset (0..7) | 3 bits |

One Shadow Word represents a single memory access to a subset of bytes within
the 8-byte word of application memory.
Therefore the Shadow State describes **N** different accesses
to the corresponding application memory region.

The core of the algorithm is the **State Machine**,
a function that updates the Shadow State on every memory access.

First, the thread's clock is incremented and a new Shadow Word that corresponds to the current memory access is created. Then the State Machine iterates over all Shadow Words stored in the Shadow State. If one of the old Shadow Words constitutes a race with the new Shadow Word, a warning message is printed. The new Shadow Word is inserted in place of an empty Shadow Word or in place of a Shadow Word that happened-before the new one. If no place for insertion is found, a random Shadow Word is evicted.

All accesses to Shadow Words are 64-bit atomic loads/stores, but otherwise no locking is involved. Even if two threads are modifying the same Shadow State at the same time, the Shadow State will remain consistent. There is tiny probability to miss a data race though.

Approximate pseudo code follows
(for an exact algorithm see the code in `tsan_rtl.cc`

;
we will update the pseudo code once
the actual code is frozen).

```
def HandleMemoryAccess(addr, tid, is_write, size, pc):
shadow_address = MapApplicationToShadow(addr)
IncrementThreadClock(tid)
LogEvent(tid, pc);
new_shadow_word = {tid, CurrentClock(tid), is_write, size, addr & 7}
store_word = new_shadow_word
for i in 1..N:
UpdateOneShadowState(shadow_address, i, new_shadow_word, store_word)
if store_word:
# Evict a random Shadow Word
shadow_address[Random(N)] = store_word # Atomic
```


```
def UpdateOneShadowState(shadow_address, i, new_shadow_word, store_word):
idx = (i + new_shadow_word.offset) % N
old_shadow_word = shadow_address[idx] # Atomic
if old_shadow_word == 0: # The old state is empty
if store_word:
StoreIfNotYetStored(shadow_address[idx], store_word)
return
if AccessedSameRegion(old_shadow_word, new_shadow_word):
if SameThreads(old_shadow_word, new_shadow_word):
TODO
else: # Different threads
if not HappensBefore(old_shadow_word, new_shadow_word):
ReportRace(old_shadow_word, new_shadow_word)
elif AccessedIntersectingRegions(old_shadow_word, new_shadow_word):
if not SameThreads(old_shadow_word, new_shadow_word)
if not HappensBefore(old_shadow_word, new_shadow_word)
ReportRace(old_shadow_word, new_shadow_word)
else: # regions did not intersect
pass # do nothing
def StoreIfNotYetStored(shadow_address, store_word):
*shadow_address = store_word # Atomic
store_word = 0
```


`ThreadSanitizer`

needs to report stack traces and other information for **previous** accesses
involved in a data race.
`LogEvent(tid, pc)`

(see above) stores the access event in a large thread-local
circular buffer for later recovery. **TODO**: add more detail.

Finding races and memory errors with compiler instrumentation