---
title: golang map源码详解 - Yiz96
source: https://blog.yiz96.com/golang-map/
kind: external
domain: observability
author: Yiz
original_date: 2017-11-23
fetched_at: 2026-05-16
tags: [external, observability]
---

> [!info] 外部文章 · 自动导入
> 来源：[blog.yiz96.com](https://blog.yiz96.com/golang-map/)
> 作者：Yiz
> 原始日期：2017-11-23
> 抓取日期：2026-05-16

# golang map源码详解 - Yiz96

# golang map源码详解

本文将主要分析一下golang中map的实现原理，并对使用中的常见问题进行讨论。进行分析的golang版本为1.9。

golang中的map是用hashmap作为底层实现的，在github的源码中相关的代码有两处：runtime/hashmap.go定义了map的基本结构和方法，runtime/hashmap_fast.go提供了一些快速操作map的函数。

### map基本数据结构

map的底层结构是hmap（即hashmap的缩写），核心元素是一个由若干个桶（bucket，结构为bmap）组成的数组，每个bucket可以存放若干元素（通常是8个），key通过哈希算法被归入不同的bucket中。当超过8个元素需要存入某个bucket时，hmap会使用extra中的overflow来拓展该bucket。下面是hmap的结构体。

|
1 2 3 4 5 6 7 8 9 10 11 12 13 |
type hmap struct { count int // # 元素个数 flags uint8 B uint8 // 说明包含2^B个bucket noverflow uint16 // 溢出的bucket的个数 hash0 uint32 // hash种子 buckets unsafe.Pointer // buckets的数组指针 oldbuckets unsafe.Pointer // 结构扩容的时候用于复制的buckets数组 nevacuate uintptr // 搬迁进度（已经搬迁的buckets数量） extra *mapextra } |

在extra中不仅有overflow，还有oldoverflow（用于扩容）和nextoverflow（prealloc的地址）。

bucket（bmap）的结构如下

|
1 2 3 4 5 6 7 8 9 10 11 |
type bmap struct { // tophash generally contains the top byte of the hash value // for each key in this bucket. If tophash[0] < minTopHash, // tophash[0] is a bucket evacuation state instead. tophash [bucketCnt]uint8 // Followed by bucketCnt keys and then bucketCnt values. // NOTE: packing all the keys together and then all the values together makes the // code a bit more complicated than alternating key/value/key/value/... but it allows // us to eliminate padding which would be needed for, e.g., map[int64]int8. // Followed by an overflow pointer. } |

- tophash用于记录8个key哈希值的高8位，这样在寻找对应key的时候可以更快，不必每次都对key做全等判断。
**注意后面几行注释，hmap并非只有一个tophash，而是后面紧跟8组kv对和一个overflow的指针，这样才能使overflow成为一个链表的结构。但是这两个结构体并不是显示定义的，而是直接通过指针运算进行访问的。**- kv的存储形式为”key0key1key2key3…key7val1val2val3…val7″，这样做的好处是：在key和value的长度不同的时候，节省padding空间。如上面的例子，在map[int64]int8中，4个相邻的int8可以存储在同一个内存单元中。如果使用kv交错存储的话，每个int8都会被padding占用单独的内存单元（为了提高寻址速度）。

hmap的结构差不多如图

### map的访问

以mapaccess1为例，部分无关代码被修改。

|
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31 32 33 34 35 36 37 38 39 40 41 42 43 44 45 46 47 |
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer { // do some race detect things // do some memory sanitizer thins if h == nil || h.count == 0 { return unsafe.Pointer(&zeroVal[0]) } if h.flags&hashWriting != 0 { // 检测是否并发写，map不是gorountine安全的 throw("concurrent map read and map write") } alg := t.key.alg // 哈希算法 alg -> algorithm hash := alg.hash(key, uintptr(h.hash0)) m := bucketMask(h.B) b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize))) // 如果老的bucket没有被移动完，那么去老的bucket中寻找 （增长部分下一节介绍） if c := h.oldbuckets; c != nil { if !h.sameSizeGrow() { // There used to be half as many buckets; mask down one more power of two. m >>= 1 } oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize))) if !evacuated(oldb) { b = oldb } } // 寻找过程：不断比对tophash和key top := tophash(hash) for ; b != nil; b = b.overflow(t) { for i := uintptr(0); i < bucketCnt; i++ { if b.tophash[i] != top { continue } k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize)) if t.indirectkey { k = *((*unsafe.Pointer)(k)) } if alg.equal(key, k) { v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize)) if t.indirectvalue { v = *((*unsafe.Pointer)(v)) } return v } } } return unsafe.Pointer(&zeroVal[0]) } |

### map的增长

随着元素的增加，在一个bucket链中寻找特定的key会变得效率低下，所以在插入的元素个数/bucket个数达到某个阈值（当前设置为6.5，实验得来的值）时，map会进行扩容，代码中详见 hashGrow函数。首先创建bucket数组，长度为原长度的两倍 newbuckets, nextOverflow := makeBucketArray(t, h.B+bigger) ，然后替换原有的bucket，原有的bucket被移动到oldbucket指针下。

扩容完成后，每个hash对应两个bucket（一个新的一个旧的）。oldbucket不会立即被转移到新的bucket下，而是当访问到该bucket时，会调用growWork方法进行迁移，growWork方法会将oldbucket下的元素rehash到新的bucket中。随着访问的进行，所有oldbucket会被逐渐移动到bucket中。

但是这里有个问题：如果需要进行扩容的时候，上一次扩容后的迁移还没结束，怎么办？在代码中我们可以看到很多”again”标记，会不断进行迁移，知道迁移完成后才会进行下一次扩容。


### 使用中常见问题

Q：删除掉map中的元素是否会释放内存？

A：不会，删除操作仅仅将对应的tophash[i]设置为empty，并非释放内存。若要释放内存只能等待指针无引用后被系统gc


Q：如何并发地使用map？

A：map不是goroutine安全的，所以在有多个gorountine对map进行写操作是会panic。多gorountine读写map是应加锁（RWMutex），或使用sync.Map（1.9新增，在下篇文章中会介绍这个东西，总之是不太推荐使用）。


Q：map的iterator是否安全？

A：map的delete并非真的delete，所以对迭代器是没有影响的，是安全的。