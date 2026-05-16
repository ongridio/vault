---
title: Filesystem Hierarchy Standard
source: http://www.pathname.com/fhs/pub/fhs-2.3.html#PURPOSE18
kind: external
domain: disk
original_date: 2004-01-01
fetched_at: 2026-05-16
tags: [external, disk]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.pathname.com](http://www.pathname.com/fhs/pub/fhs-2.3.html#PURPOSE18)
> 原始日期：2004-01-01
> 抓取日期：2026-05-16

# Filesystem Hierarchy Standard

Copyright © 1994-2004 Daniel Quinlan

Copyright © 2001-2004 Paul 'Rusty' Russell

Copyright © 2003-2004 Christopher Yeoh

This standard consists of a set of requirements and guidelines for file and directory placement under UNIX-like operating systems. The guidelines are intended to support interoperability of applications, system administration tools, development tools, and scripts as well as greater uniformity of documentation for these systems.

All trademarks and copyrights are owned by their owners, unless specifically noted otherwise. Use of a term in this document should not be regarded as affecting the validity of any trademark or service mark.

Permission is granted to make and distribute verbatim copies of this standard provided the copyright and this permission notice are preserved on all copies.

Permission is granted to copy and distribute modified versions of this standard under the conditions for verbatim copying, provided also that the title page is labeled as modified including a reference to the original standard, provided that information on retrieving the original standard is included, and provided that the entire resulting derived work is distributed under the terms of a permission notice identical to this one.

Permission is granted to copy and distribute translations of this standard into another language, under the above conditions for modified versions, except that this permission notice may be stated in a translation approved by the copyright holder.

**Table of Contents**- 1. Introduction
- 2. The Filesystem
- 3. The Root Filesystem
- Purpose
- Requirements
- Specific Options
- /bin : Essential user command binaries (for use by all users)
- /boot : Static files of the boot loader
- /dev : Device files
- /etc : Host-specific system configuration
- /home : User home directories (optional)
- /lib : Essential shared libraries and kernel modules
- /lib<qual> : Alternate format essential shared libraries (optional)
- /media : Mount point for removeable media
- /mnt : Mount point for a temporarily mounted filesystem
- /opt : Add-on application software packages
- /root : Home directory for the root user (optional)
- /sbin : System binaries
- /srv : Data for services provided by this system
- /tmp : Temporary files

- 4. The /usr Hierarchy
- Purpose
- Requirements
- Specific Options
- /usr/X11R6 : X Window System, Version 11 Release 6 (optional)
- /usr/bin : Most user commands
- /usr/include : Directory for standard include files.
- /usr/lib : Libraries for programming and packages
- /usr/lib<qual> : Alternate format libraries (optional)
- /usr/local/share
- /usr/sbin : Non-essential standard system binaries
- /usr/share : Architecture-independent data
- /usr/src : Source code (optional)

- 5. The /var Hierarchy
- Purpose
- Requirements
- Specific Options
- /var/account : Process accounting logs (optional)
- /var/cache : Application cache data
- /var/crash : System crash dumps (optional)
- /var/games : Variable game data (optional)
- /var/lib : Variable state information
- /var/lock : Lock files
- /var/log : Log files and directories
- /var/mail : User mailbox files (optional)
- /var/opt : Variable data for /opt
- /var/run : Run-time variable data
- /var/spool : Application spool data
- /var/tmp : Temporary files preserved between system reboots
- /var/yp : Network Information Service (NIS) database files (optional)

- 6. Operating System Specific Annex
- Linux
- / : Root directory
- /bin : Essential user command binaries (for use by all users)
- /dev : Devices and special files
- /etc : Host-specific system configuration
- /lib64 and /lib32 : 64/32-bit libraries (architecture dependent)
- /proc : Kernel and process information virtual filesystem
- /sbin : Essential system binaries
- /usr/include : Header files included by C programs
- /usr/src : Source code
- /var/spool/cron : cron and at jobs


- 7. Appendix

This standard enables:

Software to predict the location of installed files and directories, and

Users to predict the location of installed files and directories.


We do this by:

Specifying guiding principles for each area of the filesystem,

Specifying the minimum files and directories required,

Enumerating exceptions to the principles, and

Enumerating specific cases where there has been historical conflict.


The FHS document is used by:

Independent software suppliers to create applications which are FHS compliant, and work with distributions which are FHS complaint,

OS creators to provide systems which are FHS compliant, and

Users to understand and maintain the FHS compliance of a system.


The FHS document has a limited scope:

Local placement of local files is a local issue, so FHS does not attempt to usurp system administrators.

FHS addresses issues where file placements need to be coordinated between multiple parties such as local sites, distributions, applications, documentation, etc.


We recommend that you read a typeset version of this document rather than the plain text version. In the typeset version, the names of files and directories are displayed in a constant-width font.

Components of filenames that vary are represented by a description
of the contents enclosed in "*<*" and
"*>*" characters, *<thus>*. Electronic mail addresses are also
enclosed in "<" and ">" but are shown in the usual
typeface.

Optional components of filenames are enclosed in
"*[*" and "*]*" characters and may
be combined with the "*<*" and
"*>*" convention. For example, if a filename is
allowed to occur either with or without an extension, it might be
represented by
*<filename>[.<extension>]*.

Variable substrings of directory names and filenames are indicated
by "***".

The sections of the text marked as
*Rationale* are explanatory and are
non-normative.

This standard assumes that the operating system underlying an FHS-compliant file system supports the same basic security features found in most UNIX filesystems.

It is possible to define two independent distinctions among files: shareable vs. unshareable and variable vs. static. In general, files that differ in either of these respects should be located in different directories. This makes it easy to store files with different usage characteristics on different filesystems.

"Shareable" files are those that can be stored on one host and used on others. "Unshareable" files are those that are not shareable. For example, the files in user home directories are shareable whereas device lock files are not.

"Static" files include binaries, libraries, documentation files and other files that do not change without system administrator intervention. "Variable" files are files that are not static.

Rationale | |
|---|---|
Shareable files can be stored on one host and used on several others. Typically, however, not all files in the filesystem hierarchy are shareable and so each system has local storage containing at least its unshareable files. It is convenient if all the files a system requires that are stored on a foreign host can be made available by mounting one or a few directories from the foreign host. Static and variable files should be segregated because static files, unlike variable files, can be stored on read-only media and do not need to be backed up on the same schedule as variable files. Historical UNIX-like filesystem hierarchies contained both
static and variable files under both Here is an example of a FHS-compliant system. (Other FHS-compliant layouts are possible.) |

The contents of the root filesystem must be adequate to boot, restore, recover, and/or repair the system.

To boot a system, enough must be present on the root partition to mount other filesystems. This includes utilities, configuration, boot loader information, and other essential start-up data.

`/usr`,`/opt`, and`/var`are designed such that they may be located on other partitions or filesystems.To enable recovery and/or repair of a system, those utilities needed by an experienced maintainer to diagnose and reconstruct a damaged system must be present on the root filesystem.

To restore a system, those utilities needed to restore from system backups (on floppy, tape, etc.) must be present on the root filesystem.


Rationale | |
|---|---|
The primary concern used to balance these considerations, which favor placing many things on the root filesystem, is the goal of keeping root as small as reasonably possible. For several reasons, it is desirable to keep the root filesystem small: It is occasionally mounted from very small media. The root filesystem contains many system-specific configuration files. Possible examples include a kernel that is specific to the system, a specific hostname, etc. This means that the root filesystem isn't always shareable between networked systems. Keeping it small on servers in networked systems minimizes the amount of lost space for areas of unshareable files. It also allows workstations with smaller local hard drives. While you may have the root filesystem on a large partition, and may be able to fill it to your heart's content, there will be people with smaller partitions. If you have more files installed, you may find incompatibilities with other systems using root filesystems on smaller partitions. If you are a developer then you may be turning your assumption into a problem for a large number of users. Disk errors that corrupt data on the root filesystem are a greater problem than errors on any other partition. A small root filesystem is less prone to corruption as the result of a system crash.
|

Applications must never create or require special files or subdirectories in the root directory. Other locations in the FHS hierarchy provide more than enough flexibility for any package.

Rationale | |
|---|---|
There are several reasons why creating a new subdirectory of the root filesystem is prohibited: It demands space on a root partition which the system administrator may want kept small and simple for either performance or security reasons. It evades whatever discipline the system administrator may have set up for distributing standard file hierarchies across mountable volumes.
Distributions should not create new directories in the root hierarchy without extremely careful consideration of the consequences including for application portability. |

The following directories, or symbolic links to directories, are
required in `/`.

| Directory | Description |
|---|---|
bin | Essential command binaries |
boot | Static files of the boot loader |
dev | Device files |
etc | Host-specific system configuration |
lib | Essential shared libraries and kernel modules |
media | Mount point for removeable media |
mnt | Mount point for mounting a filesystem temporarily |
opt | Add-on application software packages |
sbin | Essential system binaries |
srv | Data for services provided by this system |
tmp | Temporary files |
usr | Secondary hierarchy |
var | Variable data |

Each directory listed above is specified in detail in separate
subsections below. `/usr` and
`/var` each have a complete section in this
document due to the complexity of those directories.

The following directories, or symbolic links to directories,
must be in `/`, if the corresponding subsystem is
installed:

| Directory | Description |
|---|---|
home | User home directories (optional) |
lib<qual> | Alternate format essential shared libraries (optional) |
root | Home directory for the root user (optional) |

Each directory listed above is specified in detail in separate subsections below.

`/bin` contains commands that may be used by
both the system administrator and by users, but which are required
when no other filesystems are mounted (e.g. in single user mode). It
may also contain commands which are used indirectly by scripts.
[1]

There must be no subdirectories in
`/bin`.

The following commands, or symbolic links to commands, are
required in `/bin`.

| Command | Description |
|---|---|
cat | Utility to concatenate files to standard output |
chgrp | Utility to change file group ownership |
chmod | Utility to change file access permissions |
chown | Utility to change file owner and group |
cp | Utility to copy files and directories |
date | Utility to print or set the system data and time |
dd | Utility to convert and copy a file |
df | Utility to report filesystem disk space usage |
dmesg | Utility to print or control the kernel message buffer |
echo | Utility to display a line of text |
false | Utility to do nothing, unsuccessfully |
hostname | Utility to show or set the system's host name |
kill | Utility to send signals to processes |
ln | Utility to make links between files |
login | Utility to begin a session on the system |
ls | Utility to list directory contents |
mkdir | Utility to make directories |
mknod | Utility to make block or character special files |
more | Utility to page through text |
mount | Utility to mount a filesystem |
mv | Utility to move/rename files |
ps | Utility to report process status |
pwd | Utility to print name of current working directory |
rm | Utility to remove files or directories |
rmdir | Utility to remove empty directories |
sed | The `sed' stream editor |
sh | The Bourne command shell |
stty | Utility to change and print terminal line settings |
su | Utility to change user ID |
sync | Utility to flush filesystem buffers |
true | Utility to do nothing, successfully |
umount | Utility to unmount file systems |
uname | Utility to print system information |

If **/bin/sh** is not a true Bourne shell, it
must be a hard or symbolic link to the real shell command.

The **[** and **test**
commands must be placed together in either `/bin`
or `/usr/bin`.

Rationale | |
|---|---|
For example bash behaves differently when called as
The requirement for the |

The following programs, or symbolic links to programs, must be
in `/bin` if the corresponding subsystem is
installed:

| Command | Description |
|---|---|
csh | The C shell (optional) |
ed | The `ed' editor (optional) |
tar | The tar archiving utility (optional) |
cpio | The cpio archiving utility (optional) |
gzip | The GNU compression utility (optional) |
gunzip | The GNU uncompression utility (optional) |
zcat | The GNU uncompression utility (optional) |
netstat | The network statistics utility (optional) |
ping | The ICMP network test utility (optional) |

If the **gunzip** and **zcat**
programs exist, they must be symbolic or hard links to
gzip. **/bin/csh** may be a symbolic link to
**/bin/tcsh** or
**/usr/bin/tcsh**.

Rationale | |
|---|---|
The tar, gzip and cpio commands have been added to make restoration of a
system possible (provided that Conversely, if no restoration from the root partition is ever
expected, then these binaries might be omitted (e.g., a ROM chip root,
mounting |

This directory contains everything required for the boot process except configuration files not needed at boot time and the map installer. Thus /boot stores data that is used before the kernel begins executing user-mode programs. This may include saved master boot sectors and sector map files. [2]

The `/dev` directory is the location of
special or device files.

If it is possible that devices in `/dev` will
need to be manually created, `/dev` must contain a
command named `MAKEDEV`, which can create devices
as needed. It may also contain a `MAKEDEV.local`
for any local devices.

If required, `MAKEDEV` must have provisions
for creating any device that may be found on the system, not just
those that a particular implementation installs.

The `/etc` hierarchy contains configuration
files. A "configuration file" is a local file used to control the
operation of a program; it must be static and cannot be an executable
binary.
[4]

No binaries may be located under `/etc`.
[5]

The following directories, or symbolic links to directories are
required in `/etc`:

The following directories, or symbolic links to directories must
be in `/etc`, if the corresponding subsystem is
installed:

The following files, or symbolic links to files, must be in
`/etc` if the corresponding subsystem is
installed:
[6]

| File | Description |
|---|---|
csh.login | Systemwide initialization file for C shell logins (optional) |
exports | NFS filesystem access control list (optional) |
fstab | Static information about filesystems (optional) |
ftpusers | FTP daemon user access control list (optional) |
gateways | File which lists gateways for routed (optional) |
gettydefs | Speed and terminal settings used by getty (optional) |
group | User group file (optional) |
host.conf | Resolver configuration file (optional) |
hosts | Static information about host names (optional) |
hosts.allow | Host access file for TCP wrappers (optional) |
hosts.deny | Host access file for TCP wrappers (optional) |
hosts.equiv | List of trusted hosts for rlogin, rsh, rcp (optional) |
hosts.lpd | List of trusted hosts for lpd (optional) |
inetd.conf | Configuration file for inetd (optional) |
inittab | Configuration file for init (optional) |
issue | Pre-login message and identification file (optional) |
ld.so.conf | List of extra directories to search for shared libraries (optional) |
motd | Post-login message of the day file (optional) |
mtab | Dynamic information about filesystems (optional) |
mtools.conf | Configuration file for mtools (optional) |
networks | Static information about network names (optional) |
passwd | The password file (optional) |
printcap | The lpd printer capability database (optional) |
profile | Systemwide initialization file for sh shell logins (optional) |
protocols | IP protocol listing (optional) |
resolv.conf | Resolver configuration file (optional) |
rpc | RPC protocol listing (optional) |
securetty | TTY access control for root login (optional) |
services | Port names for network services (optional) |
shells | Pathnames of valid login shells (optional) |
syslog.conf | Configuration file for syslogd (optional) |

`mtab` does not fit the static nature of
`/etc`: it is excepted for historical reasons.
[7]

Host-specific configuration files for add-on application
software packages must be installed within the directory
`/etc/opt/<subdir>`, where
`<subdir>` is the name of the subtree in
`/opt` where the static data from that package is
stored.

No structure is imposed on the internal arrangement of
`/etc/opt/<subdir>`.

If a configuration file must reside in a different location in
order for the package or system to function properly, it may be placed
in a location other than
`/etc/opt/<subdir>`.

Rationale | |
|---|---|
Refer to the rationale for |

*/etc/X11* is the location for all X11
host-specific configuration. This directory is necessary to allow
local control if */usr* is mounted read
only.

The following files, or symbolic links to files, must be in
`/etc/X11` if the corresponding subsystem is
installed:

| File | Description |
|---|---|
Xconfig | The configuration file for early versions of XFree86 (optional) |
XF86Config | The configuration file for XFree86 versions 3 and 4 (optional) |
Xmodmap | Global X11 keyboard modification file (optional) |

Subdirectories of `/etc/X11` may include
those for `xdm` and for any other programs (some
window managers, for example) that need them.
[8]
We recommend that window managers with only one configuration file
which is a default `.*wmrc` file must name it
`system.*wmrc` (unless there is a widely-accepted
alternative name) and not use a subdirectory. Any window manager
subdirectories must be identically named to the actual window manager
binary.

Generic configuration files defining high-level parameters of
the SGML systems are installed here. Files with names
`*.conf` indicate generic configuration files.
File with names `*.cat` are the DTD-specific
centralized catalogs, containing references to all other catalogs
needed to use the given DTD. The super catalog file
`catalog` references all the centralized
catalogs.

Generic configuration files defining high-level parameters of
the XML systems are installed here. Files with names
`*.conf` indicate generic configuration files.
The super catalog file
`catalog` references all the centralized
catalogs.

`/home` is a fairly standard concept, but it
is clearly a site-specific filesystem.
[9]
The setup will differ from host to host. Therefore, no program should
rely on this location.
[10]

User specific configuration files for applications are stored in the user's home directory in a file that starts with the '.' character (a "dot file"). If an application needs to create more than one dot file then they should be placed in a subdirectory with a name starting with a '.' character, (a "dot directory"). In this case the configuration files should not start with the '.' character. [11]

The `/lib` directory contains those shared
library images needed to boot the system and run the commands in the
root filesystem, ie. by binaries in `/bin` and
`/sbin`.
[12]

At least one of each of the following filename patterns are required (they may be files, or symbolic links):

| File | Description |
|---|---|
libc.so.* | The dynamically-linked C library (optional) |
ld* | The execution time linker/loader (optional) |

If a C preprocessor is installed, */lib/cpp*
must be a reference to it, for historical reasons.
[13]

The following directories, or symbolic links to directories,
must be in `/lib`, if the corresponding subsystem
is installed:

There may be one or more variants of the
`/lib` directory on systems which support more than
one binary format requiring separate libraries.
[14]

If one or more of these directories exist, the requirements for
their contents are the same as the normal `/lib`
directory, except that `/lib<qual>/cpp` is
not required.
[15]

This directory contains subdirectories which are used as mount points for removeable media such as floppy disks, cdroms and zip disks.

Rationale | |
|---|---|
Historically there have been a number of other different places
used to mount removeable media such as |

The following directories, or symbolic links to directories,
must be in `/media`, if the corresponding subsystem
is installed:

| Directory | Description |
|---|---|
floppy | Floppy drive (optional) |
cdrom | CD-ROM drive (optional) |
cdrecorder | CD writer (optional) |
zip | Zip drive (optional) |

On systems where more than one device exists for mounting a certain type of media, mount directories can be created by appending a digit to the name of those available above starting with '0', but the unqualified name must also exist. [16]

This directory is provided so that the system administrator may temporarily mount a filesystem as needed. The content of this directory is a local issue and should not affect the manner in which any program is run.

This directory must not be used by installation programs: a suitable temporary directory not in use by the system must be used instead.

`/opt` is reserved for the installation of
add-on application software packages.

A package to be installed in `/opt` must
locate its static files in a separate
`/opt/<package>` or
`/opt/<provider>` directory
tree, where `<package>` is a name that
describes the software package and
`<provider>` is the provider's LANANA
registered name.

The directories `/opt/bin`,
`/opt/doc`, `/opt/include`,
`/opt/info`, `/opt/lib`, and
`/opt/man` are reserved for local system
administrator use. Packages may provide "front-end" files intended to
be placed in (by linking or copying) these reserved directories by the
local system administrator, but must function normally in the absence
of these reserved directories.

Programs to be invoked by users must be located in the directory
`/opt/<package>/bin` or under the
`/opt/<provider>` hierarchy. If the package
includes UNIX manual pages, they must be located in
`/opt/<package>/share/man` or under the
`/opt/<provider>` hierarchy, and the same
substructure as `/usr/share/man` must be
used.

Package files that are variable (change in normal operation)
must be installed in `/var/opt`. See the section
on `/var/opt` for more information.

Host-specific configuration files must be installed in
`/etc/opt`. See the section on
`/etc` for more information.

No other package files may exist outside the
`/opt`, `/var/opt`, and
`/etc/opt` hierarchies except for those package
files that must reside in specific locations within the filesystem
tree in order to function properly. For example, device lock files
must be placed in `/var/lock` and devices must be
located in `/dev`.

Distributions may install software in `/opt`,
but must not modify or delete software installed by the local system
administrator without the assent of the local system
administrator.

Rationale | |
|---|---|
The use of The Intel Binary Compatibility Standard v. 2 (iBCS2) also
provides a similar structure for Generally, all data required to support a package on a system
must be present within The minor restrictions on distributions using
The structure of the directories below
|


[... 内容超长，已截断；完整原文见 source URL ...]
