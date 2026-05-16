---
title: Goroutine IDs
source: https://blog.sgmansfield.com/2015/12/goroutine-ids/
kind: external
domain: observability
original_date: 2015-12-16
fetched_at: 2026-05-16
bookmark_title: Goroutine IDs · Scott Mansfield
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.sgmansfield.com](https://blog.sgmansfield.com/2015/12/goroutine-ids/)
> 原始日期：2015-12-16
> 抓取日期：2026-05-16

# Goroutine IDs

# Goroutine IDs

Dec 16, 2015 · 3 minute read · CommentsGoHacks

## Do Goroutine IDs Exist?

Yes, of course. The runtime has to have some way to track them.

## Should I use them?

## Are There Packages I Can Use?

There exist packages from people on the Go team, with colorful descriptions like “If you use this package, you will go straight to hell.“

There are also packages to do goroutine local storage built on top of goroutine IDs, like github.com/jtolds/gls and github.com/tylerb/gls that use dirty knowledge from the above to create an environment contrary to Go design principles.

## Minimal Code

So you like pain? Here’s how to get the current goroutine ID:

### The Hacky Way in “Pure” Go

This is adapted from Brad Fitzpatrick’s http/2 library that is being rolled into Go 1.6. It was used solely for debugging purposes, and is not used during regular operation

```
package main
import (
"bytes"
"fmt"
"runtime"
"strconv"
)
func main() {
fmt.Println(getGID())
}
func getGID() uint64 {
b := make([]byte, 64)
b = b[:runtime.Stack(b, false)]
b = bytes.TrimPrefix(b, []byte("goroutine "))
b = b[:bytes.IndexByte(b, ' ')]
n, _ := strconv.ParseUint(string(b), 10, 64)
return n
}
```


#### How does this work?

It is possible to parse debug information (meant for human consumption, mind you) and retrieve the
goroutine ID. There’s even **debug** code in the http/2 library that uses this to track ownership of
connections. It was used sparingly, however, for debugging purposes only. It’s not meant to be a
high-performance operation.

The debug information itself was retrieved by calling `runtime.Stack(buf []byte, all bool) int`

which will print a stacktrace into the buffer it’s given in
textual form. The very first line in the stacktrace is the text “goroutine #### […” where #### is
the actual goroutine ID. The rest is just text manipulation to extract and parse the number. I took
out some error handling, but if you want the goroutine ID in the first place, I assume you want to
live dangerously already.

### The Legitimate CGo Version

The C version is from github.com/davecheney/junk/id where the C code directly accesses the `goid`

property of the current goroutine and just
returns that. The code below was copied verbatim from Dave Cheney’s repo, so all credit to him.

File `id.c`


```
#include "runtime.h"
int64 ·Id(void) {
return g->goid;
}
```


File `id.go`


```
package id
func Id() int64
```


## Where to go from here?

Running screaming away from goroutine IDs. Forget they exist, and you will be much happier. The use of them is a red flag from a design standpoint, because nearly all uses (speaking from research, not tacit knowledge) try to create something to do goroutine-local state, which violates the “Share Memory By Communicating” tenet of Go programming.