---
title: GitHub - wangbojing/NtyTcp: 单线程用户态TCP/IP协议栈，epoll实现，包含服务器案例，并发测试案例
source: https://github.com/wangbojing/NtyTcp
kind: external
domain: network
author: Wangbojing
original_date: 2018-03-18
fetched_at: 2026-05-16
bookmark_title: wangbojing/NtyTcp: netmap, dpdk, pf_ring, Tcp Stack for Userspace
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[github.com](https://github.com/wangbojing/NtyTcp)
> 作者：Wangbojing
> 原始日期：2018-03-18
> 抓取日期：2026-05-16

# GitHub - wangbojing/NtyTcp: 单线程用户态TCP/IP协议栈，epoll实现，包含服务器案例，并发测试案例

```
$ git clone https://github.com/wangbojing/netmap.git
$ ./configure
$ make
$ sudo make install
```

1. problem : configure --> /bin/sh^M.
```
$ dos2unix configure
$ dos2unix ./LINUX/configure
```

2. problem : cannot stat 'bridge': No such or directory
```
$ make clean
$ cd build-apps/bridge
$ gcc -O2 -pipe -Werror -Wall -Wunused-function -I ../../sys -I ../../apps/include -Wextra ../../apps/bridge/bridge.c -lpthread -lrt -o bridge
$ sudo make && make install
```

netmap, dpdk, pf_ring, Tcp Stack for Userspace

compile:

```
$ sudo apt-get install libhugetlbfs-dev
$ make
```

update NtyTcp/include/nty_config.h

```
#define NTY_SELF_IP "192.168.0.106" //your ip
#define NTY_SELF_IP_HEX 0x6A00A8C0 //your ip hex.
#define NTY_SELF_MAC "00:0c:29:58:6f:f4" //your mac
```

block server run:

```
$ ./bin/nty_example_block_server
```

epoll server run:

```
$ ./bin/nty_example_epoll_rb_server
```

if you discover bug to sending email to wangbojing@0voice.com .

also, want to be an NtyTcper, so you can sent email to wangbojing@0voice.com .