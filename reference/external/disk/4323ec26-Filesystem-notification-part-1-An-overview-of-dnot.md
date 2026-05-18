---
title: Filesystem notification, part 1: An overview of dnotify and inotify
source: https://lwn.net/Articles/604686/
kind: external
domain: disk
author: July 9
original_date: 2014-07-09
fetched_at: 2026-05-16
bookmark_title: Filesystem notification, part 1: An overview of dnotify and inotify [LWN.net]
tags: [external, disk]
---

> [!info] 外部文章 · 自动导入
> 来源：[lwn.net](https://lwn.net/Articles/604686/)
> 作者：July 9
> 原始日期：2014-07-09
> 抓取日期：2026-05-16

# Filesystem notification, part 1: An overview of dnotify and inotify

# Filesystem notification, part 1: An overview of dnotify and inotify

Filesystem notification APIs provide a mechanism by which applications can be informed when events happen within a filesystem—for example, when a file is opened, modified, deleted, or renamed. Over time, Linux has acquired three different filesystem notification APIs, and it is instructive to look at them to understand what the differences between the APIs are. It's also worthwhile to consider what lessons have been learned during the design of the APIs—and what lessons remain to be learned.

$ sudo subscribe todaySubscribe today and elevate your LWN privileges. You’ll have access to all of LWN’s high-quality articles as soon as they’re published, and help support LWN in the process. Act now and you can start with a free trial subscription.


This article is thus the first in a series that looks at the Linux filesystem notification APIs: dnotify, inotify, and fanotify. To begin with, we briefly describe the original API, dnotify, and look at its limitations. We'll then look at the inotify API, and consider the ways in which it improves on dnotify. In a subsequent article, we'll take a look at the fanotify API.

#### Filesystem notification use cases

In order to compare filesystem notification APIs, it's useful to consider some of the use cases for those APIs. Some of the common use cases are the following:

-
*Caching a model of filesystem objects*: The application wants to maintain an internal representation that accurately reflects the current set of objects in a filesystem, or some subtree of that filesystem. An example of such an application is a file manager, which presents the user with a graphical representation of the objects in a filesystem. -
*Logging filesystem activity*: The application wants to record all of the events (or some subset of event types) that occur for the monitored filesystem objects. -
*Gatekeeping filesystem operations*: The application wants to intervene when a filesystem event occurs. The classic example of such an application is an antivirus system: when another program tries to (for example) execute a file, the antivirus system first checks the contents of the file for malware, and then either allows the execution to proceed if the file contents are benign, or prevents execution if a virus is detected.

#### In the beginning: dnotify

Without a kernel-supported filesystem notification API, an
application must resort to techniques such as polling the state of
directories and files using repeated invocations of system calls
such as `stat()` and the `readdir()` library
function. Such polling is, of course, slow and inefficient.
Furthermore, this approach allows only a limited range of events
to be detected, for example, creation of a file, deletion of a
file, and changes of file metadata such as permissions and file
size. By contrast, operations such as file renames are difficult
to identify.

Those problems led to the creation of the first in-kernel implementation of a filesystem notification API, dnotify, which was implemented by Stephen Rothwell (these days, the maintainer of the linux-next tree) and which first appeared in Linux 2.4.0 (in 2001).

Because it was the first attempt at implementing a filesystem
notification API, done at a time when the problem was less
well understood and when some of the pitfalls of API design were
less easily recognized, the dnotify API has a number of
peculiarities.
To begin with, the interface is multiplexed on the existing
`fcntl()` system
call.
(By
contrast, the later inotify and fanotify APIs were each
implemented using new system calls.) To enable
monitoring, one makes a call of the form:

fcntl(fd, F_NOTIFY, mask);

Here, `fd` is a file descriptor that specifies a directory
to be monitored, and this brings us to the second oddity of
the API: dnotify can be used to monitor only whole directories;
monitoring individual files is not possible.
The `mask` specifies the set of events to be monitored in
the directory. These include events for file access, modification,
creation, deletion, and attribute changes
(e.g., permission and ownership changes) that are fully listed in the
fcntl(2)
man page.

A further dnotify oddity is its method of notification. When an
event occurs, the monitoring application is sent a signal
(`SIGIO` by default, but this can be changed). The signal
on its own does not identify which directory had the event, but if
we
use `sigaction()`
to establish the handler using the `SA_SIGINFO` flag, then
the handler receives a `siginfo_t` argument
whose `si_fd` field contains the file descriptor associated
with the directory. At that point, the application then needs to
rescan the directory to determine which file has changed. (In
typical usage, the application would maintain a data structure
that caches a mapping of file descriptors to directory names, so
that it can map `si_fd` back to a directory name.)

A simple example of the use of dnotify can be found here.

#### Problems with dnotify

As is probably clear, the dnotify API is cumbersome, and has a number of limitations. As already noted, we can monitor only entire directories, not individual files. Furthermore, dnotify provides notification for a rather modest range of events. Most notably, by comparison to inotify, dnotify can't tell us when a file was opened or closed. However, there are also some other serious limitations of the API.

The use of signals as a notification method causes a number of
difficulties. The first of these is that signals are delivered
asynchronously: catching signals with a handler can be racy and
error-prone. One way around that particular difficulty is to
instead accept signals synchronously using
`sigwaitinfo()`.
The use of `SIGIO` as the default notification signal is
also undesirable, because it is one of the traditional signals
that does not queue. This means that if events are generated more
quickly than the application can process the signals, then some
notifications will be lost. (This difficulty can be circumvented
by changing the notification signal to one of the so-called realtime
signals, which can be queued.)

Signals are also problematic because they convey little information: at most, we get a signal number (it is possible to arrange for different directories to notify using different signals) and a file descriptor number. We get no information about which particular file in a directory triggered an event, or indeed what kind of event occurred. (One can play tricks such as opening multiple file descriptors for the same directory, each of which notifies a different set of events, but this adds complexity to the application.) One further reason that using signals as a notification method can be a problem is that an application that uses dnotify might also make use of a library that employs signals: the use of a particular signal by dnotify in the main program may conflict with the library's use of the same signal (or vice versa).

A final significant limitation of the dnotify API is the need to open a file descriptor for each directory that is monitored. This is problematic for two reasons. First, an application that monitors a large number of directories may quickly run out of file descriptors. However, a more serious problem is that holding file descriptors open on a filesystem prevents that filesystem from being unmounted.

Notwithstanding these API problems, dnotify did provide an efficiency improvement over simply polling a filesystem, and dnotify came to be employed in some widely used tools such as the Beagle desktop search tool. However, it soon became clear that a better API would make life easier for user-space applications.

#### Enter inotify

The inotify API was developed by John McCutchan with support from Robert Love. First released in Linux 2.6.13 (in 2005), inotify aimed to address all of the obvious problems with dnotify.

The API employs three dedicated system
calls—`inotify_init()`, `inotify_add_watch()`,
and `inotify_rm_watch()`—and makes use of the
traditional `read()` system call as well.


`inotify_init()`
creates an inotify instance—a kernel data structure that
records which filesystem objects should be monitored and
maintains a list of events that have been generated for those
objects. The call returns a file descriptor that is employed by
the rest of the API to refer to this inotify instance. The
diagram at right summarizes the operation of an inotify instance.

`inotify_add_watch()`
allows us to modify the set of filesystem objects monitored by an
inotify instance. We can add new objects (files and directories)
to the monitoring list, specifying which events are to be
notified, and change the set of events that are notified for an
object that is already in the monitoring
list. Unsurprisingly, `inotify_rm_watch()`
is the converse of `inotify_add_watch()`: it removes an
object from the monitoring list.

The three arguments to `inotify_add_watch()` are an inotify
file descriptor, a filesystem pathname, and a bit mask:

int inotify_add_watch(int fd, const char *pathname, uint32_t mask);

The
`mask` argument specifies the set of events to be notified
for the filesystem object referred to by `pathname` and can
include some additional bits that affect the behavior of the
call. As an example, the following code allows us to monitor file
creation and deletion events inside the directory `mydir`,
as well as monitor for deletion of the directory itself:

int fd, wd; fd = inotify_init(); wd = inotify_add_watch(fd, "mydir", IN_CREATE | IN_DELETE | IN_DELETE_SELF);

A full list of the bits that can be included in the `mask`
argument is given in
the inotify(7)
man page.
The set of events notified by inotify
is a superset of that provided by dnotify.
Most notably, inotify provides notifications when filesystem objects
are opened and closed, and provides much more information
for file rename events, as we outline below.

The return value of `inotify_add_watch()` is a "watch
descriptor", which is an integer value that uniquely identifies the
specified filesystem object within the inotify monitoring list.
An `inotify_add_watch()` call that specifies a filesystem
object that is already being monitored (possibly via a different
pathname) will return the same watch descriptor number as was
returned by the `inotify_add_watch()` that first added the
object to the monitoring list.

When events occur for objects in the monitoring list, they can be
read from the inotify file descriptor using `read()`. (The
inotify file descriptor can also be monitored for readability
using `select()`, `poll()`, and `epoll()`.)
Each
`read()` returns one or more structures of the following
form to describe an event:

struct inotify_event { int wd; /* Watch descriptor */ uint32_t mask; /* Bit mask describing event */ uint32_t cookie; /* Unique cookie associating related events */ uint32_t len; /* Size of name field */ char name[]; /* Optional null-terminated name */ };

The `wd` field is a watch descriptor that was previously
returned by `inotify_add_watch()`. By maintaining a data
structure that maps watch descriptors to pathnames,
the application can determine the filesystem object for which this
event occurred.
`mask` is a bit mask that describes the event that
occurred. In most cases, this field will include one of the bits
specified in the mask specified when the watch was
established. For example, given the `inotify_add_watch()`
call that we showed earlier, if the directory `mydir` was
deleted, `read()` would return an event whose `mask`
field has the `IN_DELETE_SELF` bit set. (By contrast, dnotify
does not generate
an event when a monitored directory is deleted.)

In addition to the various events for which an application may
request notification, there are certain events for which inotify
always generates automatic notifications. The most notable of
these is `IN_IGNORED`, which is generated whenever inotify
ceases to monitor an object. This can occur, for example, because
the object was deleted or the filesystem on which it resides was
unmounted. The `IN_IGNORED` event can be used by the
application to adjust its internal model of what is currently
being monitored. (Again, dnotify has no analog of this event.)

The `name` field is used (only) when an event occurs for a
file inside a monitored directory: it contains the null-terminated
name of the file that triggered this event. The `len` field
indicates the total size of the `name` field, which may be
terminated by multiple null bytes in order to pad out the
`inotify_event` structure to a size that allows successive
structures in the read buffer to be aligned at
architecture-appropriate byte boundaries (typically, multiples of
16 bytes).

The `cookie` field exists to help applications interpret
rename events. When a file is renamed inside (or between) monitored
directories, two events are generated: an `IN_MOVED_FROM`
event for the directory from which the file is moved, and an
`IN_MOVED_TO` event for the directory to which the file is
moved. The first event contains the old name of the file, and the
second event contains the new name. Both events have the same
unique `cookie` value, allowing the application to connect
the two events, and thus work out the old and new name of the
file (a task that is rather difficult with dnotify).
We'll say rather more about rename events in the next
article in this series.

Inotify does not provide recursive monitoring. In other words, if
we are monitoring the directory `mydir`, then we will
receive notifications for that directory as well as all of its
immediate descendants, including subdirectories. However, we will
not receive notifications for events inside the subdirectories.
But, with some effort, it is possible to perform recursive
monitoring by creating watches for each of the subdirectories in a
directory tree. To assist with this task, when a subdirectory is
created inside a monitored directory (or indeed, when any event is
generated for a subdirectory), inotify generates an event that has
the `IN_ISDIR` bit set. This
provides the application with the opportunity to add watches for
new subdirectories.

#### Example program

The code below demonstrates the basic steps in using the
inotify API. The program first creates an inotify instance and
adds watches for all possible events for each of the pathnames
specified in its command line. It then sits in a loop reading
events from the inotify file descriptor and displaying information
from those events (using our `displayInotifyEvent()`, shown
in the full version of the code here).

int main(int argc, char *argv[]) { struct inotify_event *event ... inotifyFd = inotify_init(); /* Create inotify instance */ for (j = 1; j < argc; j++) { wd = inotify_add_watch(inotifyFd, argv[j], IN_ALL_EVENTS); printf("Watching %s using wd %d\n", argv[j], wd); } for (;;) { /* Read events forever */ numRead = read(inotifyFd, buf, BUF_LEN); ... /* Process all of the events in buffer returned by read() */ for (p = buf; p < buf + numRead; ) { event = (struct inotify_event *) p; displayInotifyEvent(event); p += sizeof(struct inotify_event) + event->len; } } }

Suppose that we use this program to monitor two
subdirectories, `xxx` and `yyy`:

$ ./inotify_demo xxx yyy Watching xxx using wd 1 Watching yyy using wd 2

If we now execute the following command:

$ mv xxx/aaa yyy/bbb

we see the following output from our program:

Read 64 bytes from inotify fd wd = 1; cookie =140040; mask = IN_MOVED_FROM name = aaa wd = 2; cookie =140040; mask = IN_MOVED_TO name = bbb

The `mv` command generated an `IN_MOVED_FROM` event
for the `xxx` directory (watch descriptor 1) and
an `IN_MOVED_TO` event for the `yyy` directory
(watch descriptor 2). The two events contained, respectively, the
old and new name of the file. The events also had the
same `cookie` value, thus allowing an application to
connect them.

#### How inotify improves on dnotify

Inotify improves on dnotify in a number of respects. Among the more notable improvements are the following:

- Both directories and individual files can be monitored.
- Instead of signals, applications are notified of filesystem events by reading structured data from a file descriptor created using the API. This approach allows an application to deal with notifications synchronously, and also allows for richer information to be provided with notifications.
- Inotify does not require an application to open file descriptors for each monitored object. Instead, it uses an API-specific handle (the watch descriptor). This avoids the problems of file-descriptor exhaustion and open file descriptors preventing filesystems from being unmounted.
- Inotify provides more information when notifying events. First, it can be used to detect a wider range of events. Second, when the subject of an event is a file inside a monitored directory, inotify provides the name of that file as part of the event notification.
- Inotify provides richer information in its notification of rename events, allowing an application to easily determine the old and new name of the renamed object.
-
`IN_IGNORED`events make it (relatively) easy for an inotify application to maintain an internal model of the currently monitored set of filesystem objects.

#### Concluding remarks

We've briefly seen how inotify improves on dnotify. In the next
article in this series, we look in more detail at inotify,
considering how it can be used in a robust application that
monitors a filesystem tree. This will allow us to see the full
capabilities of inotify, while at the same time discovering some
of its limitations.

| Index entries for this article | |
|---|---|
| Kernel | Development model/User-space ABI |
| Kernel | Dnotify |
| Kernel | Inotify |
| GuestArticles | Kerrisk, Michael |