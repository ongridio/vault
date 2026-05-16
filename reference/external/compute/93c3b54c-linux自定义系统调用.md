---
title: linux自定义系统调用
source: http://www.cnblogs.com/chengxuyuancc/p/3489306.html
kind: external
domain: compute
author: 在于思考
original_date: 2013-12-24
fetched_at: 2026-05-16
bookmark_title: linux自定义系统调用 - 在于思考 - 博客园
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.cnblogs.com](http://www.cnblogs.com/chengxuyuancc/p/3489306.html)
> 作者：在于思考
> 原始日期：2013-12-24
> 抓取日期：2026-05-16

# linux自定义系统调用

# linux自定义系统调用

# 1 Linux3.10.21内核系统调用设置

以前看的内核版本时2.6.11的，里面的系统调用设置一目了然啊！在文件entry.S中直接定义了sys_call_table表，并在这个文件中用各个系统调用函数的地址初始化了这个表。今天看了下3.10.21的内核，想自己添加个系统调用到这个内核里面，没想到看了半天还是在云里雾里中，只知道sys_call_table表的定义在文件/usr/src/linux-3.10/arch/x86/kernel/syscall_32.c中，当具体怎么初始化这个表的还是不知道。找了半天都没找到表sys_call_table初始化的地方，后来回到syscall_32.c文件中才发现在它定义的地方就初始化了，下面看下定义的代码：

|
const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = { /* * Smells like a compiler bug -- it doesn't work * when the & below is removed. */ [0 ... __NR_syscall_max] = &sys_ni_syscall, #include <asm/syscalls_32.h> }; |

看大花括号里面第一句话应该是将sys_call_table表项都默认设置为sys_ni_syscal函数的地址，然后第二句是包含了进文件asm/syscalls_32.h，不过我找了半天都没找到这个文件。后来在网上看了别人的一篇博客才知道这个文件是在编译内核的时候动态生成的。编译内核的时候，当执行到文件/usr/src/linux-3.10.21/arch/x86/syscalls/Makefile时，该文件会执行/usr/src/linux-3.10.21/arch/x86/syscalls/目录下的shell脚本syscalltdr.sh，该脚本将同目录下的syscall_32.tbl文件作为输入，然后生成文件/usr/src/linux-3.10.21/arch/x86/include/generated/asm/syscalls_32.h，这个文件正是sys_call_table定义中包含的文件。

现在我们来看下shell脚本syscalltdr.sh到底做了什么，这个文件的内容如下：

|
#!/bin/sh
in="$1" out="$2"
grep '^[0-9]' "$in" | sort -n | ( while read nr abi name entry compat; do abi=`echo "$abi" | tr '[a-z]' '[A-Z]'` if [ -n "$compat" ]; then echo "__SYSCALL_${abi}($nr, $entry, $compat)" elif [ -n "$entry" ]; then echo "__SYSCALL_${abi}($nr, $entry, $entry)" fi done ) > "$out" |

其中in和out分别代表的就是syscall_32.tbl和syscalls_32.h文件的路径。脚本大概意思就是读取syscall_32.tbl内容，然后构造语句__SYSCALL_${abi}($nr, $entry, $entry)"。

文件syscall_32.tbl的部分内容：

|
# # 32-bit system call numbers and entry vectors # # The format is: # <number> <abi> <name> <entry point> <compat entry point> # # The abi is always "i386" for this file. # 0 i386 restart_syscall sys_restart_syscall 1 i386 exit sys_exit 2 i386 fork sys_fork stub32_fork 3 i386 read sys_read 4 i386 write sys_write 5 i386 open sys_open compat_sys_open 6 i386 close sys_close 7 i386 waitpid sys_waitpid sys32_waitpid |

文件syscalls_32.h的部分内容：

|
__SYSCALL_I386(0, sys_restart_syscall, sys_restart_syscall) __SYSCALL_I386(1, sys_exit, sys_exit) __SYSCALL_I386(2, sys_fork, stub32_fork) __SYSCALL_I386(3, sys_read, sys_read) __SYSCALL_I386(4, sys_write, sys_write) __SYSCALL_I386(5, sys_open, compat_sys_open) __SYSCALL_I386(6, sys_close, sys_close) __SYSCALL_I386(7, sys_waitpid, sys32_waitpid) __SYSCALL_I386(8, sys_creat, sys_creat) __SYSCALL_I386(9, sys_link, sys_link) __SYSCALL_I386(10, sys_unlink, sys_unlink) __SYSCALL_I386(11, sys_execve, stub32_execve) __SYSCALL_I386(12, sys_chdir, sys_chdir) __SYSCALL_I386(13, sys_time, compat_sys_time) __SYSCALL_I386(14, sys_mknod, sys_mknod) __SYSCALL_I386(15, sys_chmod, sys_chmod) __SYSCALL_I386(16, sys_lchown16, sys_lchown16) __SYSCALL_I386(18, sys_stat, sys_stat) __SYSCALL_I386(19, sys_lseek, compat_sys_lseek) |

看了这两个文件的内容想必对于脚本文件yscalltdr.sh的功能一目了然了。到了这里我才正在理解了3.10.21的内核是怎么给表sys_call_table初始化的，可以推测宏__SYSCALL_I386应该是完成对表中的特定一项初始化。现在看下这个宏的实现，它和表sys_call_table的定义放在同一个文件中。

|
#define __SYSCALL_I386(nr, sym, compat) [nr] = sym, |

看到这里想必什么都明白了吧，syscalls_32.h中一连串的宏其实完成的就是用文件syscall_32.tbl中声明的系统调用来初始化表sys_call_table中的每一项。

现在想想，其实原理和以前2.6.11版本时一样的，只是多拐了一个弯。对gcc中居然可以这样初始化数组表示以前没见过啊！深感gcc的强大啊！

然后定义每个系统调用的调用号和上面初始化表sys_call_table原理是一样的，执行shell脚本并提取出syscall_32.tbl的系统调用号存放到另一个文件中。

# 2 添加自己的系统调用

1、在系统调用向量表里添加自定义的系统调用号，

在文件：/usr/src/linux-3.10/arch/x86/syscalls/syscall_32.tbl中加入自定义的系统调用号和函数名。


2、添加函数声明

在文件/usr/src/linux-3.10/arch/x86/include/asm/syscalls.h文件中加入sys_foo函数的声明。


3、添加函数的定义

在文件/usr/src/linux-3.10/kernel/sys.c文件中加入对sys_foo的定义。


4、编译和安装内核

直接敲下面的命令来编译和安装内核：

|
make -j10 bzImage make install |

# 3 测试自定义的系统调用

为了测试是否自己定义的系统调用能够正常的使用，写个测试程序是有必要的。代码如下：

|
#include<stdio.h>
#define __NR_foo 351
int main(void) { int rs;
rs = syscall(__NR_foo); if (!rs) printf("syscall success!\n"); return 0}
|

代码中用到了syscall函数，这个函数功能就是根据给定的系统调用号来调用系统调用。

编译运行上面的代码，用dmesg可以看到系统输出的信息：


出现这个画面，说明自定义的系统调用能正常工作了。