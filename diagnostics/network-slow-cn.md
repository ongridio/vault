---
title: 网络慢 / 连不通怎么排查
kind: howto
tags: [网络, tcp, dns, mtu, conntrack, iptables, 丢包]
applies_to: [edge, manager]
---

# 网络慢 / 连不通怎么排查（中文）

适用：用户反馈"网慢"、"打不开"、"超时"、"偶发失败"。**先定位是哪一段慢/断**（DNS / TCP /
TLS / 应用），再用对应工具深挖。打从一开始就 tcpdump 是浪费 —— 90% 的问题在前 3 步定位完。

## 第 1 步 — 是哪一段慢 / 断

```bash
# 一行覆盖 DNS + TCP + TLS + HTTP 各阶段耗时
curl -o /dev/null -s -w \
  'dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n' \
  https://<target>/
```

读法：
- `dns` 大 —— DNS 解析慢（>50ms 一般就有问题）→ 第 2 步
- `connect - dns` 大 —— TCP 握手慢 / 不通 → 第 3 步
- `tls - connect` 大 —— TLS 握手慢（证书 / 中间盒）→ 第 4 步
- `ttfb - tls` 大 —— 服务端应用响应慢，不是网络问题
- `total - ttfb` 大 —— 下载慢（带宽 / 拥塞窗口）

## 第 2 步 — DNS 排查

```bash
# 走哪个 resolver
cat /etc/resolv.conf
resolvectl status                                  # systemd-resolved
nscd -g 2>/dev/null || true                        # nscd 缓存

# 直接挑 resolver 测
dig +short +time=2 +tries=1 @8.8.8.8 example.com
dig +short +time=2 +tries=1 @<内网DNS>  example.com

# 看是否走了运营商劫持的 resolver
dig +short @<resolver> myip.opendns.com    # 看返回 IP 是不是预期出口
```

常见问题：
- `/etc/resolv.conf` 指了挂掉的 resolver；多 nameserver 时 glibc 默认 5s 才切换
- 内网 DNS 服务有 split-horizon 但解析有空洞
- IPv6 AAAA 解析超时拖慢 dual-stack（应用代码常见）
- DNS-over-HTTPS 中间盒重写 / 限流

## 第 3 步 — TCP 不通 / 慢

```bash
# 端口探活（不动 ping，因为 ICMP 经常被禁）
nc -zvw 3 <ip> <port>                              # TCP 探测
# 或者
timeout 3 bash -c "</dev/tcp/<ip>/<port>" && echo OK || echo FAIL

# 看路径丢包 / 延迟
mtr -rwn -c 100 <ip>                               # 100 包统计每跳
ping -i 0.2 -c 100 <ip>                            # 200ms 间隔 100 包
```

`mtr` 输出列：`Loss% Snt Last Avg Best Wrst StDev`。判断：
- 某跳 Loss% 高 + 后续跳 Loss% 都恢复 → ICMP 速率限制，不是真丢包
- 某跳之后 Loss% 持续高 → **真丢包从这跳开始**

```bash
# 看本机 TCP 健康度（系统级）
ss -s                                              # estab / closed / orphaned
nstat -z | grep -iE 'retrans|fail|drop|listen'
# 关键指标：
#   TcpRetransSegs/TcpOutSegs > 1% = 有路径质量问题
#   TcpExtListenOverflows > 0 = 服务端 accept 队列满
```

## 第 4 步 — TLS 握手慢 / 报错

```bash
# 详细看 TLS 流程 + 证书
openssl s_client -connect <host>:<port> -servername <sni> 2>&1 | head -40

# 看 cipher 协商
openssl s_client -connect <host>:<port> -tls1_2 -cipher ECDHE-ECDSA-AES128-GCM-SHA256 </dev/null 2>&1 | grep -iE 'protocol|cipher'
```

常见 TLS 慢因：
- 客户端 / 服务端协议不匹配（强制 TLS 1.3 但对方只支持 1.2）
- 中间盒做 TLS inspection（多一跳握手）
- OCSP stapling 拖慢首次握手（应用层可关）
- 证书链不完整，客户端在 fetch intermediate

## 第 5 步 — 一段时间内偶发：本机 TCP 微观状态

每个 TCP 连接的细节：

```bash
ss -tin                                            # 所有 TCP + 内部 info
ss -tin state established '( dport = :443 )'      # 仅 443 的 ESTABLISHED
```

`ss -tin` 输出每个连接下面一行：
```
cubic wscale:7,7 rto:200 rtt:12/4 mss:1448 cwnd:10 ssthresh:7 bytes_sent:... unacked:... retrans:0/3
```

重点字段：
- `retrans:X/Y` —— Y 累计重传段数。Y 增长快 = 路径质量差
- `unacked > 0` 长期 —— 对端没在 ACK
- `cwnd:1` 持续 —— 拥塞窗口塌到 1，刚从严重丢包恢复
- `rcv_space` 持续缩小 —— 接收方处理不过来

## 第 6 步 — netfilter / conntrack 层

```bash
# conntrack 满了？
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max
dmesg -T | grep conntrack | tail
```

`count` 接近 `max` → 新连接被丢，老连接没事 → 见 `conntrack-table-full.md`。

```bash
# iptables / nftables 在丢包吗
iptables -L -n -v | grep -E 'DROP|REJECT'
nft list ruleset | grep -iE 'drop|reject' | head

# 实时统计 drop（用 nflog 或 nf_trace）
sudo nft monitor trace                             # 谨慎，输出量大
```

## 第 7 步 — 容器 / overlay 网络

```bash
# 物理网卡丢包
ip -s link show
# 看 RX errors / RX dropped / TX errors —— 任意非 0 都要看

# 容器 veth 状态
docker exec <ctr> ip -s link show
docker network inspect <net>

# 是不是 MTU 不一致（overlay 必踩坑）
ip link show <iface> | grep mtu
# 物理 1500, 容器 1450 等 —— 容器内 ping -M do -s 1418 <peer> 看大包是否通
ping -M do -s 1418 <peer>
ping -M do -s 1472 <peer>  # 1500 - 28 = 1472，物理上限
```

VXLAN / Geneve overlay 默认会扣 50 字节 → 内层 MTU 应是 1450，否则大包静默丢。

## 第 8 步 — 兜底：抓包（先窄后宽，不要漫无目的）

```bash
# 只抓特定主机 + 端口；保持包小（snap len 96 看 header 够了）
sudo tcpdump -nn -i any -s 96 'host <peer> and tcp port <port>' -w /tmp/cap.pcap -G 60 -W 1
# -G 60 -W 1: 写一份 60s 就停，不会爆磁盘

# 离线分析
tshark -r /tmp/cap.pcap -q -z io,stat,1            # 每秒包数 / 字节数
tshark -r /tmp/cap.pcap -Y 'tcp.analysis.retransmission' | head
tshark -r /tmp/cap.pcap -Y 'tcp.analysis.zero_window'
tshark -r /tmp/cap.pcap -Y 'tcp.flags.reset == 1'
```

## ongrid 工具提示

- 派 `specialist-network`：自带 `host_probe_dns / host_probe_tcp / host_probe_http` 三件套，比手敲 nc/dig/curl 快
- 路由 / 网卡查看：`host_bash` 跑 `ip route`、`ip -s link`、`iptables -L -n`
- conntrack 状态：`host_bash conntrack -L | wc -l`

## 决策树

| 信号 | 操作 |
|---|---|
| `curl -w` dns 字段大 | 第 2 步 DNS |
| `connect` 字段大或失败 | nc / mtr 找哪段断 |
| `tls` 字段大 | openssl s_client 看握手 |
| `ttfb` 大 | 后端慢，不是网络 |
| mtr 某跳之后持续丢 | 该跳是真问题，对接网络 |
| `ss -s` retrans 多 + 多 peer | 本机出口路径质量差 |
| conntrack count 接近 max | `conntrack-table-full.md` |
| 大包丢小包通 | MTU 问题（overlay 网络高发） |
| `iptables -L -v` 某 chain 包数飙高 | 防火墙规则在丢 |

## 参考资料

- [TCP Tuning Guide — kernel.org](https://www.kernel.org/doc/html/latest/networking/ip-sysctl.html)
- [Linux 上的基础网络设备详解 — IBM developerWorks](https://www.ibm.com/developerworks/cn/linux/1310_xiawc_networkdevice/)
- [tcp(7) — man7.org](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [What comes after iptables: nftables — RedHat Developers](https://developers.redhat.com/blog/2017/01/05/iptables-soon-to-be-replaced)
- vault: `systems/network/tcp-stack.md`、`systems/network/conntrack.md`、`diagnostics/tcp-retransmit-loss.md` (English)、`diagnostics/conntrack-table-full.md` (English)
