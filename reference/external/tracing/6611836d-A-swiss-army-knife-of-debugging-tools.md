---
title: A swiss army knife of debugging tools
source: https://jvns.ca/blog/2016/09/17/strange-loop-talk/
kind: external
domain: tracing
author: Julia Evans
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, tracing]
---

> [!info] External article · imported reference
> Source: [jvns.ca](https://jvns.ca/blog/2016/09/17/strange-loop-talk/)
> Author: Julia Evans
> License: cc-by-sa
> Fetched: 2026-05-18

# A swiss army knife of debugging tools

Before we move on, I want to talk about sampling vs tracing for a little bit.
In a sampling profiler, you look at a small percentage of what’s going on in
your program (what’s the stack trace now? how about now?), and then use that information to generalize about what your program is doing.

This is great because it reduces overhead, and usually gives you a good idea of what your program is doing.

But what happens if you have an error that happens infrequently? like 1 in 1000 times. You might think this is not a big deal, and in fact for a lot of people that might not be a big deal.

But I write programs that do things millions or billions of times, and I want
those things to be really highly reliable, so even relatively rare events are
important to me. So I love tracing tools and log files that can tell me
**everything** about when a certain function is called. This makes it a lot easier to debug than just having a general statistical distribution!   
 This is a great [post about tracing tools](http://danluu.com/perf-tracing/) by Dan Luu.
