---
title: Recycling memory buffers in Go
source: https://blog.cloudflare.com/recycling-memory-buffers-in-go/
kind: external
domain: compute
author: John Graham-Cumming
original_date: 2013-08-24
fetched_at: 2026-05-16
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.cloudflare.com](https://blog.cloudflare.com/recycling-memory-buffers-in-go/)
> 作者：John Graham-Cumming
> 原始日期：2013-08-24
> 抓取日期：2026-05-16

# Recycling memory buffers in Go

**THIS BLOG POST IS VERY OLD NOW. YOU PROBABLY DON'T WANT TO USE THE TECHNIQUE DESCRIBED HERE. GO'S ****sync.Pool**** IS A BETTER WAY TO GO.**

The other day I wrote about our use of Lua to implement our new Web Application Firewall. The other language that's become very popular at CloudFlare is Go. In the past, I've written about how we use Go to write network services including Railgun.

One of the operational challenges of using a garbage-collected language like Go for long-running network services is memory management.

To understand Go's memory management it's necessary to dig a little into the Go run-time code. There are two separate processes that take care of identifying memory that is no longer used by the program (that's garbage collection) and returning memory to the operating system when it doesn't look like it will be used again (that's called scavenging in the Go code).

Here's a small program that creates a lot of garbage. Once a second it asks for a byte array of between 5,000,000 and 10,000,000 bytes. It keeps a pool of 20 of these buffers around and discards others. This program is designed to simulate something that happens quite often in programs: memory is allocated by different parts of the program over time, some of it is retained, most of it is no longer needed. In a networked Go program this can easily happen with goroutines that handle network connections or requests. It's common to have goroutines that allocate blocks of memory (such as slices to store received data) and then no longer need them. Over time there will be some number of blocks of memory in use by network connections that are being handled and accumulated garbage from connections that have come and gone.

package main

import ("fmt""math/rand""runtime""time" )

func makeBuffer() []byte {return make([]byte, rand.Intn(5000000)+5000000)}

func main() {pool := make([][]byte, 20)

```
var m runtime.MemStats
makes := 0
for {
b := makeBuffer()
makes += 1
i := rand.Intn(len(pool))
pool\[i\] = b
time.Sleep(time.Second)
bytes := 0
for i := 0; i < len(pool); i++ {
if pool\[i\] != nil {
bytes += len(pool\[i\])
}
}
runtime.ReadMemStats(&m)
fmt.Printf("%d,%d,%d,%d,%d,%d\\n", m.HeapSys, bytes, m.HeapAlloc,
m.HeapIdle, m.HeapReleased, makes)
}
```


}

The program uses the runtime.ReadMemStats function to get information about the size of the heap. It prints out four values: `HeapSys`

(the number of bytes the program has asked the operating system for), `HeapAlloc`

(the number of bytes currently allocated in the heap), `HeapIdle`

(the number of bytes in the heap that are unused) and `HeapReleased`

(the number of bytes returned to the operating system).

Garbage collection runs frequently in a Go program (see the `GOGC`

environment variable to understand how to control the garbage collector's operation), so when running the size of the heap will change as memory is spotted as unused: this will causes `HeapAlloc`

and `HeapIdle`

to change over time. The scavenger will only release memory that has been unused for more than 5 minutes so `HeapReleased`

does not change often.

Here's a chart of the program above running for ten minutes.

(On this and subsequent charts the left axis in in bytes and the right axis is a count used for Make Count)

The red line shows the number of bytes that are actually in byte buffers in the pool. Because it has a fixed size of 20 buffers it quickly plateaus at around 150,000,000 bytes. The top blue line shows the amount of memory request from the operating system; it plateaus at around 375,000,000 bytes. So the program is using about 2.5x the amount of memory it needs.

As the program is churning through memory the allocated size of the heap and the amount of idle memory jumps around as garbage collection occurs. The orange line just counts the number of times `makeBuffer()`

has been called to create a new buffer.

This sort of over request of memory is common in garbage collected programs (see, for example, the paper Quantifying the Performance of Garbage Collection vs. Explicit Memory Management). As the program churns through memory the idle memory gets reused and rarely gets released to the operating system.

One way to solve this problem is to perform some memory management manually in the program. For example, using a channel it's possible to keep a separate pool of buffers that are no longer used and use that pool to retrieve a buffer (or make a new one if the channel is empty).

The program can then be rewritten as follows:

package main

import ( "fmt" "math/rand" "runtime" "time" )

func makeBuffer() []byte { return make([]byte, rand.Intn(5000000)+5000000) }

func main() { pool := make([][]byte, 20)

```
buffer := make(chan \[\]byte, 5)
var m runtime.MemStats
makes := 0
for {
var b \[\]byte
select {
case b = <-buffer:
default:
makes += 1
b = makeBuffer()
}
i := rand.Intn(len(pool))
if pool\[i\] != nil {
select {
case buffer <- pool\[i\]:
pool\[i\] = nil
default:
}
}
pool\[i\] = b
time.Sleep(time.Second)
bytes := 0
for i := 0; i < len(pool); i++ {
if pool\[i\] != nil {
bytes += len(pool\[i\])
}
}
runtime.ReadMemStats(&m)
fmt.Printf("%d,%d,%d,%d,%d,%d\\n", m.HeapSys, bytes, m.HeapAlloc,
m.HeapIdle, m.HeapReleased, makes)
}
```


}

And here's a graph of its operation for 10 minutes:

This tells a very different story from above. The memory actually in pool is very close to the memory requested from the operating system and the garbage collector has little to no work to do. The heap only has a very small amount of idle memory that eventually gets released to the operating system.

The key to the operation of this memory recycling mechanism is a buffered channel called `buffer`

. In the code above it can store 5 `[]byte`

slices. When the program needs a buffer it first tries to read one from the channel using a `select`

trick:

select { case b = <-buffer: default: b = makeBuffer() }

This will never block because if the channel called `buffer`

has a slice in it then the first case works and the slice is placed in `b`

. If the channel is empty (meaning a receive would block) the `default`

case happens a new buffer is allocated.

Putting slices back in the channel uses a similar non-blocking trick:

select { case buffer <- pool[i]: pool[i] = nil default: }

If the `buffer`

channel is full then a send to it would block. In that case the empty `default`

clause occurs. This simple mechanism can be used to safely make a shared pool. It can even be shared across goroutines since channel communication is perfectly safe from multiple goroutines.

We actually use a similar technique to this in our Go programs. The actual recycler (in simplified form) is shown below. It works by having a goroutine that handles the creation of buffers and shares them across goroutines in our software. Two channels `get`

(to get a new buffer) and `give`

(to return a buffer to the pool) are used for all communication.

The recycler keeps a linked list of returned buffers and periodically throws away those that are too old and are unlikely to be reused (in the example code that's buffers that are older than one minute). That allows the program to cope with bursts of demand for buffers dynamically.

package main

import ( "container/list" "fmt" "math/rand" "runtime" "time" )

var makes int var frees int

func makeBuffer() []byte { makes += 1 return make([]byte, rand.Intn(5000000)+5000000) }

type queued struct { when time.Time slice []byte }

func makeRecycler() (get, give chan []byte) { get = make(chan []byte) give = make(chan []byte)

```
go func() {
q := new(list.List)
for {
if q.Len() == 0 {
q.PushFront(queued{when: time.Now(), slice: makeBuffer()})
}
e := q.Front()
timeout := time.NewTimer(time.Minute)
select {
case b := <-give:
timeout.Stop()
q.PushFront(queued{when: time.Now(), slice: b})
case get <- e.Value.(queued).slice:
timeout.Stop()
q.Remove(e)
case <-timeout.C:
e := q.Front()
for e != nil {
n := e.Next()
if time.Since(e.Value.(queued).when) > time.Minute {
q.Remove(e)
e.Value = nil
}
e = n
}
}
}
}()
return
```


}

func main() { pool := make([][]byte, 20)

```
get, give := makeRecycler()
var m runtime.MemStats
for {
b := <-get
i := rand.Intn(len(pool))
if pool\[i\] != nil {
give <- pool\[i\]
}
pool\[i\] = b
time.Sleep(time.Second)
bytes := 0
for i := 0; i < len(pool); i++ {
if pool\[i\] != nil {
bytes += len(pool\[i\])
}
}
runtime.ReadMemStats(&m)
fmt.Printf("%d,%d,%d,%d,%d,%d,%d\\n", m.HeapSys, bytes, m.HeapAlloc
m.HeapIdle, m.HeapReleased, makes, frees)
}
```


}

Running that program for ten minutes looks very similar to the second program above:

These techniques can be used to recycle memory that the programmer knows is likely to be reused without asking the garbage collector to do the work. They can significantly reduce the amount of memory a program needs. And they can be used for more than just byte slices. Any arbitrary Go type (user-defined or not) could be recycled in a similar manner.