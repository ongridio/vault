---
title: A detailed view of the vhost user protocol and its implementation in OVS DPDK, qemu and virtio-net - Red Hat Customer Portal
source: https://access.redhat.com/solutions/3394851
kind: external
domain: network
original_date: 2024-06-14
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[access.redhat.com](https://access.redhat.com/solutions/3394851)
> 原始日期：2024-06-14
> 抓取日期：2026-05-16

# A detailed view of the vhost user protocol and its implementation in OVS DPDK, qemu and virtio-net - Red Hat Customer Portal

# A detailed view of the vhost user protocol and its implementation in OVS DPDK, qemu and virtio-net

## Environment

Red Hat OpenStack Platform 10

Open vSwitch 2.6.1

## Issue

A detailed view of the vhost user protocol and its implementation in OVS DPDK, qemu and virtio-net

## Resolution


Disclaimer:Links contained herein to external website(s) are provided for convenience only. Red Hat has not reviewed the links and is not responsible for the content or its availability. The inclusion of any link to an external website does not imply endorsement by Red Hat of the website or their entities, products or services. You agree that Red Hat is not responsible or liable for any loss or expenses that may result due to your use of (or reliance on) the external site or content.

## A detailed view of the vhost user protocol and its implementation in OVS DPDK, qemu and virtio-net

## Overview: How OVS DPDK and qemu communicate via the vhost user protocol

The vhost user protocol consists of a control path and a data path.

-
All control information is exchanged via a Unix socket. This includes information for exchanging memory mappings for direct memory access, as well as kicking / interrupting the other side if data is put into the virtio queue. The Unix socket, in neutron, is named

`vhuxxxxxxxx-xx`

. -
The actual dataplane is implemented via direct memory access. The virtio-net driver within the guest allocates part of the instance memory for the virtio queue. The structure of this queue is standardized in the virtio standard. Qemu shares this memory section’s address with OVS DPDK over the control channel. DPDK itself then maps the same standardized virtio queue structure onto this memory section and can thus directly read from and write to the virtio queue within the instance’s hugepage memory. This direct memory access is one of the reasons why both OVS DPDK and qemu need to use hugepage memory. If qemu is otherwise set up correctly, but lacks configuration for huge page memory, then OVS DPDK will not be able to access qemu’s memory and hence no packets can be exchanged. Users will notice this if they forget to request instance hugepages via nova’s metadata.


When OVS DPDK transmits towards the instance, these packets will show up within OVS DPDK’s statistics as Tx on port `vhuxxxxxxxx-xx`

. Within the instance, these packets show up as Rx.

When the instance transmits packets to OVS DPDK, then on the instance, these packets show up as Tx, and on OVS DPDK’s `vhuxxxxxxxx-xx`

port, they show up as Rx.

Note that the instance does not have “hardware” counters. `ethtool -s`

is not implemented. All low level counters do only show up within OVS (`ovs-vsctl list get interfave vhuxxxxxxxx-xx statistics`

) and report OVS DPDK’s perspective.

Although packets can be directly transmitted via shared memory, either side needs a means to tell the opposite side that a packet was copied into the virtio queue. This happens by `kicking`

the other side over the control plane which is implemented with the vhost user socket `vhuxxxxxxxx-xx`

. Kicking the other side comes at a cost. Firstly, a system call is needed to write to the socket. Secondly, an interrupt will have to be processed by the other side. Hence both sender and receiver spend costly extra time within the control channel.

In order to avoid costly `kicks`

via the control plane, both Open vSwitch and qemu can set specific flags to signal to the other side that they do not wish to receive an interrupt. However, they can only do so if they either temporarily or constantly poll the virtio queue.

For instance network performance this means that the optimal means of packet processing is DPDK within the instance itself. While Linux kernel networking (`NAPI`

) uses a mix of interrupt and poll mode processing, it is still exposed to a high number of interrupts. OVS DPDK sends packets towards the instance at very high rates. At the same time, the RX and TX buffers of qemu’s virtio queue are limited to a default of 256 and a maximum of 1024 entries. As a consequence, the instance itself needs to process packets very quickly. This is ideally achieved by constantly polling with a DPDK PMD on the instance’s interface.

## The vhost user protocol

https://github.com/qemu/qemu/blob/master/docs/interop/vhost-user.txt

```
Vhost-user Protocol
===================
Copyright (c) 2014 Virtual Open Systems Sarl.
This work is licensed under the terms of the GNU GPL, version 2 or later.
See the COPYING file in the top-level directory.
===================
This protocol is aiming to complement the ioctl interface used to control the
vhost implementation in the Linux kernel. It implements the control plane needed
to establish virtqueue sharing with a user space process on the same host. It
uses communication over a Unix domain socket to share file descriptors in the
ancillary data of the message.
The protocol defines 2 sides of the communication, master and slave. Master is
the application that shares its virtqueues, in our case QEMU. Slave is the
consumer of the virtqueues.
In the current implementation QEMU is the Master, and the Slave is intended to
be a software Ethernet switch running in user space, such as Snabbswitch.
Master and slave can be either a client (i.e. connecting) or server (listening)
in the socket communication.
```


vhost user has since 2 sides:

-
**Master**- qemu -
**Slave**- Open vSwitch or any other software switch

vhost user can run in 2 modes:

-
**vhostuser-client**- qemu is the server, the software switch is the client -
**vhostuser**- the software switch is the server, qemu is the client

vhost user is based on the vhost architecture and implements all features in user space.

When a qemu instance boots, it will allocate all of the instance memory as shared hugepages. The OS' virtio paravirtualized driver will reserve part of this hugepage memory for holding the virtio ring buffer. This allows OVS DPDK to directly read from and write into the instance's virtio ring. Both OVS DPDK and qemu can directly exchange packets across this reserved memory section.

"The user space application will receive file descriptors for the pre-allocated shared guest RAM. It will directly access the related vrings in the guest's memory space" (http://www.virtualopensystems.com/en/solutions/guides/snabbswitch-qemu/).

For example, look at the following VM, mode vhostuser:

```
qemu 528828 0.1 0.0 2920084 34188 ? Sl Mar28 1:45 /usr/libexec/qemu-kvm -name guest=instance-00000028,debug-threads=on -S -object secret,id=masterKey0,format=raw,file=/var/lib/libvirt/qemu/domain-58-instance-00000028/master-key.aes -machine pc-i440fx-rhel7.4.0,accel=kvm,usb=off,dump-guest-core=off -cpu Skylake-Client,ss=on,hypervisor=on,tsc_adjust=on,pdpe1gb=on,mpx=off,xsavec=off,xgetbv1=off -m 2048 -realtime mlock=off -smp 8,sockets=4,cores=1,threads=2 -object memory-backend-file,id=ram-node0,prealloc=yes,mem-path=/dev/hugepages/libvirt/qemu/58-instance-00000028,share=yes,size=1073741824,host-nodes=0,policy=bind -numa node,nodeid=0,cpus=0-3,memdev=ram-node0 -object memory-backend-file,id=ram-node1,prealloc=yes,mem-path=/dev/hugepages/libvirt/qemu/58-instance-00000028,share=yes,size=1073741824,host-nodes=1,policy=bind -numa node,nodeid=1,cpus=4-7,memdev=ram-node1 -uuid 48888226-7b6b-415c-bcf7-b278ba0bca62 -smbios type=1,manufacturer=Red Hat,product=OpenStack Compute,version=14.1.0-3.el7ost,serial=3d5e138a-8193-41e4-ac95-de9bfc1a3ef1,uuid=48888226-7b6b-415c-bcf7-b278ba0bca62,family=Virtual Machine -no-user-config -nodefaults -chardev socket,id=charmonitor,path=/var/lib/libvirt/qemu/domain-58-instance-00000028/monitor.sock,server,nowait -mon chardev=charmonitor,id=monitor,mode=control -rtc base=utc,driftfix=slew -global kvm-pit.lost_tick_policy=delay -no-hpet -no-shutdown -boot strict=on -device piix3-usb-uhci,id=usb,bus=pci.0,addr=0x1.0x2 -drive file=/var/lib/nova/instances/48888226-7b6b-415c-bcf7-b278ba0bca62/disk,format=qcow2,if=none,id=drive-virtio-disk0,cache=none -device virtio-blk-pci,scsi=off,bus=pci.0,addr=0x4,drive=drive-virtio-disk0,id=virtio-disk0,bootindex=1 -chardev socket,id=charnet0,path=/var/run/openvswitch/vhuc26fd3c6-4b -netdev vhost-user,chardev=charnet0,queues=8,id=hostnet0 -device virtio-net-pci,mq=on,vectors=18,netdev=hostnet0,id=net0,mac=fa:16:3e:52:30:73,bus=pci.0,addr=0x3 -add-fd set=0,fd=33 -chardev file,id=charserial0,path=/dev/fdset/0,append=on -device isa-serial,chardev=charserial0,id=serial0 -chardev pty,id=charserial1 -device isa-serial,chardev=charserial1,id=serial1 -device usb-tablet,id=input0,bus=usb.0,port=1 -vnc 172.16.2.10:1 -k en-us -device cirrus-vga,id=video0,bus=pci.0,addr=0x2 -device virtio-balloon-pci,id=balloon0,bus=pci.0,addr=0x5 -msg timestamp=on
```


Qemu is instructed to allocate memory from the huge page pool and to make it shared memory (`share=yes`

):

```
-object memory-backend-file,id=ram-node0,prealloc=yes,mem-path=/dev/hugepages/libvirt/qemu/58-instance-00000028,share=yes,size=1073741824,host-nodes=0,policy=bind -numa node,nodeid=0,cpus=0-3,memdev=ram-node0 -object memory-backend-file,id=ram-node1,prealloc=yes,mem-path=/dev/hugepages/libvirt/qemu/58-instance-00000028,share=yes,size=1073741824,host-nodes=1,policy=bind
```


Simply copying packets into the other party's buffer is not enough, however. Additionally, vhost user uses a Unix domain socket (`vhu[a-f0-9-]`

) for communication between the vswitch and qemu, both during initialization and to `kick`

the other side when packets were copied into the virtio ring in shared memory. Interaction hence consists of a control path (`vhu`

socket) for setup and notification and a datapath (direct memory access) for moving the actual payload.

```
For the described Virtio mechanism to work, we need a setup interface to initialize the shared memory regions and exchange the event file descriptors. A Unix domain socket implements an API which allows us to do that. This straightforward socket interface can be used to initialize the userspace Virtio transport (vhost-user), in particular:
* Vrings are determined at initialization and are placed in shared memory between the two processed.
* For Virtio events (Vring kicks) we shall use eventfds that map to Vring events. This allows us compatibility with the QEMU/KVM implementation described in the next chapter, since KVM allows us to match events coming from virtio_pci in the guest with eventfds (ioeventfd and irqfd).
Sharing file descriptors between two processes differs than sharing them between a process and the kernel. One needs to use sendmsg over a Unix domain socket with SCM_RIGHTS set.
```


(http://www.virtualopensystems.com/en/solutions/guides/snabbswitch-qemu/)

In vhostuser mode, OVS creates the `vhu`

socket and qemu connects to it. in vhostuser client mode, qemu creates the `vhu`

socket and OVS connects to it.

In the above example instance with vhostuser mode, qemu is instructed to connect a `netdev`

of type `vhost-user`

to `/var/run/openvswitch/vhuc26fd3c6-4b`

:

```
-chardev socket,id=charnet0,path=/var/run/openvswitch/vhuc26fd3c6-4b -netdev vhost-user,chardev=charnet0,queues=8,id=hostnet0 -device virtio-net-pci,mq=on,vectors=18,netdev=hostnet0,id=net0,mac=fa:16:3e:52:30:73,bus=pci.0,addr=0x3
```


`lsof`

reveals that the socket is created by OVS:

```
[root@overcloud-compute-0 ~]# lsof -nn | grep vhuc26fd3c6-4b | awk '{print $1}' | uniq
ovs-vswit
vfio-sync
eal-intr-
lcore-sla
dpdk_watc
vhost_thr
ct_clean3
urcu4
handler12
handler13
handler14
handler15
revalidat
pmd189
pmd182
pmd187
pmd184
pmd185
pmd186
pmd183
pmd188
```


When a packet is copied into the virtio ring in shared memory by one of the participants, the other side either

-
does currently (e.g. Linux kernel's NAPI) or constantly (e.g. DPDK's PMD) poll the queue in case of which it will pick up new packets without further notice.

-
does not poll the queue and must be notified of the arrival of packets.


For the second case, the instance can be `kicked`

via the separate control path across the `vhu`

socket. The control path implements interrupts in user space by exchanging `eventfd`

objects. Note that writing to the socket requires system calls and will cause the PMDs to spend time in kernel space. The VM can switch off the control path by setting the `VRING_AVAIL_F_NO_INTERRUPT`

flag. Otherwise, Open vSwitch will `kick`

(interrupt) the VM whenever it puts new packets into the virtio ring.

Further details can be found in the following blog post: http://blog.vmsplice.net/2011/09/qemu-internals-vhost-architecture.html

```
Vhost as a userspace interface
One surprising aspect of the vhost architecture is that it is not tied to KVM in any way. Vhost is a userspace interface and has no dependency on the KVM kernel module. This means other userspace code, like libpcap, could in theory use vhost devices if they find them convenient high-performance I/O interfaces.
When a guest kicks the host because it has placed buffers onto a virtqueue, there needs to be a way to signal the vhost worker thread that there is work to do. Since vhost does not depend on the KVM kernel module they cannot communicate directly. Instead vhost instances are set up with an eventfd file descriptor which the vhost worker thread watches for activity. The KVM kernel module has a feature known as ioeventfd for taking an eventfd and hooking it up to a particular guest I/O exit. QEMU userspace registers an ioeventfd for the VIRTIO_PCI_QUEUE_NOTIFY hardware register access which kicks the virtqueue. This is how the vhost worker thread gets notified by the KVM kernel module when the guest kicks the virtqueue.
On the return trip from the vhost worker thread to interrupting the guest a similar approach is used. Vhost takes a "call" file descriptor which it will write to in order to kick the guest. The KVM kernel module has a feature called irqfd which allows an eventfd to trigger guest interrupts. QEMU userspace registers an irqfd for the virtio PCI device interrupt and hands it to the vhost instance. This is how the vhost worker thread can interrupt the guest.
In the end the vhost instance only knows about the guest memory mapping, a kick eventfd, and a call eventfd.
Where to find out more
Here are the main points to begin exploring the code:
drivers/vhost/vhost.c - common vhost driver code
drivers/vhost/net.c - vhost-net driver
virt/kvm/eventfd.c - ioeventfd and irqfd
The QEMU userspace code shows how to initialize the vhost instance:
hw/vhost.c - common vhost initialization code
hw/vhost_net.c - vhost-net initialization
```


## The datapath - direct memory access

### How memory is mapped for the virtq

The virtio standard defines exactly what a virtq should look like.

```
2.4 Virtqueues
The mechanism for bulk data transport on virtio devices is pretentiously called a virtqueue. Each device can have zero or more virtqueues. Each queue has a 16-bit queue size parameter, which sets the number of entries and implies the total size of the queue.
Each virtqueue consists of three parts:
Descriptor Table
Available Ring
Used Ring
```


http://docs.oasis-open.org/virtio/virtio/v1.0/virtio-v1.0.html

The standard exactly defines the structure of the descriptor table, available ring and used ring. For example, for the available ring:

```
2.4.6 The Virtqueue Available Ring
struct virtq_avail {
#define VIRTQ_AVAIL_F_NO_INTERRUPT 1
le16 flags;
le16 idx;
le16 ring[ /* Queue Size */ ];
le16 used_event; /* Only if VIRTIO_F_EVENT_IDX */
};
The driver uses the available ring to offer buffers to the device: each ring entry refers to the head of a descriptor chain. It is only written by the driver and read by the device.
idx field indicates where the driver would put the next descriptor entry in the ring (modulo the queue size). This starts at 0, and increases. Note: The legacy [Virtio PCI Draft] referred to this structure as vring_avail, and the constant as VRING_AVAIL_F_NO_INTERRUPT, but the layout and value were identical.
```


http://docs.oasis-open.org/virtio/virtio/v1.0/virtio-v1.0.html

In order to make direct memory access possible, DPDK implements the above standard.

`dpdk-stable-16.11.4/drivers/net/virtio/virtio_ring.h`


```
48 /* The Host uses this in used->flags to advise the Guest: don't kick me
49 * when you add a buffer. It's unreliable, so it's simply an
50 * optimization. Guest will still kick if it's out of buffers. */
51 #define VRING_USED_F_NO_NOTIFY 1
52 /* The Guest uses this in avail->flags to advise the Host: don't
53 * interrupt me when you consume a buffer. It's unreliable, so it's
54 * simply an optimization. */
55 #define VRING_AVAIL_F_NO_INTERRUPT 1
56
57 /* VirtIO ring descriptors: 16 bytes.
58 * These can chain together via "next". */
59 struct vring_desc {
60 uint64_t addr; /* Address (guest-physical). */
61 uint32_t len; /* Length. */
62 uint16_t flags; /* The flags as indicated above. */
63 uint16_t next; /* We chain unused descriptors via this. */
64 };
65
66 struct vring_avail {
67 uint16_t flags;
68 uint16_t idx;
69 uint16_t ring[0];
70 };
71
72 /* id is a 16bit index. uint32_t is used here for ids for padding reasons. */
73 struct vring_used_elem {
74 /* Index of start of used descriptor chain. */
75 uint32_t id;
76 /* Total length of the descriptor chain which was written to. */
77 uint32_t len;
78 };
79
80 struct vring_used {
81 uint16_t flags;
82 volatile uint16_t idx;
83 struct vring_used_elem ring[0];
84 };
85
86 struct vring {
87 unsigned int num;
88 struct vring_desc *desc;
89 struct vring_avail *avail;
90 struct vring_used *used;
91 };
```


`dpdk-stable-16.11.4/lib/librte_vhost/vhost.h`


```
81 struct vhost_virtqueue {
82 struct vring_desc *desc;
83 struct vring_avail *avail;
84 struct vring_used *used;
85 uint32_t size;
86
87 uint16_t last_avail_idx;
88 uint16_t last_used_idx;
89 #define VIRTIO_INVALID_EVENTFD (-1)
90 #define VIRTIO_UNINITIALIZED_EVENTFD (-2)
91
92 /* Backend value to determine if device should started/stopped */
93 int backend;
94 /* Used to notify the guest (trigger interrupt) */
95 int callfd;
96 /* Currently unused as polling mode is enabled */
97 int kickfd;
98 int enabled;
99
100 /* Physical address of used ring, for logging */
101 uint64_t log_guest_addr;
102
103 uint16_t nr_zmbuf;
104 uint16_t zmbuf_size;
105 uint16_t last_zmbuf_idx;
106 struct zcopy_mbuf *zmbufs;
107 struct zcopy_mbuf_list zmbuf_list;
108
109 struct vring_used_elem *shadow_used_ring;
110 uint16_t shadow_used_idx;
111 } __rte_cache_aligned;
```


Once the memory mapping is done, DPDK can directly act on and manipulate the same structures as virtio-net within the guest's shared memory.

## The control path - Unix sockets

### qemu and DPDK message exchange over vhost user socket

DPDK and qemu communicate via the standardized vhost-user protocol.

The message types are:

`dpdk-stable-16.11.4/lib/librte_vhost/vhost_user.h`


```
54 typedef enum VhostUserRequest {
55 VHOST_USER_NONE = 0,
56 VHOST_USER_GET_FEATURES = 1,
57 VHOST_USER_SET_FEATURES = 2,
58 VHOST_USER_SET_OWNER = 3,
59 VHOST_USER_RESET_OWNER = 4,
60 VHOST_USER_SET_MEM_TABLE = 5,
61 VHOST_USER_SET_LOG_BASE = 6,
62 VHOST_USER_SET_LOG_FD = 7,
63 VHOST_USER_SET_VRING_NUM = 8,
64 VHOST_USER_SET_VRING_ADDR = 9,
65 VHOST_USER_SET_VRING_BASE = 10,
66 VHOST_USER_GET_VRING_BASE = 11,
67 VHOST_USER_SET_VRING_KICK = 12,
68 VHOST_USER_SET_VRING_CALL = 13,
69 VHOST_USER_SET_VRING_ERR = 14,
70 VHOST_USER_GET_PROTOCOL_FEATURES = 15,
71 VHOST_USER_SET_PROTOCOL_FEATURES = 16,
72 VHOST_USER_GET_QUEUE_NUM = 17,
73 VHOST_USER_SET_VRING_ENABLE = 18,
74 VHOST_USER_SEND_RARP = 19,
75 VHOST_USER_MAX
76 } VhostUserRequest;
```


Further details about the message types can be found in qemu's source code in: https://github.com/qemu/qemu/blob/master/docs/interop/vhost-user.txt

DPDK processes incoming messages with …

`dpdk-stable-16.11.4/lib/librte_vhost/vhost_user.c`


```
920 int
921 vhost_user_msg_handler(int vid, int fd)
922 {
```


… which uses:

`dpdk-stable-16.11.4/lib/librte_vhost/vhost_user.c`

:

```
872 /* return bytes# of read on success or negative val on failure. */
873 static int
874 read_vhost_message(int sockfd, struct VhostUserMsg *msg)
875 {
```


DPDK writes outgoing messages with:

`dpdk-stable-16.11.4/lib/librte_vhost/vhost_user.c`


```
902 static int
903 send_vhost_message(int sockfd, struct VhostUserMsg *msg)
904 {
```


qemu has an equivalent method for receiving:

`qemu-2.9.0/contrib/libvhost-user/libvhost-user.c`


```
746 static bool
747 vu_process_message(VuDev *dev, VhostUserMsg *vmsg)
```


And qemu obviously also has an equivalent method for sending:

`qemu-2.9.0/hw/virtio/vhost-user.c`


```
198 /* most non-init callers ignore the error */
199 static int vhost_user_write(struct vhost_dev *dev, VhostUserMsg *msg,
200 int *fds, int fd_num)
201 {
```


### How DPDK registers the Unix socket and uses it for message exchange

neutron instructs Open vSwitch to create a port with name vhuxxxxxxxx-xx. Within OVS, this name is saved in the `netdev`

structure as `netdev->name`

.

When it creates the vhost user port, Open vSwitch instructs DPDK to register a new vhost-user socket. The socket's path is set as `dev->vhost_id`

which is a concatenation of `vhost_sock_dir`

and `netdev->name`

.

OVS can request to create the socket in vhost user client mode by passing the `RTE_VHOST_USER_CLIENT`

flag.

OVS' `netdev_dpdk_vhost_construct`

method calls DPDK's `rte_vhost_driver_register`

method, which in turn executes `vhost_user_create_server`

or `vhost_user_create_client`

. By default, vhost user server mode is used, or if `RTE_VHOST_USER_CLIENT`

is set, vhost user client mode.

Overview of the involved methods:

```
OVS
netdev_dpdk_vhost_construct
(struct netdev *netdev)
|
|
DPDK V
rte_vhost_driver_register
(const char *path, uint64_t flags)
|
-----------------------------------------------
| |
V |
vhost_user_create_server |
(struct vhost_user_socket *vsocket) |
| |
V V
vhost_user_server_new_connection vhost_user_create_client vhost_user_client_reconnect
(int fd, void *dat, int *remove __rte_unused) (struct vhost_user_socket *vsocket) (void *arg __rte_unused)
| | |
V V V
--------------------------------------------------------------------------------------------------
|
V
vhost_user_add_connection
(int fd, struct vhost_user_socket *vsocket)
|
V
vhost_user_read_cb
(int connfd, void *dat, int *remove)
|
V
vhost_user_msg_handler
```


`netdev_dpdk_vhost_construct`

is in `openvswitch-2.6.1/lib/netdev-dpdk.c:`


```
886 static int
887 netdev_dpdk_vhost_construct(struct netdev *netdev)
888 {
889 struct netdev_dpdk *dev = netdev_dpdk_cast(netdev);
890 const char *name = netdev->name;
891 int err;
892
893 /* 'name' is appended to 'vhost_sock_dir' and used to create a socket in
894 * the file system. '/' or '\' would traverse directories, so they're not
895 * acceptable in 'name'. */
896 if (strchr(name, '/') || strchr(name, '\\')) {
897 VLOG_ERR("\"%s\" is not a valid name for a vhost-user port. "
898 "A valid name must not include '/' or '\\'",
899 name);
900 return EINVAL;
901 }
902
903 if (rte_eal_init_ret) {
904 return rte_eal_init_ret;
905 }
906
907 ovs_mutex_lock(&dpdk_mutex);
908 /* Take the name of the vhost-user port and append it to the location where
909 * the socket is to be created, then register the socket.
910 */
911 snprintf(dev->vhost_id, sizeof dev->vhost_id, "%s/%s",
912 vhost_sock_dir, name);
913
914 dev->vhost_driver_flags &= ~RTE_VHOST_USER_CLIENT;
915 err = rte_vhost_driver_register(dev->vhost_id, dev->vhost_driver_flags);
916 if (err) {
917 VLOG_ERR("vhost-user socket device setup failure for socket %s\n",
918 dev->vhost_id);
919 } else {
920 fatal_signal_add_file_to_unlink(dev->vhost_id);
921 VLOG_INFO("Socket %s created for vhost-user port %s\n",
922 dev->vhost_id, name);
923 }
924 err = netdev_dpdk_init(netdev, -1, DPDK_DEV_VHOST);
925
926 ovs_mutex_unlock(&dpdk_mutex);
927 return err;
928 }
```


`netdev_dpdk_vhost_construct`

calls `rte_vhost_driver_register`

. All of the following code is in `dpdk-stable-16.11.4/lib/librte_vhost/socket.c`

:

```
494 /*
495 * Register a new vhost-user socket; here we could act as server
496 * (the default case), or client (when RTE_VHOST_USER_CLIENT) flag
497 * is set.
498 */
499 int
500 rte_vhost_driver_register(const char *path, uint64_t flags)
501 {
(...)
525 if ((flags & RTE_VHOST_USER_CLIENT) != 0) {
526 vsocket->reconnect = !(flags & RTE_VHOST_USER_NO_RECONNECT);
527 if (vsocket->reconnect && reconn_tid == 0) {
528 if (vhost_user_reconnect_init() < 0) {
529 free(vsocket->path);
530 free(vsocket);
531 goto out;
532 }
533 }
534 ret = vhost_user_create_client(vsocket);
535 } else {
536 vsocket->is_server = true;
537 ret = vhost_user_create_server(vsocket);
538 }
```


`vhost_user_create_server`

calls `vhost_user_server_new_connection`

:

```
304 static int
305 vhost_user_create_server(struct vhost_user_socket *vsocket)
306 {
307 int fd;
308 int ret;
309 struct sockaddr_u

[... 内容超长，已截断；完整原文见 source URL ...]
