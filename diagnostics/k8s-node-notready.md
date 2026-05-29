---
title: Kubernetes Node NotReady
kind: howto
tags: [kubernetes, node, notready, kubelet, cni, runtime, kanode]
applies_to: [edge, manager]
---

# Kubernetes Node NotReady

Use when a node goes `NotReady` and its pods stop scheduling / get
rescheduled. NotReady means the kubelet stopped reporting healthy to the
API server. **The node's conditions + the kubelet's own logs localize
it**: kubelet down, container runtime down, CNI not ready, or the node
lost network to the control plane.

| Symptom | Probable cause |
|---|---|
| NotReady, kubelet not running | kubelet crashed / OOM / disk full on node |
| `container runtime is down` | containerd/docker dead or socket gone |
| `cni plugin not initialized` | CNI daemonset not running / misconfigured |
| NotReady but node up | node↔apiserver network / cert / clock issue |

## Step 1 — Node condition + kubelet status

```bash
kubectl describe node <node> | sed -n '/Conditions:/,/Addresses:/p'
# On the node:
systemctl status kubelet
journalctl -u kubelet -n 100 --no-pager | tail -60
```

The `Ready` condition's `Message` and the kubelet log's last error are
the diagnosis. `KubeletNotReady` with a CNI/runtime message points at
those subsystems.

## Step 2 — Container runtime + CNI

```bash
# Runtime alive?
systemctl status containerd        # or docker
crictl info >/dev/null 2>&1 && echo "runtime OK" || echo "runtime DOWN"
# CNI ready? (kubelet refuses Ready until network plugin is up)
ls /etc/cni/net.d/                 # is there a CNI config?
kubectl -n kube-system get pods -o wide | grep <node> | grep -iE 'cni|calico|flannel|cilium'
```

No CNI config / a crashed CNI pod on the node = `cni plugin not
initialized` → NotReady even though the kubelet itself is fine.

## Step 3 — Node resource starvation

```bash
# A node out of disk/memory takes the kubelet (and runtime) down with it
df -h / /var/lib/kubelet /var/lib/containerd
free -h; dmesg -T | grep -i oom | tail
```

Disk-full on the node FS is a top cause: the kubelet/runtime can't write
state and the node flips NotReady. (See
`diagnostics/k8s-pod-evicted-node-pressure.md`.)

## Step 4 — Node ↔ control-plane connectivity

```bash
# From the node, reach the API server
curl -k https://<apiserver>:6443/healthz
# Clock skew breaks TLS to the apiserver
timedatectl | grep -E 'synchronized|System clock'
```

If the kubelet is healthy but can't reach (or authenticate to) the API
server — network partition, expired kubelet cert, or clock skew — the
node reports NotReady from the control plane's view.

## Decision tree

| Signal | Action |
|---|---|
| kubelet dead | restart kubelet; check why (OOM/disk/config) |
| runtime down | restart containerd/docker; check its logs/socket |
| CNI not initialized | fix/restart the CNI daemonset on the node |
| node disk/memory full | reclaim space — `k8s-pod-evicted-node-pressure.md` |
| can't reach apiserver | fix node↔CP network; renew kubelet cert; fix clock |

## References

- [Debugging Kubernetes Nodes — kubernetes.io](https://kubernetes.io/docs/tasks/debug/debug-cluster/)
- [Kubelet — Kubernetes docs](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/)
- vault: `diagnostics/k8s-pod-evicted-node-pressure.md`, `diagnostics/clock-skew-ntp.md`, `systems/container/kubernetes.md`
