---
title: Networking
source: https://yakking.branchable.com/posts/networking-4-namespaces-and-multi-host-routing/
kind: external
domain: network
author: Richard Maw
original_date: 2015-09-09
fetched_at: 2026-05-16
bookmark_title: Networking - Namespaces
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[yakking.branchable.com](https://yakking.branchable.com/posts/networking-4-namespaces-and-multi-host-routing/)
> 作者：Richard Maw
> 原始日期：2015-09-09
> 抓取日期：2026-05-16

# Networking

# What are network namespaces

Network Namespaces are a Linux feature, that allows different processes to have different views of the network.

Aspects of networking that can be isolated between processes include:

Interfaces

Different processes can connect to addresses on different interfaces.

Routes

Since processes can see different addresses from different namespaces, they also need different routes to connect to networks on those interfaces.

Firewall rules

Firewalls are mostly out of scope for this article, but since firewall rules are dependant on the source or target interfaces, you need different firewall rules in different network namespaces.


# How do you manage network namespaces

When a network namespace is created with the unshare(2) or clone(2) system calls, it is bound to the life of the current process, so if your process exits, the network namespace is removed.

This is ideal for sandboxing processes, so that they have restricted access to the network, as the network namespaces are automatically cleaned up when the process they were isolating the network for no longer exists.

A privileged process can make their network namespace persistent, which is useful for if the network namespace needs to exist when there are no processes in it.

This is what the ip(8) command does when you run

`ip netns add NAME`

.The

`ip netns delete NAME`

command undoes this, and allows the namespace to be removed when the last process using it leaves the namespace.`ip netns list`

shows current namespaces.`$ sudo ip netns add testns $ ip netns list testns $ sudo ip netns delete testns $ ip netns list`

A network namespace is of no use if it has no interfaces, so we can move an existing interface into it with the

`ip link set dev DEVICE netns NAME`

command.`$ ip link show dev usb0 5: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN mode DEFAULT group default qlen 1000 link/ether 02:5e:37:61:60:66 brd ff:ff:ff:ff:ff:ff $ sudo ip link set dev usb0 netns testns $ ip link show dev usb0 Device "usb0" does not exist.`

Network namespaces would be of limited use if we can't run a command in them.

We can run a command in a namespace with the

`ip netns exec NAME COMMAND`

command.`$ sudo ip netns exec testns ip link show dev usb0 5: usb0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000 link/ether 02:5e:37:61:60:66 brd ff:ff:ff:ff:ff:ff5: usb0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000 link/ether 02:5e:37:61:60:66 brd ff:ff:ff:ff:ff:ff`

There is no standard daemon to start persistent network namespaces on-boot, however, the PrivateNetwork= option to systemd service files can be used to start a process in a network namespace on boot.


# Uses of network namespaces

Network namespaces can be used to sandbox processes, so that they can only connect through the interfaces provided.

You could have a machine with two network interfaces, one which connects out to the internet, the other connected to your internal network. You could set up network namespaces such that you can run the ssh server on your internal network, and run a web server in a network namespace that has the internet-facing interface, so if your web server is compromised, it can't connect to your internal network.

docker and systemd-nspawn combine network namespaces with other namespaces to provide containers. docker is primarily interested in application containers, while systemd-nspawn is more interested in system containers, though both technologies can be used for either.

A more novel use of network namespaces is virtual routers, as implemented in OpenStack Neutron.

## Network namespaced routing

We previously mentioned that routes were a property of network namespaces, that virtual ethernet devices have a pair of ends, and that we can move a network interface into another network namespace.

By combining these features, we can construct a series of virtual computers and experiment with routing.

## Two node networking

To start with we're going to add another virtual computer, start a simple echo server on it, and configure the network such that we can connect to it from our host computer.

As we've done in previous articles,
we're going to make the `left`

and `right`

virtual ethernet pair.

```
$ sudo ip link add left type veth peer name right
```


But to demonstrate routing to another machine we need to create one, so we're making another network namespace, as we only care about its networking.

```
$ sudo ip netns add testns
```


We're going to move one half of the virtual ethernet device pair into another namespace, to simulate a physical link between the two virtual computers.

```
$ sudo ip link set dev right netns testns
```


We're going to say that we connect to interfaces in the `testns`

namespace
by connecting to addresses in the `10.248.179.1/24`

subnet.

```
$ sudo ip address add 10.248.179.1/24 dev left
$ sudo ip netns exec testns ip address add 10.248.179.2/24 dev right
```


We're also going to say that there's another network in there
on the `10.248.180.1/24`

subnet.

Rather than having any more complicated network interfaces, we assigning it to the loopback interface.

```
$ sudo ip netns exec testns ip address add 10.248.180.1/24 dev lo
```


Now that we've assigned addresses, we need to bring the interfaces up.

```
$ sudo ip link set dev left up
$ sudo ip netns exec testns ip link set dev right up
$ sudo ip netns exec testns ip link set dev lo up
```


This has created a chain of interfaces linking `left`

→ `right`

→ `lo`

.
Where `left`

is in your namespace,
and `right`

is in `testns`

with its own private `lo`

.

Now we can demonstrate sending messages through the network namespaces,
by creating a server on the `lo`

interface inside `testns`

.

Start an echo server in the test namespace by running this command in a new terminal.

It is *important* to run this in another terminal,
as if you background the process,
netcat may decide it doesn't need to actively output data as it recieves it.

```
$ sudo ip netns exec testns nc -l 10.248.180.1 12345
```


If we were to try to connect to it, we would not be able to send messages.

```
$ nc -v 10.248.180.1 12345
nc: connect to 10.248.180.1 port 12345 (tcp) failed: Network is unreachable
```


This is because your host namespace does not understand how to reach that address, since it is not an address in your network namespace, nor is it an address on any of the subnets in the namespace, and there are no defined rules for how to reach it.

So let's add a rule!

```
$ sudo ip route add 10.248.180.0/24 dev left via 10.248.179.2
```


This says that you can find the `10.248.180.0/24`

subnet
by sending packets to the `10.248.179.2`

address
through the `left`

interface.

You should now be able to type messages into the following netcat command, and see the result in your other terminal.

```
$ nc -v 10.248.180.1 12345
Connection to 10.248.180.1 12345 port [tcp/*] succeeded!
```


## 3 namespaces

This works fine, but in the wider internet you need to connect through long chains of computers before you reach your final destination, so we're going to use 3 network namespaces to represent a longer chain.

We are going to have two extra network namespaces, called `near`

and `far`

.

```
$ sudo ip netns add near
$ sudo ip netns add far
```


We are going to create a chain of network interfaces called `one`

, `two`

, `three`

and `four`

.

```
$ sudo ip link add one type veth peer name two
$ sudo ip link add three type veth peer name four
```


`one`

will be in our network namespace, connecting to `two`

in the `near`

namespace, and `three`

will connect to `four`

in the `far`

namespace.

```
$ sudo ip link set dev two netns near
$ sudo ip link set dev three netns near
$ sudo ip link set dev four netns far
```


This produces a chain `one`

→ `two`

→ `three`

→ `four`

.

To speak to these interfaces, we need to assign address ranges, so for our
host to `near`

link we will use `10.248.1.0/24`

,
for our `near`

to `far`

link we will use `10.248.2.0/24`

,
and for our destination address we will use `10.248.3.1/24`

.

```
$ sudo ip address add 10.248.1.1/24 dev one
$ sudo ip netns exec near ip address add 10.248.1.2/24 dev two
$ sudo ip netns exec near ip address add 10.248.2.1/24 dev three
$ sudo ip netns exec far ip address add 10.248.2.2/24 dev four
$ sudo ip netns exec far ip address add 10.248.3.1/24 dev lo
```


We need to bring these interfaces up.

```
$ sudo ip link set dev one up
$ sudo ip netns exec near ip link set dev two up
$ sudo ip netns exec near ip link set dev three up
$ sudo ip netns exec far ip link set dev four up
$ sudo ip netns exec far ip link set dev lo up
```


Now let's this time start our server in the `far`

namespace,
so we can be sure that we can't connect to it directly from our namespace.

As before, start this echo server in a new terminal.

```
$ sudo ip netns exec far nc -l 10.248.3.1 12345
```


Now let's try to connect to it from our namespace.

```
$ nc -v 10.248.3.1 12345
nc: connect to 10.248.3.1 port 12345 (tcp) failed: Network is unreachable
```


If you try to send a message, it won't arrive.
If you inspect the routing table with `ip route`

,
you will see that there is no match.

The relevant routing rules are:

```
$ ip route
default via 192.168.1.1 dev wlan0 proto static
10.248.1.0/24 dev one proto kernel scope link src 10.248.1.1
```


This says that if you're broadcasting from `10.248.1.1`

and going to
anything in `10.248.1.0/24`

then it goes to the `one`

interface. However
this doesn't match because we want to go to the `four`

interface which
has address `10.248.3.1`

.

We can make this route to our `near`

interface by adding a new rule.

```
$ sudo ip route add 10.248.2.0/24 dev one via 10.248.1.2
```


We can prove this hop works by starting a server in the `near`

namespace
(again, in a separate terminal).

```
$ sudo ip netns exec near nc -l 10.248.2.1 12345
```


We can try to talk to it:

```
$ nc -v 10.248.2.2 12345
```


This doesn't yet work, because it's a bidirectional protocol, so the return route needs to work too.

```
$ sudo ip netns exec near ip route change 10.248.1.0/24 dev two via 10.248.1.1
```


This still doesn't work, since while there's now routing rules, they aren't for any addresses which have desinations in the near network namespace, so by default they get dropped.

To change this we can turn on ip forwarding with:

```
sudo dd of=/proc/sys/net/ipv4/ip_forward <<<1
```


Now we can talk to the `near`

namespace, but we can't yet talk to the `far`

namespace.
We need to add another routing rule for that.

```
$ sudo ip route add 10.248.3.0/24 dev one via 10.248.1.2
```


As before, this means you can route to the `near`

namespace.
However, it doesn't know how to reach the `far`

namespace from there,
so we need to add another routing rule.

```
$ sudo ip netns exec near ip route add 10.248.3.0/24 dev three via 10.248.2.2
```


Now you can reach the `far`

namespace, but it's still not working.
This is because tcp is a bidirectional communication protocol,
and it doesn't know how to send the response to a message from `10.248.1.1`

,
so we need to add another rule.

```
$ sudo ip netns exec far ip route add 10.248.1.0/24 dev four via 10.248.2.1
```


## Conclusion

I think you will agree that it's far too much faff to set up this routing just to talk to a machine on another network.

You are likely thinking that there must be a better way, and you are correct, there's a couple of ways, but we will cover those in later articles.