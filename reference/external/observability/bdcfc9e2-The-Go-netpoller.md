---
title: The Go netpoller
source: https://morsmachine.dk/netpoller
kind: external
domain: observability
original_date: 2013-09-08
fetched_at: 2026-05-16
bookmark_title: The Go netpoller - Morsing's blog
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[morsmachine.dk](https://morsmachine.dk/netpoller)
> 原始日期：2013-09-08
> 抓取日期：2026-05-16

# The Go netpoller

8 September 2013

# The Go netpoller

# Introduction

I'm bored again or I have something more important to do, so it's time for another blog post about the Go runtime. This time I'm gonna take a look at how Go handles network I/O.

# Blocking

In Go, all I/O is blocking. The Go ecosystem is built around the idea that you write against a blocking interface and then handle concurrency through goroutines and channels rather than callbacks and futures. An example is the HTTP server in the "net/http" package. Whenever it accepts a connection, it will create a new goroutine to handle all the requests that will happen on that connection. This construct means that the request handler can be written in a very straightforward manner. First do this, then do that. Unfortunately, using the blocking I/O provided by the operating system isn't suitable for constructing our own blocking I/O interface.

In my previous post about the Go runtime, I covered how the Go scheduler handles syscalls. To handle a blocking syscall, we need a thread that can be blocked inside the operating system. If we were to build our blocking I/O on top of the OS' blocking I/O, we'd spawn a new thread for every client stuck in a syscall. This becomes really expensive once you have 10,000 client threads, all stuck in a syscall waiting for their I/O operation to succeed.

Go gets around this problem by using the asynchronous interfaces that the OS provides, but blocking the goroutines that are performing I/O.

# The netpoller

The part that converts asynchronous I/O into blocking I/O is called the netpoller. It sits in its own thread, receiving events from goroutines wishing to do network I/O. The netpoller uses whichever interface the OS provides to do polling of network sockets. On Linux, it uses epoll, on the BSDs and Darwin, it uses kqueue and on Windows it uses IoCompletionPort. These interfaces all have in common that they provide user space a way to efficiently poll for the status of network I/O.

Whenever you open or accept a connection in Go, the file descriptor that backs it is set to non-blocking mode. This means that if you try to do I/O on it and the file descriptor isn't ready, it will return an error code saying so. Whenever a goroutine tries to read or write to a connection, the networking code will do the operation until it receives such an error, then call into the netpoller, telling it to notify the goroutine when it is ready to perform I/O again. The goroutine is then scheduled out of the thread it's running on and another goroutine is run in its place.

When the netpoller receives notification from the OS that it can perform I/O on a file descriptor, it will look through its internal data structure, see if there are any goroutines that are blocked on that file and notify them if there are any. The goroutine can then retry the I/O operation that caused it to block and succeed in doing so.

If this is sounding a lot like using the old select and poll Unix system calls to do I/O, it's because it is. But instead of looking up a function pointer and a struct containing a bunch of state variables, the netpoller looks up a goroutine that can be scheduled in. This frees you from managing all that state, rechecking whether you received enough data on the last go around and juggling function pointers like you would do with traditional Unix networking I/O.