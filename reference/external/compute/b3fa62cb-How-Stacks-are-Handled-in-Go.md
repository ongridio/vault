---
title: How Stacks are Handled in Go
source: https://blog.cloudflare.com/how-stacks-are-handled-in-go/
kind: external
domain: compute
author: Daniel Morsing
original_date: 2014-09-15
fetched_at: 2026-05-16
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.cloudflare.com](https://blog.cloudflare.com/how-stacks-are-handled-in-go/)
> 作者：Daniel Morsing
> 原始日期：2014-09-15
> 抓取日期：2026-05-16

# How Stacks are Handled in Go

At CloudFlare, We use Go for a variety of services and applications. In this blog post, We're going to take a deep dive into some of the technical intricacies of Go.

One of the more important features of Go is goroutines. They are cheap, cooperatively scheduled threads of execution that are used for a variety of operations, like timeouts, generators and racing multiple backends against each other. To make goroutines suitable for as many tasks as possible, we have to make sure that each goroutine takes up the least amount of memory, but also have people be able to start them up with a minimal amount of configuration.

To achieve this, Go manages stacks in way that behaves like any other language, but is quite different in how they're implemented.

### An introduction to thread stacks

Before we look at Go, let's look at how stacks are managed in a traditional language like C.

When you start up a thread in C, the standard library is responsible for allocating a block of memory to be used as that thread's stack. It will allocate this block, tell the kernel where it is and let the kernel handle the execution of the thread. The problem comes when this block of memory is not enough.

Consider the following function:

```
int a(int m, int n) {
if (m == 0) {
return n + 1;
} else if (m > 0 && n == 0) {
return a(m - 1, 1);
} else {
return a(m - 1, a(m, n - 1));
}
}
```


This function is highly recursive and invoking it as `a(4,5)`

will consume all of the stack memory. To get around this, you can adjust the size of blocks that the standard library hands out to thread stacks. However, increasing your stack size across the board means that every thread will take up that amount of memory, even if they're not highly recursive. You can run out of memory, even though your program isn't using the stack memory allocated.

The other option is to decide stack size on a per thread basis. Now you're tasked with having to configure how much stack memory each thread needs, which makes creating a thread more difficult than we want it to be. Figuring out how much memory a thread will take is in the general case undecidable and in the common case just very difficult.

### How Go handles this

Instead of giving each goroutine a fixed amount of stack memory, the Go runtime attempts to give goroutines the stack space they need on demand, freeing the programmer from having to make decisions about stack size. The Go team is currently working on switching from one method to another for doing this. I'll try to explain the old method, its shortcomings, the new method and why we're switching to it.

### Segmented stacks

Segmented stacks is the original way that Go handled stacks. When a goroutine is created, we allocate an 8 kilobyte section of memory to use for the stack and we let the goroutine run its course.

The interesting bit comes when we run out of stack space for these 8 kilobytes. To handle this, each Go function has a tiny prologue at function entry. It checks if we've used up the allocated stack space and calls the `morestack`

function if we have.

The `morestack`

function allocates a new section of memory for stack space. It then fills in various pieces of data about the stack into a struct at the bottom of the stack, including the address of the stack that we just came from. After we've got a new stack segment, we restart the goroutine by retrying the function that caused us to run out of stack. This is known as a stack split.

The stack looks like this just after we've split the stack.

```
+---------------+
| |
| unused |
| stack |
| space |
+---------------+
| Foobar |
| |
+---------------+
| |
| lessstack |
+---------------+
| Stack info |
| |-----+
+---------------+ |
|
|
+---------------+ |
| Foobar | |
| | <---+
+---------------+
| rest of stack |
| |
```


At the bottom of the new stack, we've inserted a stack entry for a function called `lessstack`

. We didn't actually call this function. It is set up for when we return from the function that caused us to run out of stack. When that happens, we return into the `lessstack`

, it looks up the struct that we put at the bottom of the stack and adjusts the stack pointer so that we're returning into the previous segment. After this, we can deallocate the stack segment we came from and go on our way.

### The problem with segmented stacks

Segmented stacks give us stacks that grow and shrink on demand. The programmer doesn't have to worry about sizing the stacks, starting a new goroutine is cheap and we're now handling the case where the programmer doesn't know how big the stack will grow.

This was how Go handled stack growing up until recently, but this approach had a flaw. Shrinking the stack is a relatively expensive operation. It was most felt when you had a stack split inside a loop. A function would grow, split the stack, return and deallocate the stack segment. If you're doing this in a loop, you end up paying a rather large penalty.

This was known as the hot split problem. It was the main reason that the Go developers switched over to a new method of managing stacks, called stack copying.

### Stack copying

Stack copying starts out a lot like segmented stacks. The goroutine is running along, using the stack space and when it runs out, it hits the same stack overflow check as in the old approach.

However, instead of having a link back to the previous segment, it creates the new segment with double the size and copies the old segment into it. This means that when the stack shrinks back to its old size, the runtime doesn't have to do anything. Shrinking is now a free operation. Additionally, when the stack grows again, the runtime doesn't have to do anything. We just reuse the space that we allocated earlier.

### How are stacks copied?

Copying the stack sounds easy but it is a bit of an undertaking. Since variables on the stack can have their address taken in Go, you end up in a situation where you have pointers into the stack. When you move the stack, any pointers to the stack are now going to be invalid.

Lucky for us, the only pointers that can point into the stack are pointers that live on the stack. This is necessary for memory safety since otherwise, it would be possible for a program to access parts of the stack no longer in use.

Because we need to know where the pointers are for garbage collection, we know which parts of the stack are pointers. When we move the stack, we can update the pointers within the stack to their new targets and all the relevant pointers are taken care of.

Since we use the garbage collection information to copy stacks, any function that can show up on a stack must have this information available. This was not always the case. Because large sections of the runtime were written in C, a lot of the calls into the runtime did not have pointer information and as such weren't copyable. When this happened, we fell back to stack segments, with their associated costs.

This is why the runtime developers are currently rewriting large pieces of the runtime in Go. The code that cannot reasonably be rewritten in Go, like the core of the scheduler and the garbage collector, will get executed on special stacks which are sized individually by the runtime developers.

Besides making stack copying possible, this will also allow us to do things like concurrent garbage collection in the future.

### An aside about virtual memory

A different way of handling stacks is to allocate large sections of virtual memory. Since physical memory is only allocated when the memory is touched, it looks like you can just allocate a large section and let the operating system handle it. There are a couple of problems with this approach.

Firstly, 32-bit systems only have 4 gigabytes of virtual memory, of which normally only 3 gigabytes are available for the application. Since it is not uncommon to have a million goroutines running at once, you will likely run out of virtual memory, even if we assume that stacks are only 8 kilobyte.

Secondly, while we can allocate large amounts of memory for 64-bit systems, it relies on overcommitting memory. Overcommit is when you allocate more memory than you physically have and rely on the operating system to make sure that physical memory is allocated when it is needed. However, enabling overcommit carries some risk. Since processes can allocate more memory than the machine has, it has to make up memory somehow if the processes start actually using more memory than is available. It can do this by putting sections of memory onto disk, but this adds latency that is unpredictable and often, systems are run with overcommit turned off for this reason.

### Conclusion

A lot of effort has gone into making goroutines cheap, fast and suitable for most tasks. Stack management is just a small part of that.If you want to know more about copying stacks, there is a design doc that goes more into details about this.

Additionally, if you want more details about the work of rewriting the runtime in Go, there's a mailing list post here.

We're always looking for Go programmers, so if you found this blog post interesting, why not check out our jobs page?