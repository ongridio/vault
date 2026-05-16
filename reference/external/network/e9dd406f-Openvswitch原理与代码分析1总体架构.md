---
title: Openvswitch原理与代码分析(1)：总体架构
source: https://www.cnblogs.com/popsuper1982/p/5848879.html
kind: external
domain: network
author: Popsuper
original_date: 2016-09-07
fetched_at: 2026-05-16
bookmark_title: Openvswitch原理与代码分析(1)：总体架构 - popsuper1982 - 博客园
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/popsuper1982/p/5848879.html)
> 作者：Popsuper
> 原始日期：2016-09-07
> 抓取日期：2026-05-16

# Openvswitch原理与代码分析(1)：总体架构

# Openvswitch原理与代码分析(1)：总体架构


# 一、Opevswitch总体架构


Openvswitch的架构网上有如下的图表示：







每个模块都有不同的功能

## ovs-vswitchd 为主要模块，实现交换机的守护进程daemon


在Openvswitch所在的服务器进行ps aux可以看到以下的进程

root 1008 0.1 0.8 242948 31712 ? S<Ll Aug06 32:17 |


注意这里ovs-vswitchd监听了一个本机的db.sock文件


## openvswitch.ko为Linux内核模块，支持数据流在内核的交换


我们使用lsmod列举加载到内核的模块：

~# lsmod | grep openvswitch openvswitch 66901 0 gre 13808 1 openvswitch vxlan 37619 1 openvswitch libcrc32c 12644 2 btrfs,openvswitch |


既有Openvswitch.ko，也有

## ovsdb-server 轻量级数据库服务器，保存配置信息，ovs-vswitchd通过这个数据库获取配置信息


通过ps aux可以看到如下进程

root 985 0.0 0.0 21172 2120 ? S< Aug06 1:20 |


可以看出，ovsdb-server将配置信息保存在conf.db中，并通过db.sock提供服务，ovs-vswitchd通过这个db.sock从这个进程读取配置信息。

/etc/openvswitch/conf.db是json格式的，可以通过命令ovsdb-client dump将数据库结构打印出来。


数据库结构包含如下的表格。




数据库结构如下：




通过ovs-vsctl创建的所有的网桥，网卡，都保存在数据库里面，ovs-vswitchd会根据数据库里面的配置创建真正的网桥，网卡。

ovs-dpctl 用来配置switch内核模块。

ovs-vsctl 查询和更新ovs-vswitchd的配置。

ovs-appctl 发送命令消息，运行相关daemon。

ovs-ofctl 查询和控制OpenFlow交换机和控制器。


# 二、Openvswitch的代码结构


Openvwitch进行数据流交换的主要逻辑都是在ovs-vswitchd和openvswitch.ko里面实现的。




ovs-vswitchd会从ovsdb-server读取配置，然后调用ofproto层进行虚拟网卡的创建或者流表的操作。

Ofproto是一个库，实现了软件的交换机和对流表的操作。

Netdev层抽象了连接到虚拟交换机上的网络设备。

Dpif层实现了对于流表的操作。


对于OVS来讲，有以下几种网卡类型

1). netdev: 通用网卡设备 eth0 veth

接收: 一个nedev在L2收到报文后回直接通过ovs接收函数处理，不会再走传统内核协议栈.

发送: ovs中的一条流指定从该netdev发出的时候就通过该网卡设备发送

2). internal: 一种虚拟网卡设备

接收: 当从系统发出的报文路由查找通过该设备发送的时候，就进入ovs接收处理函数

发送: ovs中的一条流制定从该internal设备发出的时候，该报文被重新注入内核协议栈

3). gre device: gre设备. 不管用户态创建多少个gre tunnel, 在内核态有且只有一个gre设备

接收: 当系统收到gre报文后，传递给L4层解析gre header, 然后传递给ovs接收处理函数

发送: ovs中的一条流制定从该gre设备发送, 报文会根据流表规则加上gre头以及外层包裹ip，查找路由发送




在如上的代码结构中，vswitchd中就是ovs-vswitchd的入口代码，ovsdb就是ovsdb-server的代码，ofproto即上述的中间抽象层，lib下面有netdev，dpif的实现，datapath里面就是内核模块openvswitch.ko的代码。


# 三、ovs-vswitchd和openvswitch.ko的交互方式netlink


datapath 运行在内核态，ovs-vswitchd 运行在用户态，两者通过netlink 通信。

netlink 是一种灵活和强大的进程间通信机制（socket），甚至可以沟通用户态和内核态。

netlink 是全双工的。作为socket，netlink 的地址族是AF_NETLINK（TCP/IP socket 的地址族是AF_INET）

目前有大量的通信场景应用了netlink，这些特定扩展和设计的netlink 通信bus，被定义为family。比如NETLINK_ROUTE、NETLINK_FIREWALL、NETLINK_ARPD 等。

因为大量的专用family 会占用了family id，而family id 数量自身有限（kernel 允许32个）；同时为了方便用户扩展使用，一个通用的netlink family 被定义出来，这就是generic netlink family。


要使用generic netlink，需要熟悉的数据结构包括genl_family、genl_ops 等。




下面写一个generic netlink的简单实例

定义family如下

- /* attributes */
- enum {
- DOC_EXMPL_A_UNSPEC,
- DOC_EXMPL_A_MSG,
- __DOC_EXMPL_A_MAX,
- };
- #define DOC_EXMPL_A_MAX (__DOC_EXMPL_A_MAX - 1)
- /* attribute policy */
- static struct nla_policy doc_exmpl_genl_policy[DOC_EXMPL_A_MAX + 1] = {
- [DOC_EXMPL_A_MSG] = { .type = NLA_NUL_STRING },
- };
- /* family definition */
- static struct genl_family doc_exmpl_gnl_family = {
- .id = GENL_ID_GENERATE,
- .hdrsize = 0,
- .name = "DOC_EXMPL",
- .version = 1,
- .maxattr = DOC_EXMPL_A_MAX,
- };
|


定义op如下

- /* handler */
- int doc_exmpl_echo(struct sk_buff *skb, struct genl_info *info)
- {
- /* message handling code goes here; return 0 on success, negative values on failure */
- }
- /* commands */
- enum {
- DOC_EXMPL_C_UNSPEC,
- DOC_EXMPL_C_ECHO,
- __DOC_EXMPL_C_MAX,
- };
- #define DOC_EXMPL_C_MAX (__DOC_EXMPL_C_MAX - 1)
- /* operation definition */
- struct genl_ops doc_exmpl_gnl_ops_echo = {
- .cmd = DOC_EXMPL_C_ECHO,
- .flags = 0,
- .policy = doc_exmpl_genl_policy,
- .doit = doc_exmpl_echo,
- .dumpit = NULL,
- };
|


注册family 到generic netlink 机制

- int rc;
- rc = genl_register_family(&doc_exmpl_gnl_family);
- if (rc != 0)
- goto failure;
|


将操作注册到family

- int rc;
- rc = genl_register_ops(&doc_exmpl_gnl_family, &doc_exmpl_gnl_ops_echo);
- if (rc != 0)
- goto failure;
|


Datapath是如何使用netlink的呢？


在dp_init()函数（datapath.c）中，调用dp_register_genl()完成对四种类型的family 以及相应操作的注册，包括datapath、vport、flow 和packet。

前三种family，都对应四种操作都包括NEW、DEL、GET、SET，而packet 的操作仅为EXECUTE。


对于flow这个family的定义如下：

- static const struct nla_policy flow_policy[OVS_FLOW_ATTR_MAX + 1] = {
- [OVS_FLOW_ATTR_KEY] = { .type = NLA_NESTED },
- [OVS_FLOW_ATTR_ACTIONS] = { .type = NLA_NESTED },
- [OVS_FLOW_ATTR_CLEAR] = { .type = NLA_FLAG },
- };
- static struct genl_family dp_flow_genl_family = {
- .id = GENL_ID_GENERATE,
- .hdrsize = sizeof(struct ovs_header),
- .name = OVS_FLOW_FAMILY,
- .version = OVS_FLOW_VERSION,
- .maxattr = OVS_FLOW_ATTR_MAX,
- SET_NETNSOK
- };
|


Flow相关的ops的定义如下：

- static struct genl_ops dp_flow_genl_ops[] = {
- {
- .cmd = OVS_FLOW_CMD_NEW,
- .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
- .policy = flow_policy,
- .doit = ovs_flow_cmd_new_or_set
- },
- {
- .cmd = OVS_FLOW_CMD_DEL,
- .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
- .policy = flow_policy,
- .doit = ovs_flow_cmd_del
- },
- {
- .cmd = OVS_FLOW_CMD_GET,
- .flags = 0, /* OK for unprivileged users. */
- .policy = flow_policy,
- .doit = ovs_flow_cmd_get,
- .dumpit = ovs_flow_cmd_dump
- },
- {
- .cmd = OVS_FLOW_CMD_SET,
- .flags = GENL_ADMIN_PERM, /* Requires CAP_NET_ADMIN privilege. */
- .policy = flow_policy,
- .doit = ovs_flow_cmd_new_or_set,
- },
- };
|


Ovs-vswitchd作为客户端如何使用netlink


Ovs-vswitchd 对于netlink 的实现，主要在lib/netlink-socket.c 文件中。

lib\dpif-provider.h定义了struct dpif_class {，包含一系列函数指针，例如open，close等。

真正的dpif_class有两种

一个是dpif-netdev.c中定义的const struct dpif_class dpif_netdev_class = {

- const struct dpif_class dpif_netdev_class = {
- "netdev",
- dpif_netdev_init,
- dpif_netdev_enumerate,
- dpif_netdev_port_open_type,
- dpif_netdev_open,
- dpif_netdev_close,
- dpif_netdev_destroy,
- dpif_netdev_run,
- dpif_netdev_wait,
- dpif_netdev_get_stats,
- dpif_netdev_port_add,
- dpif_netdev_port_del,
- dpif_netdev_port_query_by_number,
- dpif_netdev_port_query_by_name,
- NULL, /* port_get_pid */
- dpif_netdev_port_dump_start,
- dpif_netdev_port_dump_next,
- dpif_netdev_port_dump_done,
- dpif_netdev_port_poll,
- dpif_netdev_port_poll_wait,
- dpif_netdev_flow_flush,
- dpif_netdev_flow_dump_create,
- dpif_netdev_flow_dump_destroy,
- dpif_netdev_flow_dump_thread_create,
- dpif_netdev_flow_dump_thread_destroy,
- dpif_netdev_flow_dump_next,
- dpif_netdev_operate,
- NULL, /* recv_set */
- NULL, /* handlers_set */
- dpif_netdev_pmd_set,
- dpif_netdev_queue_to_priority,
- NULL, /* recv */
- NULL, /* recv_wait */
- NULL, /* recv_purge */
- dpif_netdev_register_dp_purge_cb,
- dpif_netdev_register_upcall_cb,
- dpif_netdev_enable_upcall,
- dpif_netdev_disable_upcall,
- dpif_netdev_get_datapath_version,
- NULL, /* ct_dump_start */
- NULL, /* ct_dump_next */
- NULL, /* ct_dump_done */
- NULL, /* ct_flush */
- };
|


一种是在dpif-netlink.c中，定义了const struct dpif_class dpif_netlink_class = {

- const struct dpif_class dpif_netlink_class = {
- "system",
- NULL, /* init */
- dpif_netlink_enumerate,
- NULL,
- dpif_netlink_open,
- dpif_netlink_close,
- dpif_netlink_destroy,
- dpif_netlink_run,
- NULL, /* wait */
- dpif_netlink_get_stats,
- dpif_netlink_port_add,
- dpif_netlink_port_del,
- dpif_netlink_port_query_by_number,
- dpif_netlink_port_query_by_name,
- dpif_netlink_port_get_pid,
- dpif_netlink_port_dump_start,
- dpif_netlink_port_dump_next,
- dpif_netlink_port_dump_done,
- dpif_netlink_port_poll,
- dpif_netlink_port_poll_wait,
- dpif_netlink_flow_flush,
- dpif_netlink_flow_dump_create,
- dpif_netlink_flow_dump_destroy,
- dpif_netlink_flow_dump_thread_create,
- dpif_netlink_flow_dump_thread_destroy,
- dpif_netlink_flow_dump_next,
- dpif_netlink_operate,
- dpif_netlink_recv_set,
- dpif_netlink_handlers_set,
- NULL, /* poll_thread_set */
- dpif_netlink_queue_to_priority,
- dpif_netlink_recv,
- dpif_netlink_recv_wait,
- dpif_netlink_recv_purge,
- NULL, /* register_dp_purge_cb */
- NULL, /* register_upcall_cb */
- NULL, /* enable_upcall */
- NULL, /* disable_upcall */
- dpif_netlink_get_datapath_version, /* get_datapath_version */
- #ifdef __linux__
- dpif_netlink_ct_dump_start,
- dpif_netlink_ct_dump_next,
- dpif_netlink_ct_dump_done,
- dpif_netlink_ct_flush,
- #else
- NULL, /* ct_dump_start */
- NULL, /* ct_dump_next */
- NULL, /* ct_dump_done */
- NULL, /* ct_flush */
- #endif
- };
|


datapath 中对netlink family 类型进行了注册，ovs-vswitchd 在使用这些netlink family 之前需要获取它们的信息，这一过程主要在lib/dpif-netlink.c 文件（以dpif_netlink_class 为例），dpif_netlink_init ()函数。

- static int
- dpif_netlink_init(void)
- {
- static struct ovsthread_once once = OVSTHREAD_ONCE_INITIALIZER;
- static int error;
- if (ovsthread_once_start(&once)) {
- error = nl_lookup_genl_family(OVS_DATAPATH_FAMILY,
- &ovs_datapath_family);
- if (error) {
- VLOG_ERR("Generic Netlink family '%s' does not exist. "
- "The Open vSwitch kernel module is probably not loaded.",
- OVS_DATAPATH_FAMILY);
- }
- if (!error) {
- error = nl_lookup_genl_family(OVS_VPORT_FAMILY, &ovs_vport_family);
- }
- if (!error) {
- error = nl_lookup_genl_family(OVS_FLOW_FAMILY, &ovs_flow_family);
- }
- if (!error) {
- error = nl_lookup_genl_family(OVS_PACKET_FAMILY,
- &ovs_packet_family);
- }
- if (!error) {
- error = nl_lookup_genl_mcgroup(OVS_VPORT_FAMILY, OVS_VPORT_MCGROUP,
- &ovs_vport_mcgroup);
- }
- ovsthread_once_done(&once);
- }
- return error;
- }
|


完成这些查找后，ovs-vswitchd 即可利用dpif 中的api，通过发出这些netlink 消息给datapath，实现对datapath 的操作。