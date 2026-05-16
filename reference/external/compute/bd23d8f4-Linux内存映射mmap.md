---
title: Linux内存映射（mmap）
source: https://www.cnblogs.com/lknlfy/archive/2012/04/27/2473804.html
kind: external
domain: compute
author: Lknlfy
original_date: 2012-04-27
fetched_at: 2026-05-16
bookmark_title: Linux内存映射（mmap） - lknlfy - 博客园
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](https://www.cnblogs.com/lknlfy/archive/2012/04/27/2473804.html)
> 作者：Lknlfy
> 原始日期：2012-04-27
> 抓取日期：2026-05-16

# Linux内存映射（mmap）

# Linux内存映射（mmap）

**一. 概述**

内存映射，简而言之就是将用户空间的一段内存区域映射到内核空间，映射成功后，用户对这段内存区域的修改可以直接反映到内核空间，相反，内核空间对这段区域的修改也直接反映用户空间。那么对于内核空间<---->用户空间两者之间需要大量数据传输等操作的话效率是非常高的。

首先，驱动程序先分配好一段内存，接着用户进程通过库函数mmap()来告诉内核要将多大的内存映射到内核空间，内核经过一系列函数调用后调用对应的驱动程序的file_operation中的mmap函数，在该函数中调用remap_pfn_range()来建立映射关系。直白一点就是：驱动程序在mmap()中利用remap_pfn_range()函数将内核空间的一段内存与用户空间的一段内存建立映射关系。

用户空间mmap()函数：

void *mmap(void *start, size_t length, int prot, int flags,int fd, off_t offset)

start：用户进程中要映射的某段内存区域的起始地址，通常为NULL（由内核来指定）

length：要映射的内存区域的大小

prot：期望的内存保护标志

flags：指定映射对象的类型

fd：文件描述符（由open函数返回）

**offset**：要映射的用户空间的内存区域在内核空间中已经分配好的的内存区域中的偏移。大小为PAGE_SIZE的整数倍


**二. 实现**

首先在驱动程序分配一页大小的内存，然后用户进程通过mmap()将用户空间中大小也为一页的内存映射到内核空间这页内存上。映射完成后，驱动程序往这段内存写10个字节数据，用户进程将这些数据显示出来。

驱动程序：

1 #include <linux/miscdevice.h> 2 #include <linux/delay.h> 3 #include <linux/kernel.h> 4 #include <linux/module.h> 5 #include <linux/init.h> 6 #include <linux/mm.h> 7 #include <linux/fs.h> 8 #include <linux/types.h> 9 #include <linux/delay.h> 10 #include <linux/moduleparam.h> 11 #include <linux/slab.h> 12 #include <linux/errno.h> 13 #include <linux/ioctl.h> 14 #include <linux/cdev.h> 15 #include <linux/string.h> 16 #include <linux/list.h> 17 #include <linux/pci.h> 18 #include <linux/gpio.h> 19 20 21 #define DEVICE_NAME "mymap" 22 23 24 static unsigned char array[10]={0,1,2,3,4,5,6,7,8,9}; 25 static unsigned char *buffer; 26 27 28 static int my_open(struct inode *inode, struct file *file) 29 { 30 return 0; 31 } 32 33 34 static int my_map(struct file *filp, struct vm_area_struct *vma) 35 { 36 unsigned long page; 37 unsigned char i; 38 unsigned long start = (unsigned long)vma->vm_start; 39 //unsigned long end = (unsigned long)vma->vm_end; 40 unsigned long size = (unsigned long)(vma->vm_end - vma->vm_start); 41 42 //得到物理地址 43 page = virt_to_phys(buffer); 44 //将用户空间的一个vma虚拟内存区映射到以page开始的一段连续物理页面上 45 if(remap_pfn_range(vma,start,page>>PAGE_SHIFT,size,PAGE_SHARED))//第三个参数是页帧号，由物理地址右移PAGE_SHIFT得到 46 return -1; 47 48 //往该内存写10字节数据 49 for(i=0;i<10;i++) 50 buffer[i] = array[i]; 51 52 return 0; 53 } 54 55 56 static struct file_operations dev_fops = { 57 .owner = THIS_MODULE, 58 .open = my_open, 59 .mmap = my_map, 60 }; 61 62 static struct miscdevice misc = { 63 .minor = MISC_DYNAMIC_MINOR, 64 .name = DEVICE_NAME, 65 .fops = &dev_fops, 66 }; 67 68 69 static int __init dev_init(void) 70 { 71 int ret; 72 73 //注册混杂设备 74 ret = misc_register(&misc); 75 //内存分配 76 buffer = (unsigned char *)kmalloc(PAGE_SIZE,GFP_KERNEL); 77 //将该段内存设置为保留 78 SetPageReserved(virt_to_page(buffer)); 79 80 return ret; 81 } 82 83 84 static void __exit dev_exit(void) 85 { 86 //注销设备 87 misc_deregister(&misc); 88 //清除保留 89 ClearPageReserved(virt_to_page(buffer)); 90 //释放内存 91 kfree(buffer); 92 } 93 94 95 module_init(dev_init); 96 module_exit(dev_exit); 97 MODULE_LICENSE("GPL"); 98 MODULE_AUTHOR("LKN@SCUT");

应用程序：

1 #include <unistd.h> 2 #include <stdio.h> 3 #include <stdlib.h> 4 #include <string.h> 5 #include <fcntl.h> 6 #include <linux/fb.h> 7 #include <sys/mman.h> 8 #include <sys/ioctl.h> 9 10 #define PAGE_SIZE 4096 11 12 13 int main(int argc , char *argv[]) 14 { 15 int fd; 16 int i; 17 unsigned char *p_map; 18 19 //打开设备 20 fd = open("/dev/mymap",O_RDWR); 21 if(fd < 0) 22 { 23 printf("open fail\n"); 24 exit(1); 25 } 26 27 //内存映射 28 p_map = (unsigned char *)mmap(0, PAGE_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED,fd, 0); 29 if(p_map == MAP_FAILED) 30 { 31 printf("mmap fail\n"); 32 goto here; 33 } 34 35 //打印映射后的内存中的前10个字节内容 36 for(i=0;i<10;i++) 37 printf("%d\n",p_map[i]); 38 39 40 here: 41 munmap(p_map, PAGE_SIZE); 42 return 0; 43 }

先加载驱动后执行应用程序，用户空间打印如下：