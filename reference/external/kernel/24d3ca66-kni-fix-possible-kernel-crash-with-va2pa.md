---
title: kni: fix possible kernel crash with va2pa
source: http://patches.dpdk.org/patch/50625/
kind: external
domain: kernel
original_date: 2019-02-28
fetched_at: 2026-05-16
bookmark_title: kni: fix possible kernel crash with va2pa - Patchwork
tags: [external, kernel]
---

> [!info] 外部文章 · 自动导入
> 来源：[patches.dpdk.org](http://patches.dpdk.org/patch/50625/)
> 原始日期：2019-02-28
> 抓取日期：2026-05-16

# kni: fix possible kernel crash with va2pa

| Message ID | 20190228073010.49716-1-zhouyates@gmail.com (mailing list archive) |
|---|---|
| State | Superseded, archived |
| Delegated to: | Ferruh Yigit |
| Headers | |
| Series | kni: fix possible kernel crash with va2pa | |

## Checks

| Context | Check | Description |
|---|---|---|
| ci/Intel-compilation | success | Compilation OK |
| ci/mellanox-Performance-Testing | success | Performance Testing PASS |
| ci/intel-Performance-Testing | success | Performance Testing PASS |

## Commit Message

## Comments

diff --git a/kernel/linux/kni/kni_net.c b/kernel/linux/kni/kni_net.c index 7371b6d58..caef8754f 100644 --- a/kernel/linux/kni/kni_net.c +++ b/kernel/linux/kni/kni_net.c @@ -61,18 +61,6 @@ kva2data_kva(struct rte_kni_mbuf *m) return phys_to_virt(m->buf_physaddr + m->data_off); } -/* virtual address to physical address */ -static void * -va2pa(void *va, struct rte_kni_mbuf *m) -{ - void *pa; - - pa = (void *)((unsigned long)va - - ((unsigned long)m->buf_addr - - (unsigned long)m->buf_physaddr)); - return pa; -} - /* * It can be called to process the request. */ @@ -363,7 +351,7 @@ kni_net_rx_normal(struct kni_dev *kni) if (!kva->next) break; - kva = pa2kva(va2pa(kva->next, kva)); + kva = pa2kva(kva->next_pa); data_kva = kva2data_kva(kva); } } @@ -545,7 +533,7 @@ kni_net_rx_lo_fifo_skb(struct kni_dev *kni) if (!kva->next) break; - kva = pa2kva(va2pa(kva->next, kva)); + kva = pa2kva(kva->next_pa); data_kva = kva2data_kva(kva); } } diff --git a/lib/librte_eal/linuxapp/eal/include/exec-env/rte_kni_common.h b/lib/librte_eal/linuxapp/eal/include/exec-env/rte_kni_common.h index 5afa08713..608f5c13f 100644 --- a/lib/librte_eal/linuxapp/eal/include/exec-env/rte_kni_common.h +++ b/lib/librte_eal/linuxapp/eal/include/exec-env/rte_kni_common.h @@ -87,6 +87,10 @@ struct rte_kni_mbuf { char pad3[8] __attribute__((__aligned__(RTE_CACHE_LINE_MIN_SIZE))); void *pool; void *next; + union { + uint64_t tx_offload; + void *next_pa; /**< Physical address of next mbuf. */ + }; }; /* diff --git a/lib/librte_kni/rte_kni.c b/lib/librte_kni/rte_kni.c index 73aeccccf..1aaebcfa1 100644 --- a/lib/librte_kni/rte_kni.c +++ b/lib/librte_kni/rte_kni.c @@ -353,6 +353,17 @@ va2pa(struct rte_mbuf *m) (unsigned long)m->buf_iova)); } +static void * +va2pa_all(struct rte_mbuf *m) +{ + struct rte_kni_mbuf *mbuf = (struct rte_kni_mbuf *)m; + while (mbuf->next) { + mbuf->next_pa = va2pa(mbuf->next); + mbuf = mbuf->next; + } + return va2pa(m); +} + static void obj_free(struct rte_mempool *mp __rte_unused, void *opaque, void *obj, unsigned obj_idx __rte_unused) @@ -550,7 +561,7 @@ rte_kni_tx_burst(struct rte_kni *kni, struct rte_mbuf **mbufs, unsigned num) unsigned int i; for (i = 0; i < num; i++) - phy_mbufs[i] = va2pa(mbufs[i]); + phy_mbufs[i] = va2pa_all(mbufs[i]); ret = kni_fifo_put(kni->rx_q, phy_mbufs, num); @@ -607,6 +618,8 @@ kni_allocate_mbufs(struct rte_kni *kni) offsetof(struct rte_kni_mbuf, pkt_len)); RTE_BUILD_BUG_ON(offsetof(struct rte_mbuf, ol_flags) != offsetof(struct rte_kni_mbuf, ol_flags)); + RTE_BUILD_BUG_ON(offsetof(struct rte_mbuf, tx_offload) != + offsetof(struct rte_kni_mbuf, tx_offload)); /* Check if pktmbuf pool has been configured */ if (kni->pktmbuf_pool == NULL) {