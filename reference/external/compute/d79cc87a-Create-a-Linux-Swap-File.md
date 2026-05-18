---
title: Create a Linux Swap File
source: https://linuxize.com/post/create-a-linux-swap-file/
kind: external
domain: compute
original_date: 2019-02-19
fetched_at: 2026-05-16
bookmark_title: Create a Linux Swap File | Linuxize
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[linuxize.com](https://linuxize.com/post/create-a-linux-swap-file/)
> 原始日期：2019-02-19
> 抓取日期：2026-05-16

# Create a Linux Swap File

# Create a Linux Swap File

Swap is a space on a disk that is used when the amount of physical RAM is full. When a Linux system runs out of RAM, inactive pages are moved from the RAM to the swap space.

Swap space can take the form of either a dedicated swap partition or a swap file. In most cases, when running Linux on a virtual machine, a swap partition is not present, so the only option is to create a swap file.

Swap helps when you run out of RAM, but it is not a replacement for memory. Heavy swapping usually means your system is under memory pressure and will slow down.

## How to add a swap file

Follow these steps to add 1GB of swap to your server. If you want to add 2GB instead of 1GB, replace `1G`

with `2G`

. On small systems, 1-2GB is often enough; if you use hibernation, swap should be at least the size of RAM.

Create a file that will be used for swap:

Terminal`sudo fallocate -l 1G /swapfile`

If

`fallocate`

is not installed or if you get an error message saying`fallocate failed: Operation not supported`

, you can use the following command to create the swap file:Terminal`sudo dd if=/dev/zero of=/swapfile bs=1024 count=1048576`

To confirm the file size, run:

Terminal`ls -lh /swapfile`

Only the root user should be able to write and read the swap file. To set the correct permissions type:

Terminal`sudo chmod 600 /swapfile`

If you are using Btrfs, swap files must be created on a No_COW, non-compressed file, otherwise

`swapon`

can fail. Create the swap file inside a directory with`chattr +C`

and compression disabled. For example:Terminal`sudo mkdir -p /swap sudo chattr +C /swap`

Then create the swap file inside

`/swap`

(for example,`/swap/swapfile`

).Use the

`mkswap`

utility to set up the file as Linux swap area:Terminal`sudo mkswap /swapfile`

Enable the swap with the following command:

Terminal`sudo swapon /swapfile`

To make the change permanent, open the

`/etc/fstab`

file and append the following line:/etc/fstabini`/swapfile none swap defaults 0 0`

To verify that the swap is active, use either

`swapon`

or the`free`

command as shown below:Terminal`sudo swapon --show`

output`NAME TYPE SIZE USED PRIO /swapfile file 1024M 507.4M -1`

Terminal`sudo free -h`

output`total used free shared buff/cache available Mem: 488M 158M 83M 2.3M 246M 217M Swap: 1.0G 506M 517M`


## How to adjust the swappiness value

Swappiness is a Linux kernel property that defines how often the system will use the swap space. Swappiness can have a value between 0 and 100. A low value will make the kernel try to avoid swapping whenever possible, while a higher value will make the kernel use the swap space more aggressively.

The default swappiness value is 60. You can check the current swappiness value by typing the following command:

`cat /proc/sys/vm/swappiness`


`60`


While the swappiness value of 60 is OK for most Linux systems, for production servers, you may need to set a lower value.

For example, to set the swappiness value to 10, you would run the following `sysctl`

command:

`sudo sysctl vm.swappiness=10`


To make this parameter persistent across reboots, append the following line to the `/etc/sysctl.conf`

file:

`vm.swappiness=10`


The optimal swappiness value depends on your system workload and how the memory is being used.

## How to remove a swap file

If for any reason you want to deactivate and remove the swap file, follow these steps:

First, deactivate the swap by typing:

Terminal`sudo swapoff -v /swapfile`

Remove the swap file entry

`/swapfile none swap defaults 0 0`

from the`/etc/fstab`

file.Finally, delete the actual swap file using the

`rm`

command:Terminal`sudo rm /swapfile`


## FAQ

**How much swap space should I create?**

For systems with less than 2 GB of RAM, a swap size of twice the RAM is a common starting point. For systems with 2–8 GB of RAM, matching the RAM size is usually enough. For systems with more than 8 GB of RAM, 4–8 GB of swap is a reasonable ceiling unless you use hibernation, in which case swap must be at least the size of your RAM.

**How do I resize an existing swap file?**

Turn off the current swap, recreate the file at the new size, reformat it, and re-enable it:

```
sudo swapoff /swapfile
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```


**My system already has a zram device. Can I still add a swap file?**

Yes. Ubuntu 22.04 and later (and Debian 12 and later) enable a zram-based swap device by default. You can add a swap file alongside it using the same steps above. If you run `swapon --show`

and see a `/dev/zram`

entry instead of a file, that is expected. The new swap file will appear as an additional entry once enabled.

## Conclusion

For more on memory management, the `free`

and `vmstat`

commands give you a live view of swap usage. If your system is swapping heavily under normal load, adding more RAM is a more effective fix than increasing swap.

## Linuxize Weekly Newsletter

A quick weekly roundup of new tutorials, news, and tips.

## About the authors

Dejan Panovski

Dejan Panovski is the founder of Linuxize, an RHCSA-certified Linux system administrator and DevOps engineer based in Skopje, Macedonia. Author of 800+ Linux tutorials with 20+ years of experience turning complex Linux tasks into clear, reliable guides.

View author page