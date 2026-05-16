---
title: ChapterĀ3ĀĀPage Table Management
source: https://www.kernel.org/doc/gorman/html/understand/understand006.html
kind: external
domain: kernel
fetched_at: 2026-05-16
bookmark_title: Page Table Management
tags: [external, kernel]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.kernel.org](https://www.kernel.org/doc/gorman/html/understand/understand006.html)
> 抓取日期：2026-05-16

# ChapterĀ3ĀĀPage Table Management

Linux layers the machine independent/dependent layer in an unusual manner
in comparison to other operating systemsĀ[CP99]. Other operating
systems have objects which manage the underlying physical pages such as the
`pmap` object in BSD. Linux instead maintains the concept of a
three-level page table in the architecture independent code even if the
underlying architecture does not support it. While this is conceptually
easy to understand, it also means that the distinction between different
types of pages is very blurry and page types are identified by their flags
or what lists they exist on rather than the objects they belong to.

Architectures that manage their *Memory Management Unit
(MMU)* differently are expected to emulate the three-level
page tables. For example, on the x86 without PAE enabled, only two
page table levels are available. The *Page Middle Directory
(PMD)* is defined to be of size 1 and “folds back” directly onto
the *Page Global Directory (PGD)* which is optimised
out at compile time. Unfortunately, for architectures that do not manage
their cache or *Translation Lookaside Buffer (TLB)*
automatically, hooks for machine dependent have to be explicitly left in
the code for when the TLB and CPU caches need to be altered and flushed even
if they are null operations on some architectures like the x86. These hooks
are discussed further in Section 3.8.

This chapter will begin by describing how the page table is arranged and
what types are used to describe the three separate levels of the page table
followed by how a virtual address is broken up into its component parts
for navigating the table. Once covered, it will be discussed how the lowest
level entry, the *Page Table Entry (PTE)* and what bits
are used by the hardware. After that, the macros used for navigating a page
table, setting and checking attributes will be discussed before talking about
how the page table is populated and how pages are allocated and freed for
the use with page tables. The initialisation stage is then discussed which
shows how the page tables are initialised during boot strapping. Finally,
we will cover how the TLB and CPU caches are utilised.

Each process a pointer (`mm_struct`→`pgd`) to its own
*Page Global Directory (PGD)* which is a physical page frame. This
frame contains an array of type `pgd_t` which is an architecture
specific type defined in <`asm/page.h`>. The page tables are loaded
differently depending on the architecture. On the x86, the process page table
is loaded by copying `mm_struct`→`pgd` into the `cr3`
register which has the side effect of flushing the TLB. In fact this is how
the function `__flush_tlb()` is implemented in the architecture
dependent code.

Each active entry in the PGD table points to a page frame containing an array
of *Page Middle Directory (PMD)* entries of type `pmd_t`
which in turn points to page frames containing *Page Table Entries
(PTE)* of type `pte_t`, which finally points to page frames
containing the actual user data. In the event the page has been swapped
out to backing storage, the swap entry is stored in the PTE and used by
`do_swap_page()` during page fault to find the swap entry
containing the page data. The page table layout is illustrated in Figure
3.1.

Any given linear address may be broken up into parts to yield offsets within
these three page table levels and an offset within the actual page. To help
break up the linear address into its component parts, a number of macros are
provided in triplets for each page table level, namely a `SHIFT`,
a `SIZE` and a `MASK` macro. The `SHIFT`
macros specifies the length in bits that are mapped by each level of the
page tables as illustrated in Figure 3.2.

The `MASK` values can be ANDd with a linear address to mask out
all the upper bits and is frequently used to determine if a linear address
is aligned to a given level within the page table. The `SIZE`
macros reveal how many bytes are addressed by each entry at each level.
The relationship between the `SIZE` and `MASK` macros
is illustrated in Figure 3.3.

For the calculation of each of the triplets, only `SHIFT` is
important as the other two are calculated based on it. For example, the
three macros for page level on the x86 are:

5 #define PAGE_SHIFT 12 6 #define PAGE_SIZE (1UL << PAGE_SHIFT) 7 #define PAGE_MASK (~(PAGE_SIZE-1))

`PAGE_SHIFT` is the length in bits of the offset part of
the linear address space which is 12 bits on the x86. The size of a page is
easily calculated as 2 PAGE_SHIFT which is the equivalent of
the code above. Finally the mask is calculated as the negation of the bits
which make up the

`PMD_SHIFT` is the number of bits in the linear address which
are mapped by the second level part of the table. The `PMD_SIZE`
and `PMD_MASK` are calculated in a similar way to the page
level macros.

`PGDIR_SHIFT` is the number of bits which are mapped by
the top, or first level, of the page table. The `PGDIR_SIZE`
and `PGDIR_MASK` are calculated in the same manner as above.

The last three macros of importance are the `PTRS_PER_x`
which determine the number of entries in each level of the page
table. `PTRS_PER_PGD` is the number of pointers in the PGD,
1024 on an x86 without PAE. `PTRS_PER_PMD` is for the PMD,
1 on the x86 without PAE and `PTRS_PER_PTE` is for the lowest
level, 1024 on the x86.

As mentioned, each entry is described by the structs `pte_t`,
`pmd_t` and `pgd_t` for PTEs, PMDs and PGDs
respectively. Even though these are often just unsigned integers, they
are defined as structs for two reasons. The first is for type protection
so that they will not be used inappropriately. The second is for features
like PAE on the x86 where an additional 4 bits is used for addressing more
than 4GiB of memory. To store the protection bits, `pgprot_t`
is defined which holds the relevant flags and is usually stored in the lower
bits of a page table entry.

For type casting, 4 macros are provided in `asm/page.h`, which
takes the above types and returns the relevant part of the structs. They
are `pte_val()`, `pmd_val()`, `pgd_val()`
and `pgprot_val()`. To reverse the type casting, 4 more macros are
provided `__pte()`, `__pmd()`, `__pgd()`
and `__pgprot()`.

Where exactly the protection bits are stored is architecture dependent.
For illustration purposes, we will examine the case of an x86 architecture
without PAE enabled but the same principles apply across architectures. On an
x86 with no PAE, the `pte_t` is simply a 32 bit integer within a
struct. Each `pte_t` points to an address of a page frame and all
the addresses pointed to are guaranteed to be page aligned. Therefore, there
are `PAGE_SHIFT` (12) bits in that 32 bit value that are free for
status bits of the page table entry. A number of the protection and status
bits are listed in Table ??
but what bits exist and what they mean varies between architectures.

These bits are self-explanatory except for the `_PAGE_PROTNONE`
which we will discuss further. On the x86 with Pentium III and higher,
this bit is called the *Page Attribute Table (PAT)* while earlier
architectures such as the Pentium II had this bit reserved. The PAT bit
is used to indicate the size of the page the PTE is referencing. In a PGD
entry, this same bit is instead called the *Page Size Exception
(PSE)* bit so obviously these bits are meant to be used in conjunction.

As Linux does not use the PSE bit for user pages, the PAT bit is free in the
PTE for other purposes. There is a requirement for having a page resident
in memory but inaccessible to the userspace process such as when a region
is protected with `mprotect()` with the `PROT_NONE`
flag. When the region is to be protected, the `_PAGE_PRESENT`
bit is cleared and the `_PAGE_PROTNONE` bit is set. The
macro `pte_present()` checks if either of these bits are set
and so the kernel itself knows the PTE is present, just inaccessible to
*userspace* which is a subtle, but important point. As the hardware
bit `_PAGE_PRESENT` is clear, a page fault will occur if the
page is accessed so Linux can enforce the protection while still knowing
the page is resident if it needs to swap it out or the process exits.

Macros are defined in <`asm/pgtable.h`> which are important for
the navigation and examination of page table entries. To navigate the page
directories, three macros are provided which break up a linear address space
into its component parts. `pgd_offset()` takes an address and the
`mm_struct` for the process and returns the PGD entry that covers
the requested address. `pmd_offset()` takes a PGD entry and an
address and returns the relevant PMD. `pte_offset()` takes a PMD
and returns the relevant PTE. The remainder of the linear address provided
is the offset within the page. The relationship between these fields is
illustrated in Figure 3.1.

The second round of macros determine if the page table entries are present or may be used.

-
`pte_none()`,`pmd_none()`and`pgd_none()`return 1 if the corresponding entry does not exist; -
`pte_present()`,`pmd_present()`and`pgd_present()`return 1 if the corresponding page table entries have the`PRESENT`bit set; -
`pte_clear()`,`pmd_clear()`and`pgd_clear()`will clear the corresponding page table entry; -
`pmd_bad()`and`pgd_bad()`are used to check entries when passed as input parameters to functions that may change the value of the entries. Whether it returns 1 varies between the few architectures that define these macros but for those that actually define it, making sure the page entry is marked as present and accessed are the two most important checks.

There are many parts of the VM which are littered with page table walk code and
it is important to recognise it. A very simple example of a page table walk is
the function `follow_page()` in `mm/memory.c`. The following
is an excerpt from that function, the parts unrelated to the page table walk
are omitted:

407 pgd_t *pgd; 408 pmd_t *pmd; 409 pte_t *ptep, pte; 410 411 pgd = pgd_offset(mm, address); 412 if (pgd_none(*pgd) || pgd_bad(*pgd)) 413 goto out; 414 415 pmd = pmd_offset(pgd, address); 416 if (pmd_none(*pmd) || pmd_bad(*pmd)) 417 goto out; 418 419 ptep = pte_offset(pmd, address); 420 if (!ptep) 421 goto out; 422 423 pte = *ptep;

It simply uses the three offset macros to navigate the page tables and the
`_none()` and `_bad()` macros to make sure it is looking at
a valid page table.

The third set of macros examine and set the permissions of an entry. The permissions determine what a userspace process can and cannot do with a particular page. For example, the kernel page table entries are never readable by a userspace process.

- The read permissions for an entry are tested with
`pte_read()`, set with`pte_mkread()`and cleared with`pte_rdprotect()`; - The write permissions are tested with
`pte_write()`, set with`pte_mkwrite()`and cleared with`pte_wrprotect()`; - The execute permissions are tested with
`pte_exec()`, set with`pte_mkexec()`and cleared with`pte_exprotect()`. It is worth nothing that with the x86 architecture, there is no means of setting execute permissions on pages so these three macros act the same way as the read macros; - The permissions can be modified to a new value with
`pte_modify()`but its use is almost non-existent. It is only used in the function`change_pte_range()`in`mm/mprotect.c`.

The fourth set of macros examine and set the state of an entry. There
are only two bits that are important in Linux, the dirty bit and the
accessed bit. To check these bits, the macros `pte_dirty()`
and `pte_young()` macros are used. To set the bits, the macros
`pte_mkdirty()` and `pte_mkyoung()` are used. To
clear them, the macros `pte_mkclean()` and `pte_old()`
are available.

This set of functions and macros deal with the mapping of addresses and pages to PTEs and the setting of the individual entries.

The macro `mk_pte()` takes a `struct page` and protection
bits and combines them together to form the `pte_t` that needs to
be inserted into the page table. A similar macro `mk_pte_phys()`
exists which takes a physical page address as a parameter.

The macro `pte_page()` returns the `struct page`
which corresponds to the PTE entry. `pmd_page()` returns the
`struct page` containing the set of PTEs.

The macro `set_pte()` takes a `pte_t` such as that
returned by `mk_pte()` and places it within the processes page
tables. `pte_clear()` is the reverse operation. An additional
function is provided called `ptep_get_and_clear()` which clears an
entry from the process page table and returns the `pte_t`. This
is important when some modification needs to be made to either the PTE
protection or the `struct page` itself.

The last set of functions deal with the allocation and freeing of page tables. Page tables, as stated, are physical pages containing an array of entries and the allocation and freeing of physical pages is a relatively expensive operation, both in terms of time and the fact that interrupts are disabled during page allocation. The allocation and deletion of page tables, at any of the three levels, is a very frequent operation so it is important the operation is as quick as possible.

Hence the pages used for the page tables are cached in a number of different
lists called *quicklists*. Each architecture implements these
caches differently but the principles used are the same. For example, not
all architectures cache PGDs because the allocation and freeing of them
only happens during process creation and exit. As both of these are very
expensive operations, the allocation of another page is negligible.

PGDs, PMDs and PTEs have two sets of functions each for
the allocation and freeing of page tables. The allocation functions are
`pgd_alloc()`, `pmd_alloc()` and `pte_alloc()`
respectively and the free functions are, predictably enough, called
`pgd_free()`, `pmd_free()` and `pte_free()`.

Broadly speaking, the three implement caching with the use of three
caches called `pgd_quicklist`, `pmd_quicklist`
and `pte_quicklist`. Architectures implement these three
lists in different ways but one method is through the use of a LIFO type
structure. Ordinarily, a page table entry contains points to other pages
containing page tables or data. While cached, the first element of the list
is used to point to the next free page table. During allocation, one page
is popped off the list and during free, one is placed as the new head of
the list. A count is kept of how many pages are used in the cache.

The quick allocation function from the `pgd_quicklist`
is not externally defined outside of the architecture although
`get_pgd_fast()` is a common choice for the function name. The
cached allocation function for PMDs and PTEs are publicly defined as
`pmd_alloc_one_fast()` and `pte_alloc_one_fast()`.

If a page is not available from the cache, a page will be allocated using the
physical page allocator (see Chapter 6). The functions for the three levels of page tables are `get_pgd_slow()`,
`pmd_alloc_one()` and `pte_alloc_one()`.

Obviously a large number of pages may exist on these caches and so there
is a mechanism in place for pruning them. Each time the caches grow or
shrink, a counter is incremented or decremented and it has a high and low
watermark. `check_pgt_cache()` is called in two places to check
these watermarks. When the high watermark is reached, entries from the cache
will be freed until the cache size returns to the low watermark. The function
is called after `clear_page_tables()` when a large number of page
tables are potentially reached and is also called by the system idle task.

When the system first starts, paging is not enabled as page tables do not magically initialise themselves. Each architecture implements this differently so only the x86 case will be discussed. The page table initialisation is divided into two phases. The bootstrap phase sets up page tables for just 8MiB so the paging unit can be enabled. The second phase initialises the rest of the page tables. We discuss both of these phases below.

The assembler function `startup_32()` is responsible for
enabling the paging unit in `arch/i386/kernel/head.S`. While
all normal kernel code in `vmlinuz` is compiled with the base
address at `PAGE_OFFSET + 1MiB`, the kernel is actually loaded
beginning at the first megabyte (0x00100000) of memory. The first megabyte
is used by some devices for communication with the BIOS and is skipped. The
bootstrap code in this file treats 1MiB as its base address by subtracting
`__PAGE_OFFSET` from any address until the paging unit is
enabled so before the paging unit is enabled, a page table mapping has to
be established which translates the 8MiB of physical memory to the virtual
address `PAGE_OFFSET`.

Initialisation begins with statically defining at compile time an
array called `swapper_pg_dir` which is placed using linker
directives at 0x00101000. It then establishes page table entries for 2
pages, `pg0` and `pg1`. If the processor supports the
*Page Size Extension (PSE)* bit, it will be set so that pages
will be translated are 4MiB pages, not 4KiB as is the normal case. The first
pointers to `pg0` and `pg1` are placed to cover the region
`1-9MiB` the second pointers to `pg0` and `pg1`
are placed at `PAGE_OFFSET+1MiB`. This means that when paging is
enabled, they will map to the correct pages using either physical or virtual
addressing for just the kernel image. The rest of the kernel page tables
will be initialised by `paging_init()`.

Once this mapping has been established, the paging unit is turned on by setting
a bit in the `cr0` register and a jump takes places immediately to
ensure the *Instruction Pointer (EIP register)* is correct.

The function responsible for finalising the page tables is called
`paging_init()`. The call graph for this function on the x86
can be seen on Figure 3.4.

The function first calls `pagetable_init()` to initialise the
page tables necessary to reference all physical memory in `ZONE_DMA`
and `ZONE_NORMAL`. Remember that high memory in `ZONE_HIGHMEM`
cannot be directly referenced and mappings are set up for it temporarily.
For each `pgd_t` used by the kernel, the boot memory allocator
(see Chapter 5) is called to allocate a page
for the PMDs and the PSE bit will be set if available to use 4MiB TLB entries
instead of 4KiB. If the PSE bit is not supported, a page for PTEs will be
allocated for each `pmd_t`. If the CPU supports the PGE flag,
it also will be set so that the page table entry will be global and visible
to all processes.

Next, `pagetable_init()` calls `fixrange_init()` to
setup the fixed address space mappings at the end of the virtual address
space starting at `FIXADDR_START`. These mappings are used
for purposes such as the local APIC and the atomic kmappings between
`FIX_KMAP_BEGIN` and `FIX_KMAP_END`
required by `kmap_atomic()`. Finally, the function calls
`fixrange_init()` to initialise the page table entries required for
normal high memory mappings with `kmap()`.

Once `pagetable_init()` returns, the page tables for kernel space
are now full initialised so the static PGD (`swapper_pg_dir`)
is loaded into the CR3 register so that the static table is now being used
by the paging unit.

The next task of the `paging_init()` is responsible for
calling `kmap_init()` to initialise each of the PTEs with the
`PAGE_KERNEL` protection flags. The final task is to call
`zone_sizes_init()` which initialises all the zone structures used.

There is a requirement for Linux to have a fast method of mapping virtual
addresses to physical addresses and for mapping `struct page`s to
their physical address. Linux achieves this by knowing where, in both virtual
and physical memory, the global `mem_map` array is as the global array
has pointers to all `struct page`s representing physical memory
in the system. All architectures achieve this with very similar mechanisms
but for illustration purposes, we will only examine the x86 carefully. This
section will first discuss how physical addresses are mapped to kernel
virtual addresses and then what this means to the `mem_map` array.

As we saw in Section 3.6, Linux sets up a
direct mapping from the physical address 0 to the virtual address
`PAGE_OFFSET` at 3GiB on the x86. This means that any
virtual address can be translated to the physical address by simply
subtracting `PAGE_OFFSET` which is essentially what the function
`virt_to_phys()` with the macro `__pa()` does:

/* from <asm-i386/page.h> */ 132 #define __pa(x) ((unsigned long)(x)-PAGE_OFFSET) /* from <asm-i386/io.h> */ 76 static inline unsigned long virt_to_phys(volatile void * address) 77 { 78 return __pa(address); 79 }

Obviously the reverse operation involves simply adding `PAGE_OFFSET`
which is carried out by the function `phys_to_virt()` with
the macro `__va()`. Next we see how this helps the mapping of
`struct page`s to physical addresses.

As we saw in Section 3.6.1, the kernel image is located at
the physical address 1MiB, which of course translates to the virtual address
`PAGE_OFFSET + 0x00100000` and a virtual region totaling about 8MiB
is reserved for the image which is the region that can be addressed by two
PGDs. This would imply that the first available memory to use is located
at `0xC0800000` but that is not the case. Linux tries to reserve
the first 16MiB of memory for `ZONE_DMA` so first virtual area used for
kernel allocations is actually `0xC1000000`. This is where the global
`mem_map` is usually located. `ZONE_DMA` will be still get used,
but only when absolutely necessary.

Physical addresses are translated to `struct page`s by treating
them as an index into the `mem_map` array. Shifting a physical address
`PAGE_SHIFT` bits to the right will treat it as a PFN from physical
address 0 which is *also* an index within the `mem_map` array.
This is exactly what the macro `virt_to_page()` does which is
declared as follows in <`asm-i386/page.h`>:

#define virt_to_page(kaddr) (mem_map + (__pa(kaddr) >> PAGE_SHIFT))

The macro `virt_to_page()` takes the virtual address `kaddr`,
converts it to the physical address with `__pa()`, converts it into
an array index by bit shifting it right `PAGE_SHIFT` bits and
indexing into the `mem_map` by simply adding them together. No macro
is available for converting `struct page`s to physical addresses
but at this stage, it should be obvious to see how it could be calculated.

Initially, when the processor needs to map a virtual address to a physical
address, it must traverse the full page directory searching for the PTE
of interest. This would normally imply that each assembly instruction that
references memory actually requires several separate memory references for the
page table traversalĀ[Tan01]. To avoid this considerable overhead,
architectures take advantage of the fact that most processes exhibit a locality
of reference or, in other words, large numbers of memory references tend to be
for a small number of pages. They take advantage of this reference locality by
providing a *Translation Lookaside Buffer (TLB)* which is a small
associative memory that caches virtual to physical page table resolutions.

Linux assumes that the most architectures support some type of TLB although the architecture independent code does not cares how it works. Instead, architecture dependant hooks are dispersed throughout the VM code at points where it is known that some hardware with a TLB would need to perform a TLB related operation. For example, when the page tables have been updated, such as after a page fault has completed, the processor may need to be update the TLB for that virtual address mapping.

Not all architectures require these type of operations but because some do, the hooks have to exist. If the architecture does not require the operation to be performed, the function for that TLB operation will a null operation that is optimised out at compile time.

A quite large list of TLB API hooks, most of which are declared in
<`asm/pgtable.h`>, are listed in Tables 3.2
and ??
and the APIs are quite well documented in the kernel
source by `Documentation/cachetlb.txt`Ā[Mil00]. It is
possible to have just one TLB flush function but as both TLB flushes and
TLB refills are *very* expensive operations, unnecessary TLB flushes
should be avoided if at all possible. For example, when context switching,
Linux will avoid loading new page tables using *Lazy TLB Flushing*,
discussed further in Section 4.3.


ĀThis flushes the entire TLB on all processors running in the system making it the most expensive TLB flush operation. After it completes, all modifications to the page tables will be visible globally. This is required after the kernel page tables, which are global in nature, have been modified such as after vfree()(See Chapter 7) completes or after the PKMap is flushed (See Chapter 9).ĀThis flushes all TLB entries related to the userspace portion (i.e. below PAGE_OFFSET) for the requested mm context. In some architectures, such as MIPS, this will 

[... 内容超长，已截断；完整原文见 source URL ...]
