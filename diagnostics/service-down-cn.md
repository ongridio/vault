---
title: 服务挂了 / systemd 失败怎么排查
kind: howto
tags: [systemd, systemctl, journalctl, 服务, 进程]
applies_to: [edge, manager]
---

# 服务挂了 / systemd 失败怎么排查（中文）

适用：`systemctl status xxx` 显示 `failed` / `activating (auto-restart)` / `deactivating`，
或者业务报错"连不上 xx 服务"。**先看是否还在跑，再看为什么挂，最后看怎么恢复**。

## 第 1 步 — 服务现在什么状态

```bash
systemctl status <unit>          # 当前状态 + 最近几行日志
systemctl is-active <unit>       # active / inactive / failed / activating
systemctl is-enabled <unit>      # enabled / disabled / static
```

状态语义：
- `active (running)` —— 正常
- `active (exited)` —— 一次性任务（oneshot），已完成；非异常
- `inactive (dead)` —— 没在跑，且没人启动
- `activating (auto-restart)` —— **crash 后被 systemd 拉起，进入循环**
- `failed` —— 启动失败 / 异常退出，systemd 不再重试
- `deactivating` —— 正在关闭中

`auto-restart` 循环最致命 —— 业务时断时续，监控不一定连贯告警。看 `systemctl status` 里
`Active:` 行的时间，几秒就重启一次的话肯定是 crash loop。

## 第 2 步 — 看日志找原因

```bash
journalctl -u <unit> -n 100 --no-pager       # 最近 100 行
journalctl -u <unit> --since '30 min ago'    # 时间窗口
journalctl -u <unit> -p err                  # 只看 ERROR 及以上
journalctl -u <unit> -f                      # 实时跟踪（重启过程实时看）
```

应用本身可能有自己的日志路径（不走 systemd 管道）：

```bash
systemctl cat <unit> | grep -E '^StandardOutput|^StandardError'
# 看到 file:/var/log/xxx.log 就跑去看那里
ls -lt /var/log/<app>/*.log | head -5
tail -200 /var/log/<app>/$(ls -t /var/log/<app> | head -1)
```

**重启循环时优先看最早的失败日志**（最早的错误是根因；后续都是连锁反应）：

```bash
journalctl -u <unit> --since '1 hour ago' | grep -iE 'fatal|error|panic|crash' | head -20
```

## 第 3 步 — 常见 crash 模式速诊

| 日志关键词 | 意思 |
|---|---|
| `code=killed, status=9/KILL` + dmesg 有 oom | **被 OOM Killer 杀** → 见 `memory-pressure-cn.md` |
| `code=exited, status=139` | SIGSEGV 段错误，应用 bug / so 版本不匹配 |
| `code=exited, status=137` | SIGKILL（128+9），通常是 OOM 或手动 kill |
| `code=exited, status=143` | SIGTERM（128+15），正常关闭信号 |
| `code=exited, status=255` | 应用自己定义的"未知错误"退出码 |
| `bind: address already in use` | 端口被占（前一个实例没退干净 / 别的服务） |
| `connection refused` to localhost | 依赖服务没起（DB / Redis / 配置中心） |
| `permission denied` | 权限问题（SELinux / AppArmor / 文件 mode） |
| `start-limit-hit` | systemd 限流（默认 5 次/10s）保护，需要 reset |

```bash
# 端口冲突排查
ss -tlnp | grep :<port>                       # 谁在占
lsof -i :<port>                               # 同上，更详细
```

## 第 4 步 — start-limit-hit（systemd 自己拒绝继续重启）

5 次启动失败后 systemd 会停 1 分钟，超 5 次后彻底放弃。表现：
- `systemctl status` 显示 `Active: failed (Result: start-limit-hit)`
- 你 `systemctl start` 一下还是失败

```bash
# 重置限流计数
systemctl reset-failed <unit>
systemctl start <unit>

# 如果想调阈值（unit 文件里）
[Service]
StartLimitIntervalSec=300   # 默认 10s 太短，5min 更合理
StartLimitBurst=10          # 默认 5
Restart=on-failure
RestartSec=5
```

## 第 5 步 — 依赖关系排查

```bash
systemctl list-dependencies <unit>            # 谁该先起
systemctl list-dependencies <unit> --reverse  # 谁依赖它
systemd-analyze critical-chain <unit>         # 启动关键路径耗时
```

如果服务依赖某个 mount / network-online.target，那个底层没就绪也会拉不起来：

```bash
systemctl status network-online.target
systemctl status local-fs.target
mount | grep <expected-mount>
```

## 第 6 步 — 配置 / 二进制变更（最近改过什么）

```bash
# unit 文件最近改动
ls -lt /etc/systemd/system/<unit>.service.d/* /etc/systemd/system/<unit>.service /lib/systemd/system/<unit>.service 2>/dev/null

# 应用配置最近改动
find /etc/<app> -mtime -7 -type f -ls

# 二进制最近改动
ls -lh $(systemctl cat <unit> | grep '^ExecStart=' | awk '{print $1}' | sed 's|ExecStart=||')

# 包升级历史
grep -E "install|upgrade" /var/log/dpkg.log | tail -20      # debian/ubuntu
grep -E "Installed|Updated" /var/log/yum.log* | tail -20     # rhel/centos
```

如果是刚升级完才挂的：
```bash
# 回滚（debian）
apt list --installed <pkg>
apt install <pkg>=<previous-version>
```

## 第 7 步 — 恢复 / 临时绕过

按风险从低到高：

1. **重置 + 重启**：`systemctl reset-failed <unit> && systemctl restart <unit>`
2. **降低限流**：临时把 `StartLimitBurst` 调大，buy time 排查
3. **临时停服把症状切断**：`systemctl stop <unit>`，告诉运营方"已知问题处理中"
4. **降级二进制**：用上一个已知好版本（参考 step 6）
5. **重启 systemd 自身**：`systemctl daemon-reexec`（重读 PID 1，避免 systemd 自己状态异常）

**永远不要直接 `kill -9 <main-pid>` 然后期望 systemd 拉起来** —— 不优雅关闭可能留下脏状态，
影响下一次启动。

## ongrid 工具提示

- 状态查询：直接派 `specialist-ops`，他会跑 `systemctl status` + `journalctl` 组合
- 重启动作：用 `host_restart_service` 工具，**会触发 reviewer 二审**，不要 `host_bash systemctl restart` 绕开
- 大批量重启：永远先一台一台来，**等服务真的回到 active 几分钟再下一台**

## 决策树

| 信号 | 操作 |
|---|---|
| `auto-restart` 循环 | journalctl 找最早 fatal/error 行 → 真根因 |
| `code=killed, status=9` + dmesg oom | 内存问题 → `memory-pressure-cn.md` |
| `bind: address in use` | `ss -tlnp` 找谁占端口 |
| `start-limit-hit` | `reset-failed` + 调 StartLimitBurst |
| 配置/二进制最近改过 | 回滚到上一个版本试 |
| 依赖服务没起 | `list-dependencies` 找底层，先修底层 |

## 参考资料

- [systemd.service(5) — systemd 官方](https://man7.org/linux/man-pages/man5/systemd.service.5.html)
- [Linux 进程状态浅析 — 伯乐在线](http://blog.jobbole.com/93645/)
- [记一次 Kafka 重启失败问题排查 — objcoding](https://objcoding.com/2019/05/06/kafka-restart-failed/)
- vault: `diagnostics/memory-pressure-cn.md`、`diagnostics/disk-full-cn.md`（磁盘满也会让很多服务起不来）
