---
title: HBase-配置说明
source: https://www.cnblogs.com/admln/p/HBase-configuratioin.html
kind: external
domain: compute
author: Daem
original_date: 2014-11-11
fetched_at: 2026-05-16
bookmark_title: HBase-配置说明 - Daem0n - 博客园
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/admln/p/HBase-configuratioin.html)
> 作者：Daem
> 原始日期：2014-11-11
> 抓取日期：2026-05-16

# HBase-配置说明

# HBase-配置说明

转载自：http://www.aboutyun.com/thread-7914-1-1.html

**hbase.rootdir**

这个目录是region server的共享目录，用来持久化Hbase。URL需要是'完全正确'的，还要包含文件系统的scheme。例如，要表示hdfs中的 '/hbase'目录，namenode 运行在namenode.example.org的9090端口。则需要设置为hdfs://namenode.example.org:9000 /hbase。默认情况下Hbase是写到/tmp的。不改这个配置，数据会在重启的时候丢失。

默认: file:///tmp/hbase-${user.name}/hbase

**hbase.master.port**

Hbase的Master的端口.

默认: 60000

**hbase.cluster.distributed**

Hbase的运行模式。false是单机模式，true是分布式模式。若为false,Hbase和Zookeeper会运行在同一个JVM里面。

默认: false

**hbase.tmp.dir**

本地文件系统的临时文件夹。可以修改到一个更为持久的目录上。(/tmp会在重启时清楚)

默认: /tmp/hbase-${user.name}

**hbase.master.info.port**

HBase Master web 界面端口. 设置为-1 意味着你不想让他运行。

默认: 60010

**hbase.master.info.bindAddress**

HBase Master web 界面绑定的地址

默认: 0.0.0.0

**hbase.client.write.buffer**

HTable 客户端的写缓冲的默认大小。这个值越大，需要消耗的内存越大。因为缓冲在客户端和服务端都有实例，所以需要消耗客户端和服务端两个地方的内存。得到的好处 是，可以减少RPC的次数。

默认: 2097152

**hbase.regionserver.port**

HBase RegionServer绑定的端口

默认: 60020

**hbase.regionserver.info.port**

HBase RegionServer web 界面绑定的端口 设置为 -1 意味这你不想与运行 RegionServer 界面.

默认: 60030

**hbase.regionserver.info.port.auto**

Master或RegionServer是否要动态搜一个可以用的端口来绑定界面。当hbase.regionserver.info.port已经被占用的时候，可以搜一个空闲的端口绑定。这个功能在测试的时候很有用。默认关闭。

默认: false

**hbase.regionserver.info.bindAddress**

HBase RegionServer web 界面的IP地址

默认: 0.0.0.0

**hbase.regionserver.class**

RegionServer 使用的接口。客户端打开代理来连接region server的时候会使用到。

默认: org.apache.hadoop.hbase.ipc.HRegionInterface

**hbase.client.pause**

通常的客户端暂停时间。最多的用法是客户端在重试前的等待时间。比如失败的get操作和region查询操作等都很可能用到。

默认: 1000

**hbase.client.retries.number**

最大重试次数。例如 region查询，Get操作，Update操作等等都可能发生错误，需要重试。这是最大重试错误的值。

默认: 10

只列出了我觉得可能现在会用到的。zookeeper的一些配置也很重要，但是实际生产中它都是单独配置管理监控的。