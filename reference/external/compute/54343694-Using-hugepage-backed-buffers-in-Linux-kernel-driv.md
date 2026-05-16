---
title: Using hugepage-backed buffers in Linux kernel driver
source: http://nuncaalaprimera.com/2014/using-hugepage-backed-buffers-in-linux-kernel-driver
kind: external
domain: compute
author: Guillermo Julián
original_date: 2014-08-04
fetched_at: 2026-05-16
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[nuncaalaprimera.com](http://nuncaalaprimera.com/2014/using-hugepage-backed-buffers-in-linux-kernel-driver)
> 作者：Guillermo Julián
> 原始日期：2014-08-04
> 抓取日期：2026-05-16

# Using hugepage-backed buffers in Linux kernel driver

# Using hugepage-backed buffers in Linux kernel driver

I'm not more than a complete newbie in Linux kernel development, but recently I had to modify a driver so it could use big buffers backed by hugepages. Documentation on hugepages is scarce, even more when you try to mix it in kernel driver, so I decided to write about the process and document what I found so others can have a starting point. This is probably not going to be completely correct, but it's better than nothing.

## Outline of the process

The process is simple: get the memory in user space and communicate it to the driver through a `ioctl`

call. Then, the driver has to get the real address of the buffer using `get_user_pages`

and `page_address`

.

Once the driver is done with the buffer, it needs to be released in the same way that it should be done with any other memory area brought from userspace: set page as dirty if it's not reserved, and then release the page with `page_cache_release`

. Let's now get to the details

## How are huge pages laid out in memory

First of all, some background with how hugepages are laid out in memory. The kernel allocates a pool of huge pages at boot time. I'm not sure if the pool is a continous memory area, but it seems to be.

A hugepage is just a large contigous memory region, always present in RAM and unable to be swapped out. That means that it's not necessary to map the page into the kernel address space when it's going to be used, because it's already there.

## Getting a hugepage backed buffer in userspace

This is not difficult. Of course, you should have hugepages enabled on your kernel.

From your userspace application, you have two options to get a hugepage backed buffer: using `mmap`

or `shmget`

.

I've personally used the `mmap`

, as it's easier to define which page size do you want. There's a code sample in the Linux kernel tree.

First of all, you have to mount a hugetlbfs virtual filesystem. This is done with the command

mount -t hugetlbfs {mount point} -o pagesize={page size}

The `pagesize`

parameter is optional. If it's not present, the system will use the default page size for hugepages. This article and the kernel docs detail further options for this command.

Your userspace program should open a new file on that mount point with the `open`

syscall. Say that the mount point is */mnt/hugetlb*, then you should call `open("/mnt/hugetlb/randomname", O_CREAT | O_RDWR, 0755)`

.

Once you have the file descriptor for that file, do the `mmap`

call on that fd with the size you want for your buffer:

buffer = mmap(0, buffer_size, PROT_READ | PROT_WRITE, MAP_SHARED, fr, 0);

If everything's gone ok, `buffer`

is the beginning of a memory region of `buffer_size`

bytes backed by huge pages. Note that the size will be rounded down to a multiple of huge page size. Now's the time to transfer the buffer to the kernel driver.

## Using the buffer in the kernel driver

Now that you have the buffer allocated, it's time to pass it on to the driver. I suppose there are different ways to do it. However, I found that an `ioctl`

call was pretty easy. Just create an structure with the address and the size to pass to the ioctl call.

Once in kernel space, you have to call to `get_user_pages`

in the usual fashion, as if this was not a hugepage-backed buffer. Obviously, the number of pages you have to get is big (size / PAGE_SIZE) because the kernel uses the same structure to represent both regular pages and huge-pages.

Now, calling `page_address`

on the first page returned by `get_user_pages`

would give you the address in kernel-space (hugepages are always mapped into the kernel address space) and you could work with that. However, there's a problem when you get a buffer backed by more than one hugepage: sequentality.

As I explained in this SO question, the kernel may give you non-sequential pages. The solution, as the user CL pointed out, is using `vm_map_ram`

to request a linear mapping of all of those pages.

This is a sample code for the task of mapping a hugepage backed (or, for that matter, any kind of buffer) buffer into the kernel

```
struct page** pages;
int retval;
unsigned long npages;
unsigned long buffer_start = (unsigned long) huge->addr; // Address from user-space map.
void* remapped;
npages = 1 + (bufsize - 1) / PAGE_SIZE;
pages = vmalloc(npages * sizeof(struct page *));
down_read(¤t->mm->mmap_sem);
retval = get_user_pages(current, current->mm, buffer_start, npages,
1 /* Write enable */, 0 /* Force */, pages, NULL);
up_read(¤t->mm->mmap_sem);
nid = page_to_nid(pages[0]); // Remap on the same NUMA node.
remapped = vm_map_ram(pages, npages, nid, PAGE_KERNEL);
// Do work on remapped.
```


## Mapping back into user space

Maybe you want to get access to that buffer from another userspace application, different from the one that reserved the buffer. Theoretically, you could do it in the same way you'd mmap any other kernel buffer. However, I found that, for some reason, the `mmap`

call fails with `-ENOMEM`

with buffers bigger than 1 GB, and I haven't been able to debug the cause (memory, address space and other resources are well below their limits).

So, my recommendation would be to store the hugetlb file name somewhere (your driver, a configuration file, whatever) and map that file into your application, instead of doing a mmap on the driver.

## Releasing the buffer

In order for the hugepages to be freed, you have to

- Unmap the file from any userspace app that has it mapped.
-
Release the pages from the driver

-
Delete the file in the hugetlbfs filesystem.


Easy enough, but remember the part of the driver release. If you don't free them from the driver, the pages will not be freed by the kernel. And having GBs of memory trapped in Linux memory limbo is not funny, believe me.

For reference, this is the code you should use to release the pages from the kernel driver


```
for(i = 0; i < buf->huge_pages_num; i++)
{
down_read(¤t->mm->mmap_sem);
if (!PageReserved(buf->huge_pages[i]))
{
SetPageDirty(buf->huge_pages[i]);
}
page_cache_release(buf->huge_pages[i]);
up_read(¤t->mm->mmap_sem);
}
```


Same as any other page release, except that you don't have to unmap the pages.

This was everything. As I said at the beginning, I don't guarantee this is 100% accurate, but it may be useful for total newbies as me.