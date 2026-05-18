---
title: Debugging shared library problems with strace
source: https://jvns.ca/blog/2014/03/10/debugging-shared-library-problems-with-strace/
kind: external
domain: tracing
author: Julia Evans
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, tracing]
---

> [!info] External article · imported reference
> Source: [jvns.ca](https://jvns.ca/blog/2014/03/10/debugging-shared-library-problems-with-strace/)
> Author: Julia Evans
> License: cc-by-sa
> Fetched: 2026-05-18

# Debugging shared library problems with strace

# Debugging shared library problems with strace

March 10, 2014

It’s official. I have a love affair with strace.

So strace is this Linux command that shows you what system calls a program
calls.

This doesn’t sound so useful until you find out that it is useful FOR
EVERYTHING. Seriously. strace is like an immersion blender. I use strace more
than my immersion blender.

Previously we have used strace to
[find out how killall works](http://jvns.ca/blog/2013/12/22/fun-with-strace/),
[spy on ssh](http://jvns.ca/blog/2014/02/17/spying-on-ssh-with-strace/),
[avoid reading Ruby code](http://jvns.ca/blog/2014/02/26/using-strace-to-avoid-reading-ruby-code/), and
[more](http://jvns.ca/blog/2014/02/27/more-practical-uses-for-strace/).

So today I had was trying to install the
[IRuby notebook](https://github.com/minad/iruby/). But my version of libzmq was wrong! So I upgraded it. But it was STILL WRONG. Why? WHY?

So I thought, I will get strace to tell me which shared libraries are being loaded! strace will never lie to me. Here’s how to do that:

```
strace -f -o /tmp/iruby_problems ~/clones/iruby/bin/iruby notebook
grep libzmq.so /tmp/iruby_problems | grep -v ENOENT
```

The `grep -v ENOENT` is because it looks everywhere in my LD_LIBRARY_PATH so it
fails to find libzmq a bunch of times. This reveals the following two system
calls:

```
28863 open("/opt/anaconda/lib/python2.7/site-packages/zmq/utils/../../../../libzmq.so.3", O_RDONLY|O_CLOEXEC) = 9
28910 open("/usr/lib/libzmq.so", O_RDONLY|O_CLOEXEC) = 9
```

AH HA. The first libzmq is the right version (`libzmq.so.3`), but the second one is all wrong! It is `libzmq1` and it is a disaster and a disgrace. I did `sudo apt-get remove libzmq1` and the offending `libzmq` was banished from my system.

Thanks, strace :)
