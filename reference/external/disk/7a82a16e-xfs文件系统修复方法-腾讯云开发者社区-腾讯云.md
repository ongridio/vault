---
title: xfs文件系统修复方法-腾讯云开发者社区-腾讯云
source: https://cloud.tencent.com/developer/article/1579810
kind: external
domain: disk
original_date: 2020-02-02
fetched_at: 2026-05-16
bookmark_title: xfs文件系统修复方法 - 云+社区 - 腾讯云
tags: [external, disk]
---

> [!info] 外部文章 · 自动导入
> 来源：[cloud.tencent.com](https://cloud.tencent.com/developer/article/1579810)
> 原始日期：2020-02-02
> 抓取日期：2026-05-16

# xfs文件系统修复方法-腾讯云开发者社区-腾讯云

首先尝试mount和umount文件系统，以便重放日志，修复文件系统，如果不行，再进行如下操作。

xfs_check /dev/sdd(盘符); echo $? 返回0表示正常

如果幸运的话，会发现没有问题，你可以跳过后续的操作。 该命令将表明会做出什么修改，一般情况下速度很快，即便数据量很大，没理由跳过。

xfs_repair /dev/sdd (ext系列工具为fsck)

根据打印消息，修复失败时： 先执行xfs_repair -L /dev/sdd(清空日志，会丢失文件)，再执行xfs_repair /dev/sdd，再执行xfs_check /dev/sdd 检查文件系统是否修复成功。

说明：-L是修复xfs文件系统的最后手段，慎重选择，它会清空日志，会丢失用户数据和文件。

在执行xfs_repair操作前，最好使用xfs_metadump工具保存元数据，一旦修复失败，最起码可以恢复到修复之前的状态。 xfs_metadump为调试工具，可以不管，跳过。

参考：

http://oss.sgi.com/archives/xfs/2010-06/msg00274.html

http://m.blog.csdn.net/blog/skdkjxy/41648713

本文参与 腾讯云自媒体同步曝光计划，分享自作者个人站点/博客。

原始发表：2015/06/02 ，如有侵权请联系 cloudcommunity@tencent.com 删除

评论

登录后参与评论

推荐阅读

目录