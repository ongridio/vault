---
title: Spying on ssh with strace
source: https://jvns.ca/blog/2014/02/17/spying-on-ssh-with-strace/
kind: external
domain: tracing
author: Julia Evans
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, tracing]
---

> [!info] External article · imported reference
> Source: [jvns.ca](https://jvns.ca/blog/2014/02/17/spying-on-ssh-with-strace/)
> Author: Julia Evans
> License: cc-by-sa
> Fetched: 2026-05-18

# Spying on ssh with strace

In the shower this morning I was thinking about strace and ltrace and
how they let you inspect the system calls a running process is making.
I’ve played a bit with strace on this blog before (see
[Understanding how killall works using strace](http://jvns.ca/blog/2013/12/22/fun-with-strace/)),
but it’s clear to me that there are tons of uses for it I haven’t
explored yet.

Then I thought “Hey! If you can look at the system calls with strace
and the library calls with ltrace, can you spy on people’s ssh
passwords?!”

It turns out that you can! I was going to do original research, but as
with most things one thinks up in the shower, it turns out someone’s
already done this before. So I googled it and I found this
[blog post explaining how to spy on ssh](http://pentestmonkey.net/blog/sshd-snooping).
The instructions here are just taken from there :)

The reason this is possible is that strace doesn’t just tell you which
system calls a given program is running. It also tells you what the
arguments are! So if a program ever calls a function with a password
the odds are pretty good that you can find out the password this way.

To do this you need to already be root, so it’s not a vulnerability or
anything. This just means that if your machine is already compromised,
it’s really, really, compromised. Here’s how it works:

I have a running ssh server on my machine, so I sshd to my laptop:

`$ ssh asdf@localhost`

`sshd` forks and creates a couple of new processes to handle the
incoming ssh connection. I can find them using `ps`:

```
bork@kiwi /tmp> ps aux | grep sshd
root      1242  0.0  0.0  50036   908 ?        Ss   Jan21   0:00 /usr/sbin/sshd -D
root      9412  0.0  0.0 101536  4104 ?        Ss   11:29   0:00 sshd: unknown [priv]
sshd      9413  0.0  0.0  51468  1356 ?        S    11:29   0:00 sshd: unknown [net]
```

Then I can use `strace` to spy on what the child process is doing. It
passes the password to the main `sshd` process, and that’s where we
win!

I attach `strace` to the child process like this:

`$ sudo strace -p 9412 2> strace_out`

and then go back to my `ssh` login and type in my password
(‘magicpassword’).

When I look in the `strace_out` that gets created, I can see the
password!

```
read(6, "\v\0\0\0\rmagicpassword", 18)  = 18
socket(PF_FILE, SOCK_DGRAM|SOCK_CLOEXEC, 0) = 4
connect(4, {sa_family=AF_FILE, path="/dev/log"}, 110) = 0
sendto(4, "<38>Feb 17 11:32:35 pam_fingerpr"..., 68, MSG_NOSIGNAL, NULL, 0) = 68
sendto(4, "<38>Feb 17 11:32:35 pam_fingerpr"..., 121, MSG_NOSIGNAL, NULL, 0) = 121
```

This is pretty nuts! When I think of the damage you can do as root, I
usually think of things like reading sensitive files. And when I wrote
a rootkit, I learned that you can do all kinds of crazy things by
inserting a malicious module into the kernel. (like hiding files and
processes and making every song on your computer be by Rick Astley)

But you can also spy on running processes and learn basically anything
you want around them! So if the NSA has root on your server, it can
easily find out everyone’s password who logs in via SSH. Whoa.
