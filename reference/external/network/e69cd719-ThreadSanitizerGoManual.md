---
title: ThreadSanitizerGoManual
source: https://github.com/google/sanitizers/wiki/ThreadSanitizerGoManual
kind: external
domain: network
author: Google
original_date: 2015-08-31
fetched_at: 2026-05-16
bookmark_title: ThreadSanitizerGoManual · google/sanitizers Wiki
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[github.com](https://github.com/google/sanitizers/wiki/ThreadSanitizerGoManual)
> 作者：Google
> 原始日期：2015-08-31
> 抓取日期：2026-05-16

# ThreadSanitizerGoManual

-
Notifications
You must be signed in to change notification settings - Fork 1.1k

# ThreadSanitizerGoManual

**Note: The data race detector is not yet available as part of any release. To use it you need to build Go tip from sources. It will be available in the Go 1.1 release.**

Data races are one of the most common and hardest to debug types of bugs in concurrent systems. A data race occurs when two goroutines access the same variable w/o proper synchronization and at least one of the accesses is write.

Here is an example of a data race on map variable that can lead to crashes and memory corruptions:

```
func main() {
c := make(chan bool)
m := make(map[string]string)
go func() {
m["1"] = "a" // First conflicting access.
c <- true
}()
m["2"] = "b" // Second conflicting access.
<-c
for k, v := range m {
fmt.Println(k, v)
}
}
```


Fortunately, Go includes built-in data race detector. The usage is very simple -- you just need to add -race flag to go command:

```
$ go test -race mypkg // to test the package
$ go run -race mysrc.go // to run the source file
$ go build -race mycmd // to build the command
$ go install -race mypkg // to install the package
```


When the race detector finds a data race in the program, it prints an informative report. The report contains stack traces for conflicting accesses, as well as stacks where the involved goroutines were created. You may see an example below:

```
WARNING: DATA RACE
Read by goroutine 185:
net.(*pollServer).AddFD()
src/pkg/net/fd_unix.go:89 +0x398
net.(*pollServer).WaitWrite()
src/pkg/net/fd_unix.go:247 +0x45
net.(*netFD).Write()
src/pkg/net/fd_unix.go:540 +0x4d4
net.(*conn).Write()
src/pkg/net/net.go:129 +0x101
net.func·060()
src/pkg/net/timeout_test.go:603 +0xaf
Previous write by goroutine 184:
net.setWriteDeadline()
src/pkg/net/sockopt_posix.go:135 +0xdf
net.setDeadline()
src/pkg/net/sockopt_posix.go:144 +0x9c
net.(*conn).SetDeadline()
src/pkg/net/net.go:161 +0xe3
net.func·061()
src/pkg/net/timeout_test.go:616 +0x3ed
Goroutine 185 (running) created at:
net.func·061()
src/pkg/net/timeout_test.go:609 +0x288
Goroutine 184 (running) created at:
net.TestProlongTimeout()
src/pkg/net/timeout_test.go:618 +0x298
testing.tRunner()
src/pkg/testing/testing.go:301 +0xe8
```


You can pass some options to the race detector by means of `GORACE`

environment variable. The format is:

```
GORACE="option1=val1 option2=val2"
```


The options are:

-
**log_path**: Tells race detector to write reports to 'log_path.pid' file. The special values are 'stdout' and 'stderr'. The default is 'stderr'. -
**strip_path_prefix**: Allows to strip beginnings of file paths in reports to make them more concise. -
**history_size**: Per-goroutine history size, controls how many previous memory accesses are remembered per goroutine. Possible values are [0..7]. history_size=0 amounts to 32K memory accesses. Each next value doubles the amount of memory accesses, up to history_size=7 that amounts to 4M memory accesses. The default value is 1 (64K memory accesses). Try to increase this value when you see "failed to restore the stack" in reports. However, it can significantly increase memory consumption.

Example:

```
$ GORACE="log_path=/tmp/race/report strip_path_prefix=/my/go/sources/" go test -race
```


You may start with just running your tests under the race detector (go test -race). However sometimes tests have limited coverage, especially with respect to concurrency. The race detector finds only races that actually happen in the execution, it can't find races in code paths that were not executed. So it may be beneficial to run the whole program built with -race under a realistic workload, frequently it discovers much more bugs than tests.

Here are some example of typical data races. All of them can be automatically detected with the race detector.

```
func main() {
var wg sync.WaitGroup
wg.Add(5)
for i := 0; i < 5; i++ {
go func() {
fmt.Println(i) // Not the 'i' you are looking for.
wg.Done()
}()
}
wg.Wait()
}
```


Closures capture variables by reference rather than by value, so the reads of the 'i' variable in the goroutines race with 'i' increment in the loop statement. Such program typically outputs 55555 instead of expected 01234. The program can be fixed by explicitly making a copy of the loop counter:

```
func main() {
var wg sync.WaitGroup
wg.Add(5)
for i := 0; i < 5; i++ {
go func(j int) {
fmt.Println(j) // Good. Read local copy of the loop counter.
wg.Done()
}(i)
}
wg.Wait()
}
```


```
// ParallelWrite writes data to file1 and file2, returns the errors.
func ParallelWrite(data []byte) chan error {
res := make(chan error, 2)
f1, err := os.Create("file1")
if err != nil {
res <- err
} else {
go func() {
// This err is shared with the main goroutine,
// so the write races with the write below.
_, err = f1.Write(data)
res <- err
f1.Close()
}()
}
f2, err := os.Create("file2") // The second conflicting write to err.
if err != nil {
res <- err
} else {
go func() {
_, err = f2.Write(data)
res <- err
f2.Close()
}()
}
return res
}
```


The fix is simple, one just needs to introduce new variables in the goroutines (note :=):

```
_, err := f1.Write(data)
...
_, err := f2.Write(data)
```


If the following code is called from several goroutines, it leads to bad races on the services map.

```
var services map[string]net.Addr
func RegisterService(name string, addr net.Addr) {
services[name] = addr
}
func GetService(name string) net.Addr {
return services[name]
}
```


It can be fixed by protecting the accesses with a mutex:

```
var services map[string]net.Addr
var mu sync.Mutex
func RegisterService(name string, addr net.Addr) {
mu.Lock()
defer mu.Unlock()
services[name] = addr
}
func GetService(name string) net.Addr {
mu.Lock()
defer mu.Unlock()
return services[name]
}
```


Data races can happen on variables of primitive types as well (bool, int, int64), like in the following example:

```
type Watchdog struct { last int64 }
func (w *Watchdog) KeepAlive() {
w.last = time.Now().UnixNano() // First conflicting access.
}
func (w *Watchdog) Start() {
go func() {
for {
time.Sleep(time.Second)
// Second conflicting access.
if w.last < time.Now().Add(-10*time.Second).UnixNano() {
fmt.Println("No keepalives for 10 seconds. Dying.")
os.Exit(1)
}
}
}()
}
```


Even such "innocent" data races can lead to hard to debug problems caused by (1) non-atomicity of the memory accesses, (2) interference with compiler optimizations and (3) processor memory access reordering issues.

To fix such data race one can use (aside from chan and sync.Mutex) package sync/atomic, which provides atomic operations on primitive types. sync/atomic functions solve all of the above issues.

```
type Watchdog struct { last int64 }
func (w *Watchdog) KeepAlive() {
atomic.StoreInt64(&w.last, time.Now().UnixNano())
}
func (w *Watchdog) Start() {
go func() {
for {
time.Sleep(time.Second)
if atomic.LoadInt64(&w.last) < time.Now().Add(-10*time.Second).UnixNano() {
fmt.Println("No keepalives for 10 seconds. Dying.")
os.Exit(1)
}
}
}()
}
```


Supported platforms are darwin/amd64, linux/amd64 and windows/amd64.

The data race detector significantly increases both memory consumption and execution time. The concrete numbers highly dependent on the particular program, but some reference numbers would be: memory consumption ~5-10x, execution time ~2-20x.

Send comments/questions to thread-sanitizer@googlegroups.com