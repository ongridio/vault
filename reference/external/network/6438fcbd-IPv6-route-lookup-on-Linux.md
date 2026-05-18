---
title: IPv6 route lookup on Linux
source: https://vincent.bernat.ch/en/blog/2017-ipv6-route-lookup-linux
kind: external
domain: network
author: Vincent Bernat
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, network]
---

> [!info] External article · imported reference
> Source: [vincent.bernat.ch](https://vincent.bernat.ch/en/blog/2017-ipv6-route-lookup-linux)
> Author: Vincent Bernat
> License: cc-by-sa
> Fetched: 2026-05-18

# IPv6 route lookup on Linux

TL;DR

With its implementation of IPv6 routing tables using radix trees,
Linux offers subpar performance (450 ns for a full view — 40,000 routes)
compared to IPv4 (50 ns for a full view — 500,000 routes) but fair memory usage
(20 MiB for a full view).

In a previous article, we had a look at [IPv4 route lookup on Linux](/en/blog/2017-ipv4-route-lookup-linux "IPv4 route lookup on Linux"). Let’s
see how different IPv6 is.

# Lookup trie implementation

Looking up a prefix in a routing table comes down to finding the **most specific
entry** matching the requested destination. A common structure for this task is
the [trie](https://en.wikipedia.org/wiki/Trie "Wikipedia article about trie"), a tree structure where each node has its parent as prefix.

With IPv4, Linux uses a [level-compressed trie](/en/blog/2017-ipv4-route-lookup-linux#lookup-with-a-level-compressed-trie "IPv4 route lookup on Linux: lookup with a level-compressed trie") (or *LPC-trie*), providing
good performance with low memory usage. For IPv6, Linux uses a more
classic [radix tree](/en/blog/2017-ipv4-route-lookup-linux#lookup-with-a-path-compressed-trie "IPv4 route lookup on Linux: lookup with a path-compressed trie") (or *Patricia trie*). There are three reasons for not
sharing:

- The IPv6 implementation (introduced in Linux 2.1.8, 1996) predates the IPv4
  implementation based on LPC-tries (in Linux 2.6.13, [commit 19baf839ff4a](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=19baf839ff4a8daa1f2a7400897094fc18e4f5e9 "ipv4: Add LC-Trie FIB lookup algorithm")).
- The feature set is different. Notably, IPv6 supports source-specific
  routing[1](#sidenote-ssr) (since Linux 2.1.120, 1998).
- The IPv4 address space is denser than the IPv6 address
  space. Level-compression is therefore quite efficient with IPv4. This may not
  be the case with IPv6.

1

For a given destination prefix, it’s possible to attach source-specific
prefixes:

```
ip -6 route add 2001:db8:1::/64 \
  from 2001:db8:3::/64 \
  via fe80::1 \
  dev eth0
```

Lookup is first done on the destination address, then on the source address. ❦

The trie in the below illustration encodes 6 prefixes:

Radix tree for a small routing table. Some nodes indicate how many additional input bits to skip. The black nodes contain actual route entries.

For a more in-depth explanation on the different ways to encode a routing table
into a trie and a better understanding of radix trees, see
the [explanations for IPv4](/en/blog/2017-ipv4-route-lookup-linux#route-lookup-in-a-trie "IPv4 route lookup on Linux: route lookup in a trie").

The following figure shows the in-memory representation of the previous radix
tree. Each node corresponds to a [`struct fib6_node`](https://elixir.bootlin.com/linux/v4.12.3/source/include/net/ip6_fib.h#L60 "Linux: struct fib6_node definition"). When
a node has the `RTN_RTINFO` flag set, it embeds a pointer to
a [`struct rt6_info`](https://elixir.bootlin.com/linux/v4.12.3/source/include/net/ip6_fib.h#L98 "Linux: struct rt6_info definition") containing information about the
next-hop.

In-memory representation of an IPv6 routing table in Linux. For simplicity, many fields are omitted. 2001:db8:2::/121 prefix is an ECMP route. When several non-ECMP routes match a prefix, they are linked through the dst field (not shown).

The [`fib6_lookup_1()`](https://elixir.bootlin.com/linux/v4.12.3/source/net/ipv6/ip6_fib.c#L1120 "Linux: fib6_lookup_1() definition") function walks the radix tree in two
steps:

1. walking down the tree to locate the potential candidate, and
2. checking the candidate and, if needed, backtracking until a match.

Here is a slightly simplified version without source-specific routing:

```
static struct fib6_node *fib6_lookup_1(struct fib6_node *root,
                                       struct in6_addr  *addr)
{
    struct fib6_node *fn;
    __be32 dir;

    /* Step 1: locate potential candidate */
    fn = root;
    for (;;) {
        struct fib6_node *next;
        dir = addr_bit_set(addr, fn->fn_bit);
        next = dir ? fn->right : fn->left;
        if (next) {
            fn = next;
            continue;
        }
        break;
    }

    /* Step 2: check prefix and backtrack if needed */
    while (fn) {
        if (fn->fn_flags & RTN_RTINFO) {
            struct rt6key *key;
            key = fn->leaf->rt6i_dst;
            if (ipv6_prefix_equal(&key->addr, addr, key->plen)) {
                if (fn->fn_flags & RTN_RTINFO)
                    return fn;
            }
        }

        if (fn->fn_flags & RTN_ROOT)
            break;
        fn = fn->parent;
    }

    return NULL;
}
```

# Caching

While IPv4 lost its route cache in Linux 3.6 ([commit 5e9965c15ba8](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=5e9965c15ba88319500284e590733f4a4629a288 "Merge removing route cache-related code")), IPv6
still has a caching mechanism. However cache entries are directly put in the
radix tree instead of a distinct structure.

Since Linux 2.1.30 (1997) and until Linux 4.2 ([commit 45e4fd26683c](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=45e4fd26683c9a5f88600d91b08a484f7f09226a "ipv6: Only create RTF_CACHE routes after encountering pmtu exception")), almost
any successful route lookup inserts a cache entry in the radix tree. For
example, a router forwarding a ping between `2001:db8:1::1` and `2001:db8:3::1`
would get these two cache entries:

```
$ ip -6 route show cache
2001:db8:1::1 dev r2-r1  metric 0
    cache
2001:db8:3::1 via 2001:db8:2::2 dev r2-r3  metric 0
    cache
```

These entries are cleaned up by the [`ip6_dst_gc()`](https://elixir.bootlin.com/linux/v4.12.3/source/net/ipv6/route.c#L1736 "Linux: ip6_dst_gc() definition") function
controlled by the following parameters:

```
$ sysctl -a | grep -F net.ipv6.route
net.ipv6.route.gc_elasticity = 9
net.ipv6.route.gc_interval = 30
net.ipv6.route.gc_min_interval = 0
net.ipv6.route.gc_min_interval_ms = 500
net.ipv6.route.gc_thresh = 1024
net.ipv6.route.gc_timeout = 60
net.ipv6.route.max_size = 4096
net.ipv6.route.mtu_expires = 600
```

The garbage collector is triggered at most every 500 ms when there are more than
1024 entries or at least every 30 seconds. The garbage collection won’t run for
more than 60 seconds, except if there are more than 4096 routes. When running,
it will first delete entries older than 30 seconds. If the number of cache
entries is still greater than 4096, it will continue to delete more recent
entries (but no more recent than 512 jiffies, which is the value of
`gc_elasticity`) after a 500 ms pause.

Starting from Linux 4.2 ([commit 45e4fd26683c](https://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git/commit/?id=45e4fd26683c9a5f88600d91b08a484f7f09226a "ipv6: Only create RTF_CACHE routes after encountering pmtu exception")), only a PMTU exception would
create a cache entry. A router doesn’t have to handle these exceptions, so only
hosts would get cache entries.[2](#sidenote-ddos) And they should be pretty rare. Martin KaFai
Lau, from Facebook, [explains](https://lore.kernel.org/netdev/1432353366-2296465-1-git-send-email-kafai@fb.com/ "[PATCH net-next v5 00/11] ipv6: Only create RTF_CACHE route after encountering pmtu exception"):

> Out of all IPv6 `RTF_CACHE` routes that are created, the percentage that has a
> different MTU is very small. In one of our end-user facing proxy server, only
> 1k out of 80k `RTF_CACHE` routes have a smaller MTU. For our DC traffic, there
> is no MTU exception.

Here is what a cache entry with a PMTU exception looks like:

```
$ ip -6 route show cache
2001:db8:1::50 via 2001:db8:1::13 dev out6  metric 0
    cache  expires 573sec mtu 1400 pref medium
```

# Performance

We consider three distinct scenarios:

Excerpt of an Internet full view
:   In this scenario, Linux acts as an edge router attached to the default-free
    zone. Currently, the size of such a routing table is a little bit
    above [40,000 routes](https://bgp.potaroo.net/v6/as2.0/ "AS131072 IPv6 BGP Table Data").

/48 prefixes spread linearly with different densities
:   Linux acts as a core router inside a datacenter. Each customer or rack gets
    one or several /48 networks, which need to be routed around. With a density of
    1, /48 subnets are contiguous.

/128 prefixes spread randomly in a fixed /108 subnet
:   Linux acts as a leaf router for a /64 subnet with hosts getting their IP using
    autoconfiguration. It is assumed all hosts share the same OUI and therefore,
    the first 40 bits are fixed. In this scenario, neighbor reachability
    information for the /64 subnet is converted into routes by some external
    process and redistributed among other routers sharing the same subnet.[3](#sidenote-neigh)

3

This is quite different from the classic scenario where Linux acts as a
gateway for a /64 subnet. In this case, the neighbor subsystem stores the
reachability information for each host and the routing table only contains a
single /64 prefix. ❦

## Route lookup performance

With the help of a small [kernel module](https://github.com/vincentbernat/network-lab/blob/master/lab-routes-ipv6/kbench_mod.c "Kernel module to bench ip6_route_output_flags() function in various conditions"), we can accurately benchmark[4](#sidenote-bench)
the [`ip6_route_output_flags()`](https://elixir.bootlin.com/linux/v4.12.3/source/net/ipv6/route.c#L1215 "Linux: ip6_route_output_flags() definition") function and
correlate the results with the radix tree size:

Maximum depth and lookup time depending on the number of routes inserted. The x-axis scale is logarithmic. The error bars represent the median absolute deviation.

4

The measurements are done in a virtual machine with one vCPU and no
neighbors. The host is an [Intel Core i5-4670K](https://www.intel.com/content/www/us/en/products/sku/75048/intel-core-i54670k-processor-6m-cache-up-to-3-80-ghz/specifications.html "Intel® Core™ i5-4670K Processor") running at 3.7 GHz during
the experiment (CPU governor set to performance). The benchmark is
single-threaded. Many lookups are performed and the result reported is the
median value. Timings of individual runs are computed from the TSC. ❦

Getting meaningful results is challenging due to the size of the address
space. None of the scenarios have a fallback route and we only measure time for
successful hits.[5](#sidenote-hits) For the full view scenario, only the range from
`2400::/16` to `2a06::/16` is scanned (it contains more than half of the
routes). For the /128 scenario, the whole /108 subnet is scanned. For the /48
scenario, the range from the first /48 to the last one is scanned. For each
range, 5000 addresses are picked semi-randomly. This operation is repeated until
we get 5000 hits or until 1 million tests have been executed.

5

Most of the packets in the network are expected to be routed to a
destination. However, this also means the backtracking code path is not used
in the /128 and /48 scenarios. Having a fallback route gives far different
results and makes it difficult to ensure we explore the address space
correctly. ❦

The relation between the maximum depth and the lookup time is incomplete and I
cannot explain the difference in performance between the different densities of
the /48 scenario.

We can extract two important performance points:

- With a **full view**, the lookup time is **450 ns**. This is almost ten times
  the budget for forwarding at 10 Gbps — which is about 50 ns.
- With an almost **empty routing table**, the lookup time is **150 ns**. This
  is still over the time budget for forwarding at 10 Gbps.

With IPv4, the lookup time for an almost empty table was 20 ns while the lookup
time for a full view (500,000 routes) was a bit above 50 ns. How to explain such
a difference? First, the maximum depth of the IPv4 LPC-trie with 500,000 routes
was 6, while the maximum depth of the IPv6 radix tree for 40,000 routes is 40.

Second, while both IPv4’s [`fib_lookup()`](https://elixir.bootlin.com/linux/v4.12.3/source/include/net/ip_fib.h#L334 "Linux: fib_lookup() definition") and
IPv6’s [`ip6_route_output_flags()`](https://elixir.bootlin.com/linux/v4.12.3/source/net/ipv6/route.c#L1215 "Linux: ip6_route_output_flags() definition") functions have a
fixed cost implied by the evaluation of routing rules, IPv4 has several
optimizations when the rules are left unmodified.[6](#sidenote-ipv6opt) These optimizations
are removed on the first modification. If we cancel these optimizations, the
lookup time for IPv4 is impacted by about 30 ns. This still leaves a 100 ns
difference with IPv6 to be explained.

Let’s compare how time is spent in each lookup function. Here is
a [CPU flamegraph](https://www.brendangregg.com/FlameGraphs/cpuflamegraphs.html "CPU Flame Graphs") for IPv4’s [`fib_lookup()`](https://elixir.bootlin.com/linux/v4.12.3/source/include/net/ip_fib.h#L334 "Linux: fib_lookup() definition"):

&#128444; IPv4 route lookup flamegraph

CPU flame graph for IPv4 route lookup function. The local and main tables are unmerged and routing rules are evaluated. The x-axis is the time share (100% is 65 ns). The graph is interactive.

Only 50% of the time is spent in the actual route lookup. The remaining time is
spent evaluating the routing rules (about 30 ns). This ratio is dependent on the
number of routes we inserted (only 1000 in this example). It should be noted the
`fib_table_lookup()` function is executed twice: once with the local routing
table and once with the main routing table.

The equivalent flamegraph for
IPv6’s [`ip6_route_output_flags()`](https://elixir.bootlin.com/linux/v4.12.3/source/net/ipv6/route.c#L1215 "Linux: ip6_route_output_flags() definition") is depicted below:

&#128444; IPv6 route lookup flamegraph

CPU flame graph for IPv6 route lookup function. The x-axis is the time share (100% is 300 ns). The graph is interactive.

Here is an approximate breakdown on the time spent:

- 50% is spent in the route lookup in the *main* table;
- 15% is spent in handling locking (IPv4 is using the more efficient RCU mechanism);
- 5% is spent in the route lookup of the *local* table; and
- most of the remaining is spent in routing rule evaluation (about 100 ns).[7](#sidenote-compileout)

Why is the evaluation of routing rules less efficient with IPv6? Again, I
do not have a definitive answer.

## History

Update (2017-11)

This section has been moved to its [own page](/en/blog/2017-performance-progression-ipv6-route-lookup-linux "Performance progression of IPv6 route lookup on Linux").

## Insertion performance

Another interesting performance-related metric is the insertion time. Linux is
able to insert a full view in less than two seconds. For some reason, the
insertion time is not linear above 50,000 routes and climbs very fast to 60
seconds for 500,000 routes.

Insertion time (system time) for a given number of routes. The x-axis scale is logarithmic. The blue line is a linear regression.

Despite its more complex insertion logic, the IPv4 subsystem is able to insert 2
million routes in less than 10 seconds.

# Memory usage

Radix tree nodes (`struct fib6_node`) and routing information (`struct
rt6_info`) are allocated with the [slab allocator](https://en.wikipedia.org/wiki/Slab_allocation "Slab allocation on Wikipedia").[8](#sidenote-percpu) It is therefore
possible to extract the information from `/proc/slabinfo` when the kernel is
booted with the `slab_nomerge` flag:

8

There is also per-CPU pointers allocated directly (4 bytes per entry
per CPU on a 64-bit architecture). We ignore this detail. ❦

```
# sed -ne 2p -e '/^ip6_dst/p' -e '/^fib6_nodes/p' /proc/slabinfo | cut -f1 -d:
♯  name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab>
fib6_nodes         76101  76104     64   63    1
ip6_dst_cache      40090  40090    384   10    1
```

In the above example, the used memory is 76104×64+40090×384 bytes (about
20 MiB). The number of `struct rt6_info` matches the number of routes while the
number of nodes is roughly twice the number of routes:

Number of nodes in the radix tree depending on the number of routes. The x-axis scale is logarithmic. The blue line is a linear regression.

The memory usage is therefore quite predictable and reasonable, as even a small
single-board computer can support several full views (20 MiB for each):

Memory usage depending on the number of routes inserted. The x-axis scale is logarithmic. The blue line is a linear regression.

The LPC-trie used for IPv4 is more efficient: when 512 MiB of memory is needed
for IPv6 to store 1 million routes, only 128 MiB are needed for IPv4. The
difference is mainly due to the size of `struct rt6_info` (336 bytes) compared
to the size of IPv4’s `struct fib_alias` (48 bytes): IPv4 puts most information
about next-hops in `struct fib_info` structures that are shared with many
entries.

---

The takeaways from this article are:

- upgrade to Linux 4.2 or more recent to avoid excessive caching;
- route lookups are noticeably slower compared to IPv4 (by an order of magnitude);
- `CONFIG_IPV6_MULTIPLE_TABLES` option incurs a fixed penalty of 100 ns by lookup; and
- memory usage is fair (20 MiB for 40,000 routes).

Compared to IPv4, IPv6 in Linux does not foster the same interest, notably in
terms of optimizations. Hopefully, things are changing as its adoption and use
“at scale” are increasing.
