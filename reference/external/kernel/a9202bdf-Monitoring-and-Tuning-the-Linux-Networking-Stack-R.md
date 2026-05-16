---
title: Monitoring and Tuning the Linux Networking Stack: Receiving Data | Packagecloud Blog
source: https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#napi-and-napi_schedule
kind: external
domain: kernel
author: Joe Damato
original_date: 2016-06-21
fetched_at: 2026-05-16
bookmark_title: Monitoring and Tuning the Linux Networking Stack: Receiving Data - Packagecloud Blog
tags: [external, kernel]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.packagecloud.io](https://blog.packagecloud.io/eng/2016/06/22/monitoring-tuning-linux-networking-stack-receiving-data/#napi-and-napi_schedule)
> 作者：Joe Damato
> 原始日期：2016-06-21
> 抓取日期：2026-05-16

# Monitoring and Tuning the Linux Networking Stack: Receiving Data | Packagecloud Blog

## TL;DR

This blog post explains how computers running the Linux kernel receive packets, as well as how to monitor and tune each component of the networking stack as packets flow from the network toward userland programs.

**UPDATE** We’ve released the counterpart to this post: Monitoring and Tuning the Linux Networking Stack: Sending Data.

**UPDATE** Take a look at the Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data, which adds some diagrams for the information presented below.

It is impossible to tune or monitor the Linux networking stack without reading the source code of the kernel and having a deep understanding of what exactly is happening.

This blog post will hopefully serve as a reference to anyone looking to do this.


## Special thanks

Special thanks to the folks at Private Internet Access who hired us to research this information in conjunction with other network research and who have graciously allowed us to build upon the research and publish this information.

The information presented here builds upon the work done for Private Internet Access, which was originally published as a 5 part series starting here.


## General advice on monitoring and tuning the Linux networking stack

**UPDATE** We’ve released the counterpart to this post: Monitoring and Tuning the Linux Networking Stack: Sending Data.

**UPDATE** Take a look at the Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data, which adds some diagrams for the information presented below.

The networking stack is complex and there is no one size fits all solution. If the performance and health of your networking is critical to you or your business, you will have no choice but to invest a considerable amount of time, effort, and money into understanding how the various parts of the system interact.

Ideally, you should consider measuring packet drops at each layer of the network stack. That way you can determine and narrow down which component needs to be tuned.

This is where, I think, many operators go off track: the assumption is made that a set of sysctl settings or `/proc`

values can simply be reused wholesale. In some cases, perhaps, but it turns out that the entire system is so nuanced and intertwined that if you desire to have meaningful monitoring or tuning, you must strive to understand how the system functions at a deep level. Otherwise, you can simply use the default settings, which should be good enough until further optimization (and the required investment to deduce those settings) is necessary.

Many of the example settings provided in this blog post are used solely for illustrative purposes and are not a recommendation for or against a certain configuration or default setting. Before adjusting any setting, you should develop a frame of reference around what you need to be monitoring to notice a meaningful change.

Adjusting networking settings while connected to the machine over a network is dangerous; you could very easily lock yourself out or completely take out your networking. Do not adjust these settings on production machines; instead make adjustments on new machines and rotate them into production, if possible.


## Overview

For reference, you may want to have a copy of the device data sheet handy. This post will examine the Intel I350 Ethernet controller, controlled by the `igb`

device driver. You can find that data sheet (warning: LARGE PDF) here for your reference.

The high level path a packet takes from arrival to socket receive buffer is as follows:

- Driver is loaded and initialized.
- Packet arrives at the NIC from the network.
- Packet is copied (via DMA) to a ring buffer in kernel memory.
- Hardware interrupt is generated to let the system know a packet is in memory.
- Driver calls into NAPI to start a poll loop if one was not running already.
`ksoftirqd`

processes run on each CPU on the system. They are registered at boot time. The`ksoftirqd`

processes pull packets off the ring buffer by calling the NAPI`poll`

function that the device driver registered during initialization.- Memory regions in the ring buffer that have had network data written to them are unmapped.
- Data that was DMA’d into memory is passed up the networking layer as an ‘skb’ for more processing.
- Incoming network data frames are distributed among multiple CPUs if packet steering is enabled or if the NIC has multiple receive queues.
- Network data frames are handed to the protocol layers from the queues.
- Protocol layers process data.
- Data is added to receive buffers attached to sockets by protocol layers.

This entire flow will be examined in detail in the following sections.

The protocol layers examined below are the IP and UDP protocol layers. Much of the information presented will serve as a reference for other protocol layers, as well.


## Detailed Look

**UPDATE** We’ve released the counterpart to this post: Monitoring and Tuning the Linux Networking Stack: Sending Data.

**UPDATE** Take a look at the Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data, which adds some diagrams for the information presented below.

This blog post will be examining the Linux kernel version 3.13.0 with links to code on GitHub and code snippets throughout this post.

Understanding exactly how packets are received in the Linux kernel is very involved. We’ll need to closely examine and understand how a network driver works, so that parts of the network stack later are more clear.

This blog post will look at the `igb`

network driver. This driver is used for a relatively common server NIC, the Intel Ethernet Controller I350. So, let’s start by understanding how the `igb`

network driver works.


### Network Device Driver

#### Initialization

A driver registers an initialization function which is called by the kernel when the driver is loaded. This function is registered by using the `module_init`

macro.

The `igb`

initialization function (`igb_init_module`

) and its registration with `module_init`

can be found in drivers/net/ethernet/intel/igb/igb_main.c.

Both are fairly straightforward:

The bulk of the work to initialize the device happens with the call to `pci_register_driver`

as we’ll see next.


##### PCI initialization

The Intel I350 network card is a PCI express device.

PCI devices identify themselves with a series of registers in the PCI Configuration Space.

When a device driver is compiled, a macro named `MODULE_DEVICE_TABLE`

(from `include/module.h`

) is used to export a table of PCI device IDs identifying devices that the device driver can control. The table is also registered as part of a structure, as we’ll see shortly.

The kernel uses this table to determine which device driver to load to control the device.

That’s how the OS can figure out which devices are connected to the system and which driver should be used to talk to the device.

This table and the PCI device IDs for the `igb`

driver can be found in `drivers/net/ethernet/intel/igb/igb_main.c`

and `drivers/net/ethernet/intel/igb/e1000_hw.h`

, respectively:

As seen in the previous section, `pci_register_driver`

is called in the driver’s initialization function.

This function registers a structure of pointers. Most of the pointers are function pointers, but the PCI device ID table is also registered. The kernel uses the functions registered by the driver to bring the PCI device up.

From `drivers/net/ethernet/intel/igb/igb_main.c`

:


##### PCI probe

Once a device has been identified by its PCI IDs, the kernel can then select the proper driver to use to control the device. Each PCI driver registers a probe function with the PCI system in the kernel. The kernel calls this function for devices which have not yet been claimed by a device driver. Once a device is claimed, other drivers will not be asked about the device. Most drivers have a lot of code that runs to get the device ready for use. The exact things done vary from driver to driver.

Some typical operations to perform include:

- Enabling the PCI device.
- Requesting memory ranges and IO ports.
- Setting the DMA mask.
- The ethtool (described more below) functions the driver supports are registered.
- Any watchdog tasks needed (for example, e1000e has a watchdog task to check if the hardware is hung).
- Other device specific stuff like workarounds or dealing with hardware specific quirks or similar.
- The creation, initialization, and registration of a
`struct net_device_ops`

structure. This structure contains function pointers to the various functions needed for opening the device, sending data to the network, setting the MAC address, and more. - The creation, initialization, and registration of a high level
`struct net_device`

which represents a network device.

Let’s take a quick look at some of these operations in the `igb`

driver in the function `igb_probe`

.


###### A peek into PCI initialization

The following code from the `igb_probe`

function does some basic PCI configuration. From drivers/net/ethernet/intel/igb/igb_main.c:

First, the device is initialized with `pci_enable_device_mem`

. This will wake up the device if it is suspended, enable memory resources, and more.

Next, the DMA mask will be set. This device can read and write to 64bit memory addresses, so `dma_set_mask_and_coherent`

is called with `DMA_BIT_MASK(64)`

.

Memory regions will be reserved with a call to `pci_request_selected_regions`

, PCI Express Advanced Error Reporting is enabled (if the PCI AER driver is loaded), DMA is enabled with a call to `pci_set_master`

, and the PCI configuration space is saved with a call to `pci_save_state`

.

Phew.

##### More Linux PCI driver information

Going into the full explanation of how PCI devices work is beyond the scope of this post, but this excellent talk, this wiki, and this text file from the linux kernel are excellent resources.


#### Network device initialization

The `igb_probe`

function does some important network device initialization. In addition to the PCI specific work, it will do more general networking and network device work:

- The
`struct net_device_ops`

is registered. `ethtool`

operations are registered.- The default MAC address is obtained from the NIC.
`net_device`

feature flags are set.- And lots more.

Let’s take a look at each of these as they will be interesting later.


`struct net_device_ops`


The `struct net_device_ops`

contains function pointers to lots of important operations that the network subsystem needs to control the device. We’ll be mentioning this structure many times throughout the rest of this post.

This `net_device_ops`

structure is attached to a `struct net_device`

in `igb_probe`

. From drivers/net/ethernet/intel/igb/igb_main.c)

And the functions that this `net_device_ops`

structure holds pointers to are set in the same file. From drivers/net/ethernet/intel/igb/igb_main.c:

As you can see, there are several interesting fields in this `struct`

like `ndo_open`

, `ndo_stop`

, `ndo_start_xmit`

, and `ndo_get_stats64`

which hold the addresses of functions implemented by the `igb`

driver.

We’ll be looking at some of these in more detail later.

`ethtool`

registration

`ethtool`

is a command line program you can use to get and set various driver and hardware options. You can install it on Ubuntu by running `apt-get install ethtool`

.

A common use of `ethtool`

is to gather detailed statistics from network devices. Other `ethtool`

settings of interest will be described later.

The `ethtool`

program talks to device drivers by using the `ioctl`

system call. The device drivers register a series of functions that run for the `ethtool`

operations and the kernel provides the glue.

When an `ioctl`

call is made from `ethtool`

, the kernel finds the `ethtool`

structure registered by the appropriate driver and executes the functions registered. The driver’s `ethtool`

function implementation can do anything from change a simple software flag in the driver to adjusting how the actual NIC hardware works by writing register values to the device.

The `igb`

driver registers its `ethtool`

operations in `igb_probe`

by calling `igb_set_ethtool_ops`

:

All of the `igb`

driver’s `ethtool`

code can be found in the file `drivers/net/ethernet/intel/igb/igb_ethtool.c`

along with the `igb_set_ethtool_ops`

function.

From `drivers/net/ethernet/intel/igb/igb_ethtool.c`

:

Above that, you can find the `igb_ethtool_ops`

structure with the `ethtool`

functions the `igb`

driver supports set to the appropriate fields.

From `drivers/net/ethernet/intel/igb/igb_ethtool.c`

:

It is up to the individual drivers to determine which `ethtool`

functions are relevant and which should be implemented. Not all drivers implement all `ethtool`

functions, unfortunately.

One interesting `ethtool`

function is `get_ethtool_stats`

, which (if implemented) produces detailed statistics counters that are tracked either in software in the driver or via the device itself.

The monitoring section below will show how to use `ethtool`

to access these detailed statistics.


##### IRQs

When a data frame is written to RAM via DMA, how does the NIC tell the rest of the system that data is ready to be processed?

Traditionally, a NIC would generate an interrupt request (IRQ) indicating data had arrived. There are three common types of IRQs: MSI-X, MSI, and legacy IRQs. These will be touched upon shortly. A device generating an IRQ when data has been written to RAM via DMA is simple enough, but if large numbers of data frames arrive this can lead to a large number of IRQs being generated. The more IRQs that are generated, the less CPU time is available for higher level tasks like user processes.

The New Api (NAPI) was created as a mechanism for reducing the number of IRQs generated by network devices on packet arrival. While NAPI reduces the number of IRQs, it cannot eliminate them completely.

We’ll see why that is, exactly, in later sections.


##### NAPI

NAPI differs from the legacy method of harvesting data in several important ways. NAPI allows a device driver to register a `poll`

function that the NAPI subsystem will call to harvest data frames.

The intended use of NAPI in network device drivers is as follows:

- NAPI is enabled by the driver, but is in the off position initially.
- A packet arrives and is DMA’d to memory by the NIC.
- An IRQ is generated by the NIC which triggers the IRQ handler in the driver.
- The driver wakes up the NAPI subsystem using a softirq (more on these later). This will begin harvesting packets by calling the driver’s registered
`poll`

function in a separate thread of execution. - The driver should disable further IRQs from the NIC. This is done to allow the NAPI subsystem to process packets without interruption from the device.
- Once there is no more work to do, the NAPI subsystem is disabled and IRQs from the device are re-enabled.
- The process starts back at step 2.

This method of gathering data frames has reduced overhead compared to the legacy method because many data frames can be consumed at a time without having to deal with processing each of them one IRQ at a time.

The device driver implements a `poll`

function and registers it with NAPI by calling `netif_napi_add`

. When registering a NAPI `poll`

function with `netif_napi_add`

, the driver will also specify the `weight`

. Most of the drivers hardcode a value of `64`

. This value and its meaning will be described in more detail below.

Typically, drivers register their NAPI `poll`

functions during driver initialization.


##### NAPI initialization in the `igb`

driver

The `igb`

driver does this via a long call chain:

`igb_probe`

calls`igb_sw_init`

.`igb_sw_init`

calls`igb_init_interrupt_scheme`

.`igb_init_interrupt_scheme`

calls`igb_alloc_q_vectors`

.`igb_alloc_q_vectors`

calls`igb_alloc_q_vector`

.`igb_alloc_q_vector`

calls`netif_napi_add`

.

This call trace results in a few high level things happening:

- If MSI-X is supported, it will be enabled with a call to
`pci_enable_msix`

. - Various settings are computed and initialized; most notably the number of transmit and receive queues that the device and driver will use for sending and receiving packets.
`igb_alloc_q_vector`

is called once for every transmit and receive queue that will be created.- Each call to
`igb_alloc_q_vector`

calls`netif_napi_add`

to register a`poll`

function for that queue and an instance of`struct napi_struct`

that will be passed to`poll`

when called to harvest packets.

Let’s take a look at `igb_alloc_q_vector`

to see how the `poll`

callback and its private data are registered.

From drivers/net/ethernet/intel/igb/igb_main.c:

The above code is allocation memory for a receive queue and registering the function `igb_poll`

with the NAPI subsystem. It provides a reference to the `struct napi_struct`

associated with this newly created RX queue (`&q_vector->napi`

above). This will be passed into `igb_poll`

when called by the NAPI subsystem when it comes time to harvest packets from this RX queue.

This will be important later when we examine the flow of data from drivers up the network stack.


#### Bringing a network device up

Recall the `net_device_ops`

structure we saw earlier which registered a set of functions for bringing the network device up, transmitting packets, setting the MAC address, etc.

When a network device is brought up (for example, with `ifconfig eth0 up`

), the function attached to the `ndo_open`

field of the `net_device_ops`

structure is called.

The `ndo_open`

function will typically do things like:

- Allocate RX and TX queue memory
- Enable NAPI
- Register an interrupt handler
- Enable hardware interrupts
- And more.

In the case of the `igb`

driver, the function attached to the `ndo_open`

field of the `net_device_ops`

structure is called `igb_open`

.


##### Preparing to receive data from the network

Most NICs you’ll find today will use DMA to write data directly into RAM where the OS can retrieve the data for processing. The data structure most NICs use for this purpose resembles a queue built on circular buffer (or a ring buffer).

In order to do this, the device driver must work with the OS to reserve a region of memory that the NIC hardware can use. Once this region is reserved, the hardware is informed of its location and incoming data will be written to RAM where it will later be picked up and processed by the networking subsystem.

This seems simple enough, but what if the packet rate was high enough that a single CPU was not able to properly process all incoming packets? The data structure is built on a fixed length region of memory, so incoming packets would be dropped.

This is where something known as known as Receive Side Scaling (RSS) or multiqueue can help.

Some devices have the ability to write incoming packets to several different regions of RAM simultaneously; each region is a separate queue. This allows the OS to use multiple CPUs to process incoming data in parallel, starting at the hardware level. This feature is not supported by all NICs.

The Intel I350 NIC does support multiple queues. We can see evidence of this in the `igb`

driver. One of the first things the `igb`

driver does when it is brought up is call a function named `igb_setup_all_rx_resources`

. This function calls another function, `igb_setup_rx_resources`

, once for each RX queue to arrange for DMA-able memory where the device will write incoming data.

If you are curious how exactly this works, please see the Linux kernel’s DMA API HOWTO.

It turns out the number and size of the RX queues can be tuned by using `ethtool`

. Tuning these values can have a noticeable impact on the number of frames which are processed vs the number of frames which are dropped.

The NIC uses a hash function on the packet header fields (like source, destination, port, etc) to determine which RX queue the data should be directed to.

Some NICs let you adjust the weight of the RX queues, so you can send more traffic to specific queues.

Fewer NICs let you adjust this hash function itself. If you can adjust the hash function, you can send certain flows to specific RX queues for processing or even drop the packets at the hardware level, if desired.

We’ll take a look at how to tune these settings shortly.


##### Enable NAPI

When a network device is brought up, a driver will usually enable NAPI.

We saw earlier how drivers register `poll`

functions with NAPI, but NAPI is not usually enabled until the device is brought up.

Enabling NAPI is relatively straight forward. A call to `napi_enable`

will flip a bit in the `struct napi_struct`

to indicate that it is now enabled. As mentioned above, while NAPI will be enabled it will be in the off position.

In the case of the `igb`

driver, NAPI is enabled for each `q_vector`

that was initialized when the driver was loaded or when the queue count or size are changed with `ethtool`

.

From drivers/net/ethernet/intel/igb/igb_main.c:

##### Register an interrupt handler

After enabling NAPI, the next step is to register an interrupt handler. There are different methods a device can use to signal an interrupt: MSI-X, MSI, and legacy interrupts. As such, the code differs from device to device depending on what the supported interrupt methods are for a particular piece of hardware.

The driver must determine which method is supported by the device and register the appropriate handler function that will execute when the interrupt is received.

Some drivers, like the `igb`

driver, will try to register an interrupt handler with each method, falling back to the next untested method on failure.

MSI-X interrupts are the preferred method, especially for NICs that support multiple RX queues. This is because each RX queue can have its own hardware interrupt assigned, which can then be handled by a specific CPU (with `irqbalance`

or by modifying `/proc/irq/IRQ_NUMBER/smp_affinity`

). As we’ll see shortly, the CPU that handles the interrupt will be the CPU that processes the packet. In this way, arriving packets can be processed by separate CPUs from the hardware interrupt level up through the networking stack.

If MSI-X is unavailable, MSI still presents advantages over legacy interrupts and will be used by the driver if the device supports it. Read this useful wiki page for more information about MSI and MSI-X.

In the `igb`

driver, the functions `igb_msix_ring`

, `igb_intr_msi`

, `igb_intr`

are the interrupt handler methods for the MSI-X, MSI, and legacy interrupt modes, respectively.

You can find the code in the driver which attempts each interrupt method in drivers/net/ethernet/intel/igb/igb_main.c:

As you can see in the abbreviated code above, the driver first attempts to set an MSI-X interrupt handler with `igb_request_msix`

, falling back to MSI on failure. Next, `request_irq`

is used to register `igb_intr_msi`

, the MSI interrupt handler. If this fails, the driver falls back to legacy interrupts. `request_irq`

is used again to register the legacy interrupt handler `igb_intr`

.

And this is how the `igb`

driver registers a function that will be executed when the NIC raises an interrupt signaling that data has arrived and is ready for processing.


##### Enable Interrupts

At this point, almost everything is setup. The only thing left is to enable interrupts from the NIC and wait for data to arrive. Enabling interrupts is hardware specific, but the `igb`

driver does this in `__igb_open`

by calling a helper function named `igb_irq_enable`

.

Interrupts are enabled for this device by writing to registers:

##### The network device is now up

Drivers may do a few more things like start timers, work queues, or other hardware-specific setup. Once that is completed. the network device is up and ready for use.

Let’s take a look at monitoring and tuning settings for network device drivers.


#### Monitoring network devices

There are several different ways to monitor your network devices offering different levels of granularity and complexity. Let’s start with most granular and move to least granular.

##### Using `ethtool -S`


You can install `ethtool`

on an Ubuntu system by running: `sudo apt-get install ethtool`

.

Once it is installed, you can access the statistics by passing the `-S`

flag along with the name of the network device you want statistics about.

Monitor detailed NIC device statistics (e.g., packet drops) with `ethtool -S`.

```
$ sudo ethtool -S eth0
NIC statistics:
rx_packets: 597028087
tx_packets: 5924278060
rx_bytes: 112643393747
tx_bytes: 990080156714
rx_broadcast: 96
tx_broadcast: 116
rx_multicast: 20294528
....
```


Monitoring this data can be difficult. It is easy to obtain, but there is no standardization of the field values. Different drivers, or even differen

[... 内容超长，已截断；完整原文见 source URL ...]
