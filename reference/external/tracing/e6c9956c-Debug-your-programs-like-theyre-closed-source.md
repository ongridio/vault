---
title: Debug your programs like they're closed source
source: https://jvns.ca/blog/2014/04/20/debug-your-programs-like-theyre-closed-source/
kind: external
domain: tracing
author: Julia Evans
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, tracing]
---

> [!info] External article · imported reference
> Source: [jvns.ca](https://jvns.ca/blog/2014/04/20/debug-your-programs-like-theyre-closed-source/)
> Author: Julia Evans
> License: cc-by-sa
> Fetched: 2026-05-18

# Debug your programs like they're closed source

Until very recently, if I was debugging a program, I practically
always did one of these three things:

1. open a debugger
2. look at the source code
3. insert some print statements

I’ve started sometimes debugging a new way. With this method, I don’t
look at the source code, don’t edit the source code, and don’t use a
debugger. I don’t even need to have the program’s source available to
me!

Can we repeat that again? I can look at the internal behavior of
*closed-source programs*.

How?!?! AM I A WIZARD? Nope. SYSTEM CALLS! What is a system call?
Operating systems know how to open files, display things to the
screen, start processes, and all kinds of things. Programs can ask
their operating system to do these things, using functions called
**system calls**.

System calls are the API for your computer, so you don’t have to know
how a network card works to send a HTTP request.

Here’s a list of the
[system calls Linux 2.2](http://docs.cs.up.ac.za/programming/asm/derick_tut/syscalls.html)
provides, to give you a sense for what’s available. There’s `exit`,
`open`, `read`, `write`, `time`, `mount`, `kill`, and all kinds of
other things. System calls are basically the definition of platform
specific (different operating system have different system calls), so
we’re only going to be talking about Linux here.

How can we use these to debug? Here are a few of my favorite system
calls!

`open` opens files. Every time any program opens a file it needs to
use the `open` system call. There’s no other way.

So! Let’s say you have a backup program on your computer, and you want
to know which files it’s working on. And that it doesn’t show you a
progress bar or have any options. Let’s say that it has PID 60.

We can spy on this program with a tool called `strace` and print out
every file it opens! `strace` shows you which system calls a program
calls. To spy on our backup program, we would run `strace -e trace=open -p 60`, to tell it to print all the `open` system calls from PID 60.

For example, I ran `strace -e trace=open ssh` and here were some of the
things I found:

```
open("/etc/ssh/ssh_config", O_RDONLY)   = 3
open("/home/bork/.ssh/config", O_RDONLY) = 3
open("/home/bork/.ssh/id_dsa", O_RDONLY) = 4
open("/home/bork/.ssh/id_dsa.pub", O_RDONLY) = 4
open("/home/bork/.ssh/id_rsa", O_RDONLY) = 4
open("/home/bork/.ssh/id_rsa.pub", O_RDONLY) = 4
open("/home/bork/.ssh/known_hosts", O_RDONLY) = 4
```

This makes total sense! `ssh` needs to read my private and public
keys, my local ssh config, and the global ssh config. Neat! `open` is
super simple and super useful.

`execve` starts programs. All programs. There’s no way to start a
program except to use `execve`. We can use `strace` to spy on `execve`
calls too!

For example! I was trying to understand a Ruby script that was
basically just running some `ssh` commands. I could have read the Ruby
code! But I really just wanted to know which damn command it was
running! I did this by running `strace -f -s3000 -e trace=execve` and
read zero code!

The `-f` option is super important here. It also tracks the system
calls of every subprocess! I basically use `-f` all the time. Use
`-f`.
([[longer blog post about using strace + execve to poke at Ruby programs]](/blog/2014/02/26/using-strace-to-avoid-reading-ruby-code/)).

`write` writes to files. I think there are ways to write to a file
without using `write` (like by using `mmap`), but *usually* if a file
is being written to, it’s using `write`.

If I `strace -e trace=write` on an `ssh` session, this is some of what
I see:

```
write(3, "SSH-2.0-OpenSSH_5.9p1 Debian-5ubuntu1.1\r\n", 41) = 41
[...]
write(5, "[jvns /home/public]$ ", 21)   = 21
write(3, "\242\227e\376\344\36\270\343\331\307\231\332\373\273\324\303X\n<\241p`\212\21\317\353`\1/\3629\273m\23\17\26\304\fJ\352z\210\2\210\211~7W", 48) = 48
write(5, "logout\r\n", 8)               = 8
write(3, "b\277\306\16!\6J\202\tF$\241\32\302\3\0\23\310\346f\241\233\263\254\325\351z\222\234\224\270\231", 32) = 32
write(3, "\311\372\353\273\233oU\226~\373N\227\323*S\263\307\272\204VzO \10\2\316\224\335X@Hj\26\366\271J:i6\311\240A\325\331\341\220\1%\233\240\23n\23\242\34\277\2139\376\31j\255\32h", 64) = 64
write(2, "Connection to ssh.phx.nearlyfreespeech.net closed.\r\n", 52) = 52
```

So it opens an SSH connection, writes a prompt to my terminal, sends
some (encrypted!) data over the connection, and prints that the
connection is closed! Neat! I understand a bit more about how ssh
works now!

I want to talk about one more Linux thing, and it isn’t a system call.
It’s a directory called `/proc`! There are a million things that
`/proc` does, but this is my favorite:

`/proc` tells you every file your process has open. All of them! For
example, one of my Chrome processes has PID 3823. If I run `ls -l /proc/3823/fd/*`, it shows me all the files Chrome has open!

`fd` stands for “file descriptor”.

```
$ ls -l /proc/3823/fd/*
total 0
lr-x------ 1 bork bork 64 Apr 19 09:28 0 -> /dev/null
l-wx------ 1 bork bork 64 Apr 19 09:28 1 -> /dev/null
lrwx------ 1 bork bork 64 Apr 19 09:28 10 -> socket:[16583]
lr-x------ 1 bork bork 64 Apr 19 09:28 100 -> /opt/google/chrome/nacl_irt_x86_64.nexe
lrwx------ 1 bork bork 64 Apr 19 09:28 101 -> /home/bork/.config/google-chrome/Default/Application Cache/Cache/index
lrwx------ 1 bork bork 64 Apr 19 09:28 102 -> /home/bork/.config/google-chrome/Default/Application Cache/Cache/data_0
lrwx------ 1 bork bork 64 Apr 19 09:28 103 -> socket:[178726]
lrwx------ 1 bork bork 64 Apr 19 09:28 104 -> socket:[21064]
lrwx------ 1 bork bork 64 Apr 19 09:28 105 -> /home/bork/.config/google-chrome/Default/Application Cache/Cache/data_1
lrwx------ 1 bork bork 64 Apr 19 09:28 106 -> /home/bork/.config/google-chrome/Default/Application Cache/Cache/data_2
lrwx------ 1 bork bork 64 Apr 19 09:28 107 -> /home/bork/.config/google-chrome/Default/Application Cache/Cache/data_3
```

aaaand a million more. This is great. There are also a ton more things
in `/proc/3823`. Look around! I wrote a bit more about `/proc` in
[Recovering files using /proc (and spying, too!)](http://jvns.ca/blog/2014/03/23/recovering-files-using-slash-proc-and-other-useful-facts/).

Lots of things happen outside of the kernel. Like string comparisons!
I don’t need a network card for that! What if we wanted to know about
those? `strace` won’t help us at all. But `ltrace` will!

Let’s try running `ltrace killall firefox`. We see a bunch of things
like this:

```
fopen("/proc/10578/stat", "r")                               => 0x11984f0
free(0x011984d0)
fscanf(0x11984f0, 0x403fe7, 0x7fff09984980, 0x7f2fc7cd4728, 0)
fclose(0x11984f0)
strcmp("firefox", "kworker/u:0")
```

So! We’ve just learned that `killall` works by opening a file in
`/proc` (wheeee!), finding what its name is, and seeing if it’s the
same as “firefox”. That makes sense!

These systems-level debugging tools are only appropriate sometimes. If
you’re writing a graph traversal algorithm and it has a logical error,
knowing which files it opened won’t help you at all!

Here are some examples of times when using systems tools might make
your life easier:

- Is your program running a command, but the wrong one? Look at
  `execve`!
- Your program communicates with something on a network, but some of
  the information it’s sending is wrong? It’s probably sending it with
  `write`, `sendto`, or `send`.
- Your program writes to a file, but you don’t know what file it’s
  writing to? Use `/proc` to see what files it has open, or look at
  what it’s `write`ing. `/proc` doesn’t lie.

At first debugging this way is confusing, but once you’re familiar
with the tools it can actually be faster, because you don’t have to
worry about getting the wrong information! And you feel like a WIZARD.

There are all kinds of programming-language-specific debugging tools
you can use. `gdb`! `pry`! `pdb`! And you should! But you probably
switch languages more often than you switch OSes. So, learning your OS
in depth and then using it as a debugging tool is likely a better
investment of your time than learning a language-specific debugging
tool.

If you want to know which files a process has open, it doesn’t matter
if that program was originally written in C++ or Python or Java or
Haskell. The *only way* for a program to open a file on Linux is with
the `open` system call. If you learn your operating system, you
acquire superpowers. You can debug programs that are binary-only and
closed source. You can use the same tools to debug no matter which
language you’re writing.

And my favorite thing about these methods is that your OS won’t lie to
you. The **only way** to run a program is with the `execve` system
call. There aren’t other ways. So if you really want to know what
command got run, use `strace`. See exactly which parameters get passed
to `execve`. You’ll know exactly what happened.

Try Greg Price’s excellent blog post
[Strace – The Sysadmin’s Microscope](https://blogs.oracle.com/ksplice/entry/strace_the_sysadmin_s_microscope).
I have an
[ever-growing collection of blog posts about strace](/categories/strace),
too!

My favorite way to learn more, honestly, is to just strace random
programs and see what I find out. It’s a great way to spend a rainy
Sunday afternoon! =)

Thanks to [Lindsey Kuper](http://decomposition.al/) and
[Dan Luu](http://danluu.com) for reading a draft of this :)

Have fun!
