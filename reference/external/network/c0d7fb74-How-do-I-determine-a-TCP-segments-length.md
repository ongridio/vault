---
title: How do I determine a TCP segment’s length
source: https://osqa-ask.wireshark.org/questions/4178/how-do-i-determine-a-tcp-segments-length
kind: external
domain: network
original_date: 2011-05-01
fetched_at: 2026-05-16
bookmark_title: How do I determine a TCP segment's length - Wireshark Q&A
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[osqa-ask.wireshark.org](https://osqa-ask.wireshark.org/questions/4178/how-do-i-determine-a-tcp-segments-length)
> 原始日期：2011-05-01
> 抓取日期：2026-05-16

# How do I determine a TCP segment’s length

How do I determine a TCP segment's length - Header length + No. Bytes in flight? asked jaden |


3 Answers:


The TCP payload size is calculated by taking the "Total Length" from the IP header (ip.len) and then substract the "IP header length" (ip.hdr_len) and the "TCP header length" (tcp.hdr_len). The "Bytes in Flight" field shows the amount of data that has been sent, but not yet ACKed (seen from the perspective of the point of capture). answered SYN-bit ♦♦ |


You can add columns by right-clicking the fields in the Packet Details pane and select "Apply as Column" from the context menu: Here you can read more about adding and customizing columns. answered joke edited |


answered jayjair |