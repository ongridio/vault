---
title: What problems do people solve with strace?
source: https://jvns.ca/blog/2021/04/03/what-problems-do-people-solve-with-strace/
kind: external
domain: tracing
author: Julia Evans
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, tracing]
---

> [!info] External article · imported reference
> Source: [jvns.ca](https://jvns.ca/blog/2021/04/03/what-problems-do-people-solve-with-strace/)
> Author: Julia Evans
> License: cc-by-sa
> Fetched: 2026-05-18

# What problems do people solve with strace?

Yesterday I [asked on Twitter about what problems people are solving with strace](https://twitter.com/b0rk/status/1378014888405168132) and
as usual everyone really delivered! I got 200 answers and then spent a bunch of
time manually categorizing them into 9 categories of problems.

All of the problems are about either finding files a program depends on,
figuring out why a program is stuck or slow, or finding out why a program is
failing. These generally matched up with what I use strace for myself, but
there were some things I hadn’t thought of too!

I’m not going to explain what strace is in this post but I have a [free zine about it](https://wizardzines.com/zines/strace) and [a talk](https://www.youtube.com/watch?v=4pEHfGKB-OE) and [lots of blog posts](https://jvns.ca/categories/strace).

The #1 most popular problem was “this program has a configuration file and I don’t
know where it is”. This is probably my most common use for strace too, because it’s such a simple question.

This is great because there are a million ways for a program to document where
its config file is (in a man page, on its website, in `--help`, etc), but
there’s only one way for it to actually open it (with a system call!)

You can also use strace to find other types of files a program depends on, like:

You have a program, it’s just sitting there doing nothing, what’s going
on? This one is especially easy to answer because a lot of the time you just
need to run `strace -p PID` and look at what system call is currently running.
You don’t even have to look through hundreds of lines of output!

The answer is usually ‘waiting for some kind of I/O’. Some possible answers for “why is this stuck” (though there are a lot more!):

- it’s polling forever on a `select()`
- it’s `wait()`ing for a subprocess to finish
- it’s making a network request to something that isn’t responding
- it’s doing `write()` but it’s blocked because the buffer is full
- it’s doing a `read()` on stdin and it’s waiting for input

Someone also gave a nice example of using strace to debug a stuck `df`: ‘with strace df -h you can find the stuck mount and unmount it".

A variation on the previous one: sometimes a program has been running for
longer than you expected, and you just want to know if it’s stuck or of it’s
still making progress.

As long as the program makes system calls while it’s running, this is super
easy to answer with strace – just strace it and see if it’s making new
system calls!

You can use strace as a sort of coarse profiling tool – `strace -t` will show
the timestamp of each system call, so you can look for big gaps and find the culprit.

Here are 9 short stories from Twitter of people using strace to debug “why is this program slow?”.

- Back in 2000, a Java-based web site that I helped support was dying under
  modest load: pages loaded slowly, if at all. We straced the J2EE application
  server and found that it was reading class files one. byte. at. a. time. Devs
  weren’t using BufferedReader, classic Java mistake.
- Optimizing app startup times… running strace can be an eye-opening
  experience, in terms of the amount of unnecessary file system interaction
  going on (e.g. open/read/close on the same config file over and over again;
  loading gobs of font files over a slow NFS mount, etc)
- Asked myself why reading from session files in PHP (usually <100 bytes)
  was incredibly slow. Turned out some `flock`-syscalls took ~60s
- A program was behaving abnormally slow. Used strace to figure out it was
  re-initializing its internal pseudo-random number generator on every request
  by reading from /dev/random and exhausting entropy
- Last thing I remember was attaching to a job worker and seeing just how many network calls it was making (which was unexpected).
- Why is this program so slow to start? strace shows it opening/reading the same config file thousands of times.
- Server using 100% CPU time randomly with low actual traffic. Turns out it’s hitting the number of open files limit accepting a socket, and retrying forever after getting EMFILE and not reporting it.
- A workflow was running super slow but no logs, ends up it was trying to do a post request that was taking 30s before timing out and then retrying 5 times… ends up the backend service was overwhelmed but also had no visibility
- using strace to notice that gethostbyname() is taking a long time to return (you can’t see the `gethostbyname` directly but you can see the DNS packets in strace)

Sometimes a program is failing for a mysterious reason, but the problem is just
that there’s some file that it doesn’t have permission to open. In an ideal
world programs would report those errors (“Error opening file /dev/whatever:
permission denied”), but of course the world is not perfect, so strace can
really help with this!

This is actually the most recent thing I used strace for: I was using an
AxiDraw pen plotter and it printed out an inscrutable error message when I
tried to start it. I `strace`d it and it turned out that my user just didn’t
have permission to open the USB device.

Sometimes a script is running another program, and you want to know what
command line flags it’s passing!

A couple of examples from Twitter:

- find what compiler flags are actually being used to build some code
- a command was failing due to having too long a command line

Basically the goal here is just to find which domain / IP address the network
connection is being made to. You can look at the DNS request to find the domain
or the `connect` system call to find the IP.

In general there are a lot of stories about using strace to debug network
issues when `tcpdump` isn’t available for some reason or just because it’s what
the person is more familiar with.

For example:

- the same binary works on one machine, fails on another machine
- works when you run it, fails when spawned by a systemd unit file
- works when you run it, fails when you run it as “su - user /some/script”
- works when you run it, fails when run as a cron job

Being able to compare the strace output in both cases is very helpful. Though
my first step when debugging “this works as my user and fails when run in a
different way on the same computer” would be “look at my environment
variables”.

Another one quite a few people mentioned is figuring out how a Linux kernel
API (for example netlink, io_uring, hdparm, I2C, etc).

Even though these APIs are usually documented, sometimes the documentation is
confusing or there aren’t very many examples, so often it’s easier to just
strace an existing application and see how it interacts with the Linux kernel.

strace is also great for just generally figuring out “how does this program
work?”. As a simple example of this, here’s a blog post on [figuring out how killall works using strace](https://jvns.ca/blog/2013/12/22/fun-with-strace/).

The reason I’m thinking about this is that I’ve been slowly working on some
challenges to help people practice using strace and other command line tools.
The idea is that you’re given a problem to solve, a terminal, and you’re free
to solve it in any way you want.

So my goal is to use this to build some practice problems that you can solve
with strace that reflect the kinds of problems that people actually use it for
in real life.

There are probably more problems that can be solved with strace that I haven’t
covered here – I’d love to hear what I’ve missed!

I really loved seeing how many of the same uses came up over and over and over
again – at least 20 different people replied saying that they use strace to
find config files. And as always I think it’s really delightful how such a
simple tool (“trace system calls!”) can be used to solve so many different
kinds of problems.
