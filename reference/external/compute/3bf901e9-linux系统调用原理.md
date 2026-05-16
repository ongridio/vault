---
title: linux系统调用原理
source: https://blog.csdn.net/luckywang1103/article/details/51381317
kind: external
domain: compute
author: 成就一亿技术人
original_date: 2026-02-28
fetched_at: 2026-05-16
bookmark_title: linux系统调用原理 - CSDN博客
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.csdn.net](https://blog.csdn.net/luckywang1103/article/details/51381317)
> 作者：成就一亿技术人
> 原始日期：2026-02-28
> 抓取日期：2026-05-16

# linux系统调用原理

## x86架构

### trap_init

在系统启动的时候start_kernel会调用trap_init来初始化异常向量表

```
start_kernel
trap_init
set_system_trap_gate(SYSCALL_VECTOR, &system_call);
...
memcpy(&idt[entry], gate, sizeof(*gate));
```


设置0x80号软中断的服务程序为system_call, system_call是所有系统调用的总入口.

当进程执行到用户程序的系统调用命令时，实际上执行了由宏命令_syscallN()展开的函数。系统调用的参数由各通用寄存器传递，比如通过eax寄存器传递系统调用号和系统调用返回值，通过ebx/ecx/edx/esi/edi传递系统调用参数，然后执行INT 0x80，以内核态进入入口地址system_call。

### system_call

在arch/x86/kernel/entry_32.S文件中定义了system_call，在system_call里面调用了sys_call_table

```
ENTRY(system_call)
RING0_INT_FRAME # can't unwind into user space anyway
ASM_CLAC
pushl_cfi %eax # save orig_eax
SAVE_ALL
GET_THREAD_INFO(%ebp)
# system call tracing in operation / emulation
testl $_TIF_WORK_SYSCALL_ENTRY,TI_flags(%ebp)
jnz syscall_trace_entry
cmpl $(NR_syscalls), %eax
jae syscall_badsys
syscall_call:
call *sys_call_table(,%eax,4)
movl %eax,PT_EAX(%esp) # store the return value
syscall_exit:
LOCKDEP_SYS_EXIT
DISABLE_INTERRUPTS(CLBR_ANY) # make sure we don't miss an interrupt
# setting need_resched or sigpending
# between sampling and the iret
TRACE_IRQS_OFF
movl TI_flags(%ebp), %ecx
testl $_TIF_ALLWORK_MASK, %ecx # current->work
jne syscall_exit_work
...
ENDPROC(system_call)
```


### sys_call_table

sys_call_table是在哪里定义的：

```
const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
/*
* Smells like a compiler bug -- it doesn't work
* when the & below is removed.
*/
[0 ... __NR_syscall_max] = &sys_ni_syscall,
#include <asm/syscalls_32.h>
};
```


编译内核的时候，当执行到文件/usr/src/linux-3.10.21/arch/x86/syscalls/Makefile时，该文件会执行/usr/src/linux-3.10.21/arch/x86/syscalls/目录下的shell脚本syscalltbl.sh，该脚本将同目录下的syscall_32.tbl文件作为输入，然后生成文件/usr/src/linux-3.10.21/arch/x86/include/generated/asm/syscalls_32.h，这个文件正是sys_call_table定义中包含的文件asm/syscalls_32.h。

来看下脚本syscalltbl.sh

```
#!/bin/sh
in="$1"
out="$2"
grep '^[0-9]' "$in" | sort -n | (
while read nr abi name entry compat; do
abi=`echo "$abi" | tr '[a-z]' '[A-Z]'`
if [ -n "$compat" ]; then
echo "__SYSCALL_${abi}($nr, $entry, $compat)"
elif [ -n "$entry" ]; then
echo "__SYSCALL_${abi}($nr, $entry, $entry)"
fi
done
) > "$out"
```


其中in和out分别代表的就是syscall_32.tbl和syscalls_32.h文件的路径。脚本大概意思就是读取syscall_32.tbl内容，然后构造语句__SYSCALL_abi(nr, entry,entry)”。

输入文件syscall_32.tbl部分内容如下：

```
#
# 32-bit system call numbers and entry vectors
#
# The format is:
# <number> <abi> <name> <entry point> <compat entry point>
#
# The abi is always "i386" for this file.
#
0 i386 restart_syscall sys_restart_syscall
1 i386 exit sys_exit
2 i386 fork sys_fork stub32_fork
3 i386 read sys_read
4 i386 write sys_write
5 i386 open sys_open compat_sys_open
6 i386 close sys_close
```


输出文件syscalls_32.h的部分内容：

```
__SYSCALL_I386(0, sys_restart_syscall, sys_restart_syscall)
__SYSCALL_I386(1, sys_exit, sys_exit)
__SYSCALL_I386(2, sys_fork, stub32_fork)
__SYSCALL_I386(3, sys_read, sys_read)
__SYSCALL_I386(4, sys_write, sys_write)
__SYSCALL_I386(5, sys_open, compat_sys_open)
__SYSCALL_I386(6, sys_close, sys_close)
```


所以sys_call_table的定义中包含了asm/syscall_32.h，就相当于包含了上面这么多宏定义，sys_call_table就是采用这种方法定义的。

### 添加自己的系统调用

1) 在文件/usr/src/linux-3.10/arch/x86/syscalls/syscall_32.tbl中加入自定义的系统调用号和函数名


2) 在文件/usr/src/linux-3.10/arch/x86/include/asm/syscalls.h文件中加入sys_foo函数的声明


3) 在文件/usr/src/linux-3.10/kernel/sys.c文件中加入对sys_foo的定义


4) 编译和安装内核

### 测试自定义的系统调用

```
#include<stdio.h>
#define __NR_foo 351
int main(void)
{
int rs;
rs = syscall(__NR_foo);
if (!rs)
printf("syscall success!\n");
return 0;
}
```


代码中用到了syscall函数，这个函数功能就是根据给定的系统调用号来调用系统调用

编译运行上面的代码，用dmesg可以看到系统输出的信息:


## MIPS架构

### trap_init

和x86架构一样，在系统启动的时候start_kernel会调用trap_init来初始化异常向量表

```
start_kernel
trap_init
set_except_vector(8, handle_sys);
exception_handlers[n] = handler;
memcpy((void *) (RLX_TRAP_VEC_BASE), &rlx_trap_dispatch, RLX_TRAP_VEC_SIZE);
```


RLX_TRAP_VEC_BASE的地址是0x8000080，相当于是将异常处理函数rlx_trap_dispatch的地址复制到0x8000080的地方，当发生异常中断的时候，便会跳到rlx_trap_dispatch的地方执行。

### rlx_trap_dispatch

在arch/rlx/genex.S文件中定义

```
NESTED(rlx_trap_dispatch, 0, sp)
.set push
.set noat
mfc0 k1, CP0_CAUSE
andi k1, k1, 0x7c
PTR_L k0, exception_handlers(k1)
jr k0
.set pop
END(rlx_trap_dispatch)
```


这个异常处理函数又会调用exception_handlers, 他就是之前用set_except_vector函数赋值的exception_handlers数组，这里我们只关心去调用handle_sys函数。handle_sys函数在arch/rlx/kernel/scall32-o32.S中定义，他里面又调用了sys_call_table。

### sys_call_table

```
.macro syscalltable
sys sys_syscall 8 /* 4000 */
sys sys_exit 1
sys __sys_fork 0
sys sys_read 3
sys sys_write 3
sys sys_open 3 /* 4005 */
sys sys_close 1
...
.type sys_call_table,@object
EXPORT(sys_call_table)
```


### unistd.h

文件include/uapi/asm-generic/unistd.h为每个系统调用规定了唯一的编号，根据这个编号，可以在系统调用表sys_call_table中找到对应表项的内容，他正好是该系统调用的响应函数sys_name的入口地址。

```
...
/* net/socket.c */
#define __NR_socket 198
__SYSCALL(__NR_socket, sys_socket)
#define __NR_socketpair 199
__SYSCALL(__NR_socketpair, sys_socketpair)
#define __NR_bind 200
__SYSCALL(__NR_bind, sys_bind)
#define __NR_listen 201
__SYSCALL(__NR_listen, sys_listen)
#define __NR_accept 202
__SYSCALL(__NR_accept, sys_accept)
#define __NR_connect 203
__SYSCALL(__NR_connect, sys_connect)
#define __NR_getsockname 204
__SYSCALL(__NR_getsockname, sys_getsockname)
#define __NR_getpeername 205
__SYSCALL(__NR_getpeername, sys_getpeername)
#define __NR_sendto 206
__SYSCALL(__NR_sendto, sys_sendto)
#define __NR_recvfrom 207
__SC_COMP(__NR_recvfrom, sys_recvfrom, compat_sys_recvfrom)
...
```