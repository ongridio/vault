---
title: cgroups
source: https://wiki.archlinux.org/title/Cgroups
kind: external
domain: systemd
author: ArchWiki
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, systemd]
---

> [!info] External article · imported reference
> Source: [wiki.archlinux.org](https://wiki.archlinux.org/title/Cgroups)
> Author: ArchWiki
> License: cc-by-sa
> Fetched: 2026-05-18

# cgroups

**Control groups** (or **cgroups** as they are commonly known) are a feature provided by the Linux kernel to manage, restrict, and audit groups of processes. Compared to other approaches like the [nice(1)](https://man.archlinux.org/man/nice.1) command or `/etc/security/limits.conf`, cgroups are more flexible as they can operate on (sub)sets of processes (possibly with different system users).

Control groups can be accessed with various tools:

- using directives in [systemd](/title/Systemd "Systemd") unit files to specify limits for services and slices;
- by accessing the `cgroup` filesystem directly;
- via tools like `cgcreate`, `cgexec` and `cgclassify` (part of the [libcgroup](https://aur.archlinux.org/packages/libcgroup/)AUR and [libcgroup-git](https://aur.archlinux.org/packages/libcgroup-git/)AUR packages);
- using the "rules engine daemon" to automatically move certain users/groups/commands to groups (`/etc/cgrules.conf` and `cgconfig.service`) (part of the [libcgroup](https://aur.archlinux.org/packages/libcgroup/)AUR and [libcgroup-git](https://aur.archlinux.org/packages/libcgroup-git/)AUR packages); and
- through other software such as [Linux Containers](/title/Linux_Containers "Linux Containers") (LXC) virtualization.

For Arch Linux, systemd is the preferred and easiest method of invoking and configuring cgroups as it is a part of the default installation.

## Installing

Make sure you have one of these packages [installed](/title/Install "Install") for automated cgroup handling:

- [systemd](https://archlinux.org/packages/?name=systemd) - for controlling resources of a systemd service.
- [libcgroup](https://aur.archlinux.org/packages/libcgroup/)AUR, [libcgroup-git](https://aur.archlinux.org/packages/libcgroup-git/)AUR - set of standalone tools (`cgcreate`, `cgclassify`, persistence via `cgconfig.conf`).

## With systemd

### Hierarchy

Current cgroup hierarchy can be seen with `systemctl status` or `systemd-cgls` command.

```
$ systemctl status
```

```
● myarchlinux
    State: running
     Jobs: 0 queued
   Failed: 0 units
    Since: Wed 2019-12-04 22:16:28 UTC; 1 day 4h ago
   CGroup: /
           ├─user.slice 
           │ └─user-1000.slice 
           │   ├─user@1000.service 
           │   │ ├─gnome-shell-wayland.service 
           │   │ │ ├─ 1129 /usr/bin/gnome-shell
           │   │ ├─gnome-terminal-server.service 
           │   │ │ ├─33519 /usr/lib/gnome-terminal-server
           │   │ │ ├─37298 fish
           │   │ │ └─39239 systemctl status
           │   │ ├─init.scope 
           │   │ │ ├─1066 /usr/lib/systemd/systemd --user
           │   │ │ └─1067 (sd-pam)
           │   └─session-2.scope 
           │     ├─1053 gdm-session-worker [pam/gdm-password]
           │     ├─1078 /usr/bin/gnome-keyring-daemon --daemonize --login
           │     ├─1082 /usr/lib/gdm-wayland-session /usr/bin/gnome-session
           │     ├─1086 /usr/lib/gnome-session-binary
           │     └─3514 /usr/bin/ssh-agent -D -a /run/user/1000/keyring/.ssh
           ├─init.scope 
           │ └─1 /sbin/init
           └─system.slice 
             ├─systemd-udevd.service 
             │ └─285 /usr/lib/systemd/systemd-udevd
             ├─systemd-journald.service 
             │ └─272 /usr/lib/systemd/systemd-journald
             ├─NetworkManager.service 
             │ └─656 /usr/bin/NetworkManager --no-daemon
             ├─gdm.service 
             │ └─668 /usr/bin/gdm
             └─systemd-logind.service 
               └─654 /usr/lib/systemd/systemd-logind
```

### Find cgroup of a process

The cgroup name of a process can be found in `/proc/PID/cgroup`.

For example, the cgroup of the shell:

```
$ cat /proc/self/cgroup
```

```
0::/user.slice/user-1000.slice/session-3.scope
```

### cgroup resource usage

The `systemd-cgtop` command can be used to see the resource usage:

```
$ systemd-cgtop
```

```
Control Group                            Tasks   %CPU   Memory  Input/s Output/s
user.slice                                 540  152,8     3.3G        -        -
user.slice/user-1000.slice                 540  152,8     3.3G        -        -
user.slice/u…000.slice/session-1.scope     425  149,5     3.1G        -        -
system.slice                                37      -   215.6M        -        -
```

### Custom cgroups

[systemd.slice(5)](https://man.archlinux.org/man/systemd.slice.5) systemd unit files can be used to define a custom cgroup configuration. They must be placed in a systemd directory, such as `/etc/systemd/system/`. The resource control options that can be assigned are documented in [systemd.resource-control(5)](https://man.archlinux.org/man/systemd.resource-control.5).

This is an example slice unit that only allows 30% of one CPU to be used:

```
/etc/systemd/system/my.slice
```

```
[Slice]
CPUQuota=30%
```

Remember to do a [daemon-reload](/title/Daemon-reload "Daemon-reload") to pick up any new or changed `.slice` files.

### As service

#### Service unit file

Resources can be directly specified in service definition or as a [drop-in file](/title/Drop-in_file "Drop-in file"):

```
[Service]
MemoryMax=1G
```

This example limits the service to 1 gigabyte.

#### Grouping unit under a slice

Service can be specified what slice to run in:

```
[Service]
Slice=my.slice
```

### As root

`systemd-run` can be used to run a command in a specific slice.

```
# systemd-run --slice=my.slice command
```

`--uid=username` option can be used to spawn the command as specific user.

```
# systemd-run --uid=username --slice=my.slice command
```

The `--shell` option can be used to spawn a command shell inside the slice.

### As unprivileged user

Unprivileged users can divide the resources provided to them into new cgroups, if some conditions are met.

Cgroups v2 must be utilized for a non-root user to be allowed managing cgroup resources.

#### Controller types

Not all resources can be controlled by user.

| Controller | Can be controlled by user | Options |
| --- | --- | --- |
| cpu | Requires delegation | CPUAccounting, CPUWeight, CPUQuota, AllowedCPUs, AllowedMemoryNodes |
| io | Requires delegation | IOWeight, IOReadBandwidthMax, IOWriteBandwidthMax, IODeviceLatencyTargetSec |
| memory | Yes | MemoryLow, MemoryHigh, MemoryMax, MemorySwapMax |
| pids | Yes | TasksMax |
| rdma | No | ? |
| eBPF | No | IPAddressDeny, DeviceAllow, DevicePolicy |

**Note** eBPF is technically not a controller but those systemd options implemented using it and only root is allowed to set them.

#### User delegation

For user to control cpu and io resources, the resources need to be delegated. This can be done with a [drop-in file](/title/Drop-in_file "Drop-in file").

For example if your user id is 1000:

```
/etc/systemd/system/user@1000.service.d/delegate.conf
```

```
[Service]
Delegate=cpu cpuset io
```

Reboot and verify that the slice your user session is under has cpu and io controller:

```
$ cat /sys/fs/cgroup/user.slice/user-1000.slice/cgroup.controllers
```

```
cpuset cpu io memory pids
```

#### User-defined slices

The user slice files can be placed in `~/.config/systemd/user/`.

To run the command under certain slice:

```
$ systemd-run --user --slice=my.slice command
```

You can also run your login shell inside the slice:

```
$ systemd-run --user --slice=my.slice --shell
```

### Run-time adjustment

cgroups resources can be adjusted at run-time using `systemctl set-property` command. Option syntax is the same as in [systemd.resource-control(5)](https://man.archlinux.org/man/systemd.resource-control.5).

**Warning** The adjustments will be made **permanent** unless `--runtime` option is passed. Adjustments are saved at `/etc/systemd/system.control/` for system wide options and `.config/systemd/user.control/` for user options.

**Note** Not all resources changes immediately take effect. For example, changing TaskMax will only take effect on spawning new processes.

For example, cutting off internet access for all user sessions:

```
$ systemctl set-property user.slice IPAddressDeny=any
```

## With libcgroup and the cgroup virtual filesystem

One layer lower than management with systemd is the cgroup virtual file system. "libcgroup" provides a library and utilities for making management easier, so we will use them here as well.

The reason for using the lower level is simple: systemd does not provide an interface for *every single interface file* in cgroups and nor should it be expected for it to provide them at any point in the future. It is completely harmless to read from them for additional insights on a cgroup's resource use.

One cgroup *should* only have one set of programs writing to it to avoid race conditions, the "single-writer rule". This is not enforced by the kernel, but following this recommendation prevents hard-to-debug issues from happening. To set the boundary at which systemd stops managing child cgroups, see the `Delegate=` property. Otherwise do not be surprised if system to overwrites what you have set.

### Creating ad hoc groups

**Warning** Creating an "ad hoc" group manually does not tell systemd to delegate its management. This should not be done except for testing; in production environments, create the group with systemd with the proper `Delegate=` settings (`Delegate=yes` for everything).

One of the powers of cgroups is that you can create "ad-hoc" groups on the fly. You can even grant the privileges to create custom groups to regular users. `groupname` is the cgroup name:

```
# cgcreate -a user -t user -g memory,cpu:groupname
```

**Note** Since cgroup v2, the "memory,cpu" part is useless. All available controllers will be included regardless because they are all directly under the root cgroup. To speed up typing, try just `cpu` or `\*`.

Now all the tunables in the group `groupname` are writable by your user:

```
$ ls -l /sys/fs/cgroup/groupname
```

```
total 0
-r--r--r-- 1 root root 0 Jun 20 19:38 cgroup.controllers
-r--r--r-- 1 root root 0 Jun 20 19:38 cgroup.events
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.freeze
--w------- 1 root root 0 Jun 20 19:38 cgroup.kill
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.max.depth
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.max.descendants
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.pressure
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.procs
-r--r--r-- 1 root root 0 Jun 20 19:38 cgroup.stat
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.subtree_control
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.threads
-rw-r--r-- 1 root root 0 Jun 20 19:38 cgroup.type
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.idle
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.max
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.max.burst
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.pressure
-r--r--r-- 1 root root 0 Jun 20 19:38 cpu.stat
-r--r--r-- 1 root root 0 Jun 20 19:38 cpu.stat.local
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.uclamp.max
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.uclamp.min
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.weight
-rw-r--r-- 1 root root 0 Jun 20 19:38 cpu.weight.nice
-rw-r--r-- 1 root root 0 Jun 20 19:38 io.pressure
-rw-r--r-- 1 root root 0 Jun 20 19:38 irq.pressure
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.current
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.events
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.events.local
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.high
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.low
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.max
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.min
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.numa_stat
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.oom.group
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.peak
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.pressure
--w------- 1 root root 0 Jun 20 19:38 memory.reclaim
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.stat
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.swap.current
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.swap.events
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.swap.high
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.swap.max
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.swap.peak
-r--r--r-- 1 root root 0 Jun 20 19:38 memory.zswap.current
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.zswap.max
-rw-r--r-- 1 root root 0 Jun 20 19:38 memory.zswap.writeback
-r--r--r-- 1 root root 0 Jun 20 19:38 pids.current
-r--r--r-- 1 root root 0 Jun 20 19:38 pids.events
-r--r--r-- 1 root root 0 Jun 20 19:38 pids.events.local
-rw-r--r-- 1 root root 0 Jun 20 19:38 pids.max
-r--r--r-- 1 root root 0 Jun 20 19:38 pids.peak
```

Cgroups are hierarchical, so you can create as many subgroups as you like. If a normal user wants to make new subgroup called `foo` and distributes memory and cpu controller to the subgrroup:

```
$ cgcreate -g memory,cpu:groupname/foo
```

**Note** Resources are distributed top-down and a cgroup can further distribute a resource only if the resource has been distributed to it from the parent. It means that a controller is available for limiting the resources or distributing to its subgroups in some group only if its parent distributes the controller to the group.

### Using cgroups

As previously mentioned, only one thing *should* write to a cgroup at any point. This does not affect non-write operations including spawning new processes inside a group, moving processes to a group, or reading properties from cgroup files.

#### Spawning and moving processes

**Note** In cgroup v2, a cgroup that distributes domain resources to its child groups cannot have processes inside of them and vice versa. This is an intentional limitation to reduce chaos!

libcgroup contains a simple tool for running new processes inside a cgroup. If a normal user wants to run a `bash` shell under a our previous `groupname/foo`:

```
$ cgexec    -g cpu:groupname/foo bash
```

Inside of the shell, we can confirm which cgroup it belongs to with:

```
$ cat /proc/self/cgroup
```

```
0::/groupname/foo
```

This makes use of `/proc/$PID/cgroup`, a file that exists in every process. Manually writing to the file causes the cgroup to change as well.

To move all 'bash' commands to this group:

```
$ pidof bash
```

```
13244 13266
```

```
$ cgclassify -g cpu:groupname/foo `pidof bash`
```

```
$ cat /proc/13244/cgroup
```

```
0::/groupname/foo
```

Internally (i.e. without `cgclassify` the kernel provides two ways to move processes between cgroups. These two are equivalent:

```
$ echo 0::/groupname/foo > /proc/13244/cgroup
$ echo 13244 > /sys/fs/cgroup/groupname/foo/cgroup.procs
```

**Note** In this last command, only one PID can be written at a time, so repeat this for each process that must be moved.

#### Manipulating group properties

A new subdirectory is crated for `groupname/foo` at its creation, located at `/sys/fs/cgroup/groupname/foo`. These files can be read and written to change the group's properties. (Again, writing is not recommended unless delegation is done!)

Let us try to see how much memory all the processes in our group is taking up:

```
$ cat /sys/fs/cgroup/groupname/foo/memory.current
```

```
1536000
```

To limit the RAM (not swap) usage of all processes, run the following:

```
$ echo 10000000 > /sys/fs/cgroup/groupname/foo/memory.max
```

To change the CPU priority of this group (the default is 100):

```
$ echo 10 > /sys/fs/cgroup/groupname/foo/cpu.weight
```

You can find more tunables or statistics by listing the cgroup directory.

### Persistent group configuration

**Note** [systemd](/title/Systemd "Systemd")

≥ 205 offers a better way to manage cgroups in unit files. The following still works but should not be used in new setups.

If you want your cgroups to be created at boot, you can define them in `/etc/cgconfig.conf` instead. This causes a service booted at launch to configure your cgroups. See the relevant manual page about the syntax of this file; we will make no instruction on how to use a truly deprecated mechanism.

## Examples

### Restrict memory or CPU use of a command

The following example shows a *cgroup* that constrains a given command to 2GB of memory.

```
$ systemd-run --scope -p MemoryMax=2G --user command
```

The following example shows a command restricted to 20% of one CPU core.

```
$ systemd-run --scope -p CPUQuota="20%" --user command
```

### Matlab

Doing large calculations in [MATLAB](/title/MATLAB "MATLAB") can crash your system, because Matlab does not have any protection against taking all your machine's memory or CPU. The following examples show a *cgroup* that constrains Matlab to first 6 CPU cores and 5 GB of memory.

#### With systemd

```
~/.config/systemd/user/matlab.slice
```

```
[Slice]
AllowedCPUs=0-5
MemoryHigh=6G
```

Launch Matlab like this (be sure to use the right path):

```
$ systemd-run --user --slice=matlab.slice /opt/MATLAB/2012b/bin/matlab -desktop
```

## Documentation

- For information on controllers and what certain switches and tunables mean, refer to kernel's documentation [v2](https://docs.kernel.org/admin-guide/cgroup-v2.html) (or install [linux-docs](https://archlinux.org/packages/?name=linux-docs) and see `/usr/src/linux/Documentation/cgroup`)
- Linux manual page: [cgroups(7)](https://man.archlinux.org/man/cgroups.7)
- A detailed and complete Resource Management Guide can be found in the [Red Hat Enterprise Linux documentation](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/resource_management_guide/ch01#sec-How_Control_Groups_Are_Organized).

For commands and configuration files, see relevant man pages, e.g. [cgcreate(1)](https://github.com/libcgroup/libcgroup/blob/main/doc/man/cgcreate.1) or [cgrules.conf(5)](https://github.com/libcgroup/libcgroup/blob/main/doc/man/cgrules.conf.5)

## Historical note: cgroup v1

Before our current cgroup v2 there was an earlier version called [v1](https://docs.kernel.org/admin-guide/cgroup-v1/index.html). V1 allowed a lot of additional flexibility including a non-unified hierarchy and thread-granular management. This was, in retrospect, a bad idea (see [the rationales for v2](https://docs.kernel.org/admin-guide/cgroup-v2.html#issues-with-v1-and-rationales-for-v2)):

- Even though many hierarchies can exist and processes can be bound to several hierarchies, a controller can ever be used in one hierarchy. This makes the many hierarchies essentially pointless, with the usual setup being to bind each controller to one hierarchy (e.g. `/sys/fs/cgroup/memory/`), then each process to multiple hierarchies. This in turn made tools like `cgcreate` essential for syncing the membership of processes across multiple hierarchies.
- Thread-granular management caused cgroup to be abused as a way for a process to manage itself. The proper way to do this is through system calls, not the convoluted interfaces that have emerged to support this use. Self-managing requires clunky string management and is inherently vulnerable to race conditions.

To avoid further chaos, cgroup v2 has [two key design rules](https://systemd.io/CGROUP_DELEGATION/#two-key-design-rules) on top of the removal of features:

- If a cgroup has child cgroups, it cannot have attached processes (with the exception of the root cgroup). This is enforced on v2 and helps make the single-writer rule (below) viable.
- Each cgroup should only have one process managing it at the same time (single-writer rule). This is not enforced anywhere but should be respected at most times to avoid pain from software fighting each other over what to do to a group.
  - On systemd systems, the root cgroup is managed by systemd, making any change *not* by systemd a violation of this rule (or, as it's not enforced, recommendation), unless when `Delegate=` is set on the surrounding service or scope unit to tell systemd to not meddle with what is inside.

Before systemd v258, the [kernel parameters](/title/Kernel_parameter "Kernel parameter") `SYSTEMD_CGROUP_ENABLE_LEGACY_FORCE=1 systemd.unified_cgroup_hierarchy=0` could be used to force booting with cgroup-v1 (the first parameter was added in v256 [to make it harder to use cgroup-v1](https://github.com/systemd/systemd/issues/30852)). However, this feature has now been removed. It is still worth knowing about because some software like to put `systemd.unified_cgroup_hierarchy=0` in your kernel command-line without telling you, causing your entire system to break.

## See also
