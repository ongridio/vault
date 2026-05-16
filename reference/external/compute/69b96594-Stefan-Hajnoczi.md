---
title: Stefan Hajnoczi
source: http://blog.vmsplice.net/
kind: external
domain: compute
author: Stefanha
original_date: 2026-01-10
fetched_at: 2026-05-16
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.vmsplice.net](http://blog.vmsplice.net/)
> 作者：Stefanha
> 原始日期：2026-01-10
> 抓取日期：2026-05-16

# Stefan Hajnoczi

This is the fourth post in a series about building a virtio-serial device in Verilog for an FPGA development board. This time we'll look at processing the virtio-serial device's transmit and receive virtqueues.

### Series table of contents

- Part 1: Overview
- Part 2 - MMIO registers, DMA, and interrupts
- Part 3 - virtio-serial device design
- Part 4 - Virtqueue processing (you are here)
- Part 5 - UART receiver and transmitter
- Part 6 - Writing the RISC-V firmware

The code is available at https://gitlab.com/stefanha/virtio-serial-fpga.

The virtio-serial device has a pair of virtqueues that allow the driver to transmit and receive data. The driver enqueues empty buffers onto the receiveq (virtqueue 0) and the device fills them with received data. The driver enqueues buffers containing data onto the transmitq (virtqueue 1) and the device sends them.

This logic is split into two modules: virtqueue_reader for the transmitq and virtqueue_writer for the receiveq. The interface of virtqueue_reader looks like this:

/* Stream data from a virtqueue without framing */ module virtqueue_reader ( input clk, input resetn, /* Number of elements in descriptor table */ input [15:0] queue_size, /* Lower 32-bits of Virtqueue Descriptor Area address */ input [31:0] queue_desc_low, /* Lower 32-bits of Virtqueue Driver Area address */ input [31:0] queue_driver_low, /* Lower 32-bits of Virtqueue Device Area address */ input [31:0] queue_device_low, input queue_notify, /* kick */ input phase, output reg [31:0] data = 0, output reg [2:0] data_len = 0, output ready, /* For DMA */ output reg ram_valid = 0, input ram_ready, output reg [3:0] ram_wstrb = 0, output reg [21:0] ram_addr = 0, output reg [31:0] ram_wdata = 0, input [31:0] ram_rdata );

If you are familiar with the VIRTIO specification you might recognize queue_size, queue_desc_low, queue_driver_low, queue_device_low, and queue_notify since they are values provided by the VIRTIO MMIO Transport. The driver configures them with the memory addresses of the virtqueue data structures in RAM. The device will DMA to access those data structures. The driver can kick the device to indicate that new buffers have been enqueued using queue_notify.

The reader interface consists of phase, data, data_len, and ready and this is what the rdwr_stream module needs to use virtqueue_reader as a data source. rdwr_stream will keep reading the next byte(s) by flipping the phase bit and waiting for ready to be asserted by the device. Note that the device can provide up to 4 bytes at a time through the 32-bit data register and data_len allows the device to indicate how much data was read.

Finally, the DMA interface is how virtqueue_reader initiates RAM accesses so it can fetch the virtqueue data structures that the driver has configured.

## The state machine

Virtqueue processing consists of multiple steps and cannot be completed within a single clock cycle. Therefore the processing is decomposed into a state machine where each step consists of a DMA transfer or waiting for an event. Here are the states:

`define STATE_WAIT_PHASE 0 /* waiting for phase bit to flip */ `define STATE_READ_AVAIL_IDX 1 /* waiting for avail.idx read */ `define STATE_WAIT_NOTIFY 2 /* waiting for queue notify (kick) */ `define STATE_READ_DESCRIPTOR_ADDR_LOW 3 /* waiting for descriptor read */ `define STATE_READ_DESCRIPTOR_LEN 4 `define STATE_READ_DESCRIPTOR_FLAGS_NEXT 5 `define STATE_READ_BUFFER 6 /* waiting for data buffer read */ `define STATE_WRITE_USED_ELEM_ID 7 /* waiting for used element write */ `define STATE_WRITE_USED_ELEM_LEN 8 `define STATE_WRITE_USED_FLAGS_IDX 9 /* waiting for used.flags/used.idx write */ `define STATE_READ_AVAIL_RING_ENTRY 10 /* waiting for avail element read */

The device starts up in STATE_WAIT_PHASE because it is waiting to be asked to read the first byte(s). As soon as rdwr_stream flips the phase bit, virtqueue_reader must check the virtqueue to see if any data buffers are available.

I won't describe all the details of virtqueue processing, but here is a summary of the steps involved. See the VIRTIO specification or the code for the details.

- Read the avail.idx field from RAM in case the driver has enqueued more buffers.
- Read the avail.ring[i] entry from RAM to fetch the descriptor table index of the next available buffer.
- Read the descriptor from RAM to find out the buffer address and length.
- Repeatedly read bytes from the buffer until the current descriptor is empty. If the descriptor is chained, read the next descriptor from RAM and repeat.
- If the chain is finished, check avail.idx again in case there are more buffers available.

After a buffer has been fully consumed, there are also several steps to fill out a used descriptor and increment the used.idx field so that the driver is aware that the buffer is done.

There are two wait states when the device stops until it there is more work to do. First, rdwr_stream will stop asking to read more data if the writer is too slow. This flow control ensures that data is not dropped due to a slow writer. This is STATE_WAIT_PHASE. Second, if the device wants to read but the virtqueue is empty, then it has to wait until queue_notify goes high. This is STATE_WAIT_NOTIFY.

The virtqueue_writer module is similar to virtqueue_reader but it fills in the buffers with data instead of consuming them.

A quick side note about memory alignment: the memory interface is 32-bit aligned, so it is only possible to read an entire 32-bit value from memory at multiples of 4 bytes. On a fancier CPU with a cache the unit would be a cache line (e.g. 128 bytes). When the data structures being DMAed are not aligned it becomes tedious to handle the shifting and masking, especially when reading data from a source and writing it to a destination. Life is much simpler when everything is aligned, because data can be trivially read or written in a single access without any special logic to adjust the data to fit the cache line size.

## Conclusion

The virtqueue_reader and virtqueue_writer modules use DMA to read or write data from/to RAM buffers provided by the driver running on the PicoRV32 RISC-V CPU inside the FPGA. They are state machines that run through a sequence of DMA transfers and provide the reader/writer interfaces that the rdwr_module uses to transfer data. In the next post we will look at the UART receiver and transmitter.