---
title: Linux Diagnostic Commands Cheatsheet
tags: [reference, cheatsheet, linux]
---

# Linux Diagnostic Commands Cheatsheet

Grouped by the question they answer. Each line is "command — what it
tells you". For deep dives see the `systems/` docs.

## CPU and load

```
top -bn1                    # snapshot top-N processes by CPU
htop                        # interactive (if installed)
uptime                      # load1 / load5 / load15
mpstat -P ALL 1 5           # per-core utilization (sysstat)
pidstat -u 1 5              # per-process CPU sampling
vmstat 1 5                  # CPU + memory + IO + system summary
perf top                    # kernel-level hot symbols (perf)
```

## Memory

```
free -h                     # totals; watch MemAvailable not MemFree
cat /proc/meminfo           # the full breakdown
ps aux --sort=-rss | head   # top RSS consumers
slabtop                     # kernel slab cache
dmesg -T | grep -i oom      # past OOM kills
```

## Disk and filesystem

```
df -h                       # block-level usage
df -i                       # inode usage — easy to forget
du -xh / 2>/dev/null | sort -rh | head -20    # heaviest subdirs
lsblk                       # block device tree
blkid                       # filesystem UUIDs / types
findmnt                     # current mount tree with options
iostat -xz 1 5              # per-device IO stats (sysstat)
iotop -oPa                  # process IO (needs root)
lsof | grep deleted         # held-open deleted files
```

## Network

```
ip addr                     # interfaces
ip route                    # routing table
ip neigh                    # ARP / ND cache
ip -s link show             # per-interface counters
ss -ant                     # IPv4 TCP sockets, no DNS
ss -lnt                     # listening sockets + queue depths
ss -tnp 'state established' # established w/ owning process
ss -s                       # socket summary
nstat -az                   # kernel network counters
tcpdump -nni any host X     # quick packet capture
tcptraceroute               # hop-by-hop tcp probe
mtr <host>                  # continuous traceroute
dig +short <host>           # DNS A lookup
dig +trace <host>           # DNS root-down trace
```

## Firewall / NAT / conntrack

```
nft list ruleset            # nftables rules
iptables -S                 # legacy view (or shim)
iptables -t nat -S          # nat table
conntrack -L | head         # tracked flows
conntrack -S                # conntrack stats (drops, insert_failed)
sysctl net.netfilter.nf_conntrack_max
```

## Processes and limits

```
ps -ef                      # process tree (-f for parents)
ps -eo stat,pid,comm | awk '$1 ~ /^D/'   # D-state (uninterruptible IO)
pstree -p                   # visual process tree
lsof -p <pid>               # open files for a process
cat /proc/<pid>/limits      # rlimits of running process
cat /proc/<pid>/status      # full process state (RSS, threads, ctx switch)
ulimit -a                   # current shell limits
```

## File descriptors

```
cat /proc/sys/fs/file-nr    # used / allocated / max
lsof | awk '{print $2}' | sort | uniq -c | sort -rn | head    # by pid
ls -1 /proc/<pid>/fd | wc -l
```

## Kernel and dmesg

```
dmesg -T                    # kernel ring buffer with timestamps
journalctl -k --since '1h ago'
journalctl -u <service> --since '15 min ago'
journalctl --disk-usage     # journald footprint
journalctl --vacuum-time=7d # trim
```

## Containers

```
docker ps --format 'table {{.ID}}\t{{.Image}}\t{{.Status}}'
docker stats --no-stream
docker system df -v
docker logs --tail 200 <id>
docker inspect <id>
crictl ps                   # CRI view (containerd / cri-o)
ctr -n k8s.io containers ls # containerd direct
```

## Kubernetes

```
kubectl get pods -A -o wide
kubectl get events --sort-by=.lastTimestamp -A | tail -30
kubectl describe pod <name>
kubectl logs <pod> --previous --all-containers
kubectl top pod -A          # metrics-server required
kubectl top node            # metrics-server required
kubectl get nodes -o wide
kubectl describe node <name>
```

## Systemd

```
systemctl status <unit>
systemctl list-units --failed
systemctl show <unit> -p ExecMainStatus,Result,ActiveState
journalctl -u <unit> -f
systemctl cat <unit>        # full unit file (drop-ins included)
```

## Big-picture profile (where to start)

```
# 60-second linux performance walk (Brendan Gregg)
uptime; dmesg | tail; vmstat 1 5; mpstat -P ALL 1 5; pidstat 1 5
iostat -xz 1 5; free -m; sar -n DEV 1 5; sar -n TCP,ETCP 1 5; top
```
