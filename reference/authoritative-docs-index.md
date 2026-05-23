---
title: 权威技术文档参考索引
tags: [reference, index, documentation, external]
---

# 权威技术文档参考索引

排查/设计时**指向官方权威源**的索引。这里只放**链接 + 适用场景**,不是正文——
要查具体内容请打开链接,或用 `web_search` 拉取最新页面(外部站点更新快,实时检索
比内置快照更可靠)。

> **许可说明(给运维 + 给想离线化的人)**
> - 🔗 **专有/版权**:只可链接引用,**不可爬取再分发**(Cisco / Intel / AMD / ARM /
>   VMware / AWS / Azure / GCP / Juniper / Arista / Elastic 等)。
> - 📥 **开放许可**:可在**组织知识库**(注册 repo 或上传)里离线化,需遵守各自
>   LICENSE / 署名(RFC、Linux Kernel Docs、TLDP、FreeBSD/OpenBSD Handbook、
>   Arch Wiki=CC-BY-SA、Wikipedia=CC-BY-SA、GitHub awesome-* 看各自 LICENSE)。
> - **平台内置 vault 只放一方精选 playbook**,不灌外部正文(避免版权 + 稀释检索)。

## 操作系统 / Linux
- 📥 **Arch Wiki** — https://wiki.archlinux.org — 最全面的 Linux 文档(不限 Arch);命令、服务、内核参数排查首选
- 📥 **Gentoo Wiki** — https://wiki.gentoo.org — 编译/内核配置/底层细节
- 📥 **Linux Kernel Documentation** — https://docs.kernel.org — 内核子系统/参数/sysfs 权威
- 📥 **TLDP (Linux Documentation Project)** — https://tldp.org — 经典 HOWTO
- 📥 **FreeBSD Handbook** — https://docs.freebsd.org/en/books/handbook/ — BSD 系统权威
- 📥 **OpenBSD FAQ** — https://www.openbsd.org/faq/ — 安全导向 BSD
- 🔗 Debian Wiki https://wiki.debian.org · Ubuntu Wiki https://wiki.ubuntu.com — 发行版特定管理

## 系统架构 / CPU
- 🔗 **Intel SDM** — https://www.intel.com/.../intel-sdm.html — x86 指令/MSR/分页权威
- 🔗 **AMD Developer Guides** — https://www.amd.com/en/developer.html
- 🔗 **ARM Architecture** — https://developer.arm.com/documentation
- 📥 **RISC-V Specs** — https://riscv.org/technical/specifications/ — 开放 ISA 规范

## 网络
- 📥 **RFC Editor** — https://www.rfc-editor.org — IETF 协议标准(TCP/IP/DNS/TLS 行为的最终依据)
- 📥 **Wireshark Wiki** — https://wiki.wireshark.org — 抓包/协议字段分析
- 🔗 **Cisco** https://www.cisco.com/c/en/us/support/ · **Juniper** https://www.juniper.net/documentation/ · **Arista** https://www.arista.com/en/support/product-documentation — 厂商设备/命令
- 🔗 NetworkLessons https://networklessons.com — 协议教程

## 存储
- 📥 **Ceph** https://docs.ceph.com · **ZFS (OpenZFS)** https://openzfs.github.io/openzfs-docs/ · **MinIO** https://min.io/docs/ — 分布式/文件系统/对象存储
- 📥 **LVM** https://man7.org/linux/man-pages/man8/lvm.8.html · **NFS** https://nfs.sourceforge.net/ · **open-iscsi** http://www.open-iscsi.com/

## 虚拟化 / 容器
- 📥 **KVM** https://www.linux-kvm.org/page/Documents · **QEMU Wiki** https://wiki.qemu.org
- 📥 **Docker** https://docs.docker.com · **Kubernetes** https://kubernetes.io/docs/ · **Podman** https://docs.podman.io
- 🔗 **VMware** https://docs.vmware.com · **Proxmox VE** https://pve.proxmox.com/wiki/ · **LXD** https://documentation.ubuntu.com/lxd/

## 云
- 🔗 **AWS** https://docs.aws.amazon.com · **Azure** https://learn.microsoft.com/azure/ · **GCP** https://cloud.google.com/docs · **Cloudflare** https://developers.cloudflare.com
- 📥 **OpenStack** https://docs.openstack.org · **Terraform Registry** https://registry.terraform.io · **Pulumi** https://www.pulumi.com/docs/

## 大数据 / 流处理
- 📥 **Spark** https://spark.apache.org/docs/latest/ · **Hadoop** https://hadoop.apache.org/docs/stable/ · **Kafka** https://kafka.apache.org/documentation/ · **Flink** https://nightlies.apache.org/flink/flink-docs-stable/
- 📥 **ClickHouse** https://clickhouse.com/docs · **Druid** https://druid.apache.org/docs/ · **Airflow** https://airflow.apache.org/docs/
- 🔗 **Elasticsearch** https://www.elastic.co/guide/ — 检索/分析(注意 Elastic License)

## DevOps / SRE / 事件响应
- 📥 **Google SRE Books** — https://sre.google/books/ — SRE 经典(网页版可读;判 SLO/错误预算/oncall)
- 🔗 **PagerDuty Incident Response** https://response.pagerduty.com/ · **Atlassian Incident Handbook** https://www.atlassian.com/incident-management/handbook — 事件响应流程
- 📥 **System Design Primer** — https://github.com/donnemartin/system-design-primer — 系统设计/扩展性

## 用法提示(给 Agent)
- 回答运维/排查问题时,**先 `query_knowledge` 查本系统精选 playbook**;需要协议/内核/
  厂商命令的**权威细节**再引这里的官方链接,或 `web_search` 取最新页面。
- 不要凭这篇索引"编造"具体内容——它只给"去哪查",正文以官方源为准。
