---
title: PromQL Snippets
tags: [reference, promql, prometheus]
---

# PromQL Snippets

Copy-paste recipes for common questions. Always inspect labels in your
deployment before pasting — metric names vary by exporter version.

## Host (node_exporter)

### CPU utilization (%)

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### CPU iowait (%)

```promql
avg by (instance) (rate(node_cpu_seconds_total{mode="iowait"}[5m])) * 100
```

### Memory used (%) — uses MemAvailable, not MemFree

```promql
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100
```

### Swap used (%)

```promql
(1 - node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes) * 100
```

### Filesystem used (%) — excludes virtual fs

```promql
100 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay|squashfs"}
       / node_filesystem_size_bytes{fstype!~"tmpfs|overlay|squashfs"}) * 100
```

### Inodes used (%)

```promql
100 - (node_filesystem_files_free / node_filesystem_files) * 100
```

### Disk IO time (ms/s) — closest to %util

```promql
rate(node_disk_io_time_seconds_total[5m]) * 1000
```

### Network throughput (bytes/sec)

```promql
sum by (instance, device) (rate(node_network_receive_bytes_total[5m]))
sum by (instance, device) (rate(node_network_transmit_bytes_total[5m]))
```

### TCP retransmit ratio

```promql
rate(node_netstat_Tcp_RetransSegs[5m])
  / rate(node_netstat_Tcp_OutSegs[5m])
```

### Load1 per core

```promql
node_load1 / on(instance) count by (instance) (node_cpu_seconds_total{mode="idle"})
```

## RED method (Rate / Errors / Duration)

Given a request counter `http_requests_total{method, route, status}`:

### Rate per route

```promql
sum by (route) (rate(http_requests_total[5m]))
```

### Error rate (%) per route

```promql
100 *
  sum by (route) (rate(http_requests_total{status=~"5.."}[5m]))
  / sum by (route) (rate(http_requests_total[5m]))
```

### p99 latency from a histogram

```promql
histogram_quantile(0.99,
  sum by (le, route) (rate(http_request_duration_seconds_bucket[5m])))
```

### Burn-rate alert (SLO 99.9% over 30d)

```promql
# Fast burn (5min)
(sum(rate(http_requests_total{status=~"5.."}[5m]))
   / sum(rate(http_requests_total[5m]))) > (14.4 * 0.001)

# Slow burn (1h)
(sum(rate(http_requests_total{status=~"5.."}[1h]))
   / sum(rate(http_requests_total[1h]))) > (6 * 0.001)
```

## Kubernetes (kube-state-metrics + cAdvisor)

### Pod restart count climbing

```promql
rate(kube_pod_container_status_restarts_total[15m]) > 0
```

### Pod stuck Pending > 15min

```promql
kube_pod_status_phase{phase="Pending"} == 1
  and on (namespace, pod)
  (time() - kube_pod_created) > 900
```

### Container OOM-killed

```promql
increase(kube_pod_container_status_terminated_reason{reason="OOMKilled"}[15m]) > 0
```

### Container CPU throttled (cgroup-level)

```promql
rate(container_cpu_cfs_throttled_periods_total[5m])
  / rate(container_cpu_cfs_periods_total[5m]) > 0.25
```

### Container memory near limit

```promql
(container_memory_working_set_bytes
  / container_spec_memory_limit_bytes) > 0.9
```

### Node not ready

```promql
kube_node_status_condition{condition="Ready", status="true"} == 0
```

### Persistent volume claim near full

```promql
(1 - kubelet_volume_stats_available_bytes
   / kubelet_volume_stats_capacity_bytes) > 0.9
```

## Prometheus self-monitoring

```promql
# Targets that aren't being scraped
up == 0

# Scrape duration p99
histogram_quantile(0.99,
  rate(scrape_duration_seconds_bucket[5m]))

# Active series (per Prometheus)
prometheus_tsdb_head_series

# Remote-write backlog
prometheus_remote_storage_samples_pending

# Failed compactions
rate(prometheus_tsdb_compactions_failed_total[15m]) > 0
```

## Anti-patterns

```promql
# Wrong: rate on a gauge — produces noise
rate(node_memory_MemAvailable_bytes[5m])     # ❌

# Wrong: rate window too small for scrape interval
rate(http_requests_total[1m])                # ❌ if scrape_interval=15s

# Wrong: by (instance) on alerts you want per-cluster
sum by (instance) (rate(errors_total[5m]))   # ❌ if you wanted cluster total
```
