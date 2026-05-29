---
title: Kubernetes Pod Networking / CNI & Overlay Failures
kind: howto
tags: [kubernetes, cni, overlay, vxlan, pod-network, kube-proxy, service]
applies_to: [edge, manager]
---

# Kubernetes Pod Networking / CNI & Overlay Failures

Use when pods can't reach each other, can't reach Services, or DNS inside
the cluster fails. Pod networking has layers — **isolate which one
breaks**: pod→pod same node (bridge/veth), pod→pod cross node (CNI
overlay/routing), pod→Service (kube-proxy iptables/ipvs), pod→DNS
(CoreDNS). Test bottom-up.

| Symptom | Probable cause |
|---|---|
| pod→pod cross-node fails, same-node ok | CNI overlay/routing broken (VXLAN, BGP, MTU) |
| pod→Service ClusterIP fails | kube-proxy rules missing / wrong mode |
| in-cluster DNS fails | CoreDNS down / pod resolv.conf / ndots |
| large transfers hang cross-node | overlay MTU not reduced for encapsulation |

## Step 1 — Same-node vs cross-node pod reachability

```bash
# From a pod, ping another pod by IP (same node, then different node)
kubectl exec <podA> -- ping -c2 <podB-ip>
# Which nodes are they on?
kubectl get pods -o wide
```

Same-node works but cross-node fails → the CNI overlay/routing between
nodes is the problem (Step 2). Both fail → local CNI/veth on the node.

## Step 2 — CNI / overlay health

```bash
kubectl -n kube-system get pods -o wide | grep -iE 'calico|flannel|cilium|weave|cni'
kubectl -n kube-system logs <cni-pod-on-node> --tail=50
# Overlay MTU: encapsulation eats bytes — large packets black-hole if MTU wrong
ip link show | grep -E 'vxlan|cni|flannel|cali|cilium' | grep mtu
```

A crashed CNI pod on a node isolates that node's pods. Cross-node *large*
transfers hanging while small ones work = overlay MTU not reduced for the
VXLAN/encap header (see `diagnostics/mtu-blackhole.md`).

## Step 3 — pod→Service (kube-proxy)

```bash
kubectl -n kube-system get pods | grep kube-proxy
kubectl -n kube-system logs <kube-proxy-pod> --tail=40
# On the node: are the Service rules programmed?
iptables-save 2>/dev/null | grep <service-clusterip>     # iptables mode
ipvsadm -Ln 2>/dev/null | grep <service-clusterip>       # ipvs mode
```

Missing Service rules = kube-proxy not programming them (crashed,
wrong mode, or a CNI that replaces kube-proxy like Cilium isn't healthy).

## Step 4 — in-cluster DNS

```bash
kubectl exec <pod> -- nslookup kubernetes.default
kubectl -n kube-system get pods -l k8s-app=kube-dns
kubectl exec <pod> -- cat /etc/resolv.conf      # ndots:5, cluster search domain
```

CoreDNS down, or the classic `ndots:5` + 5s-timeout slowness, breaks
name-based connectivity even when IP connectivity is fine (see
`diagnostics/dns-resolution-failure.md`).

## Decision tree

| Signal | Action |
|---|---|
| cross-node pod→pod fails | CNI overlay/routing — check CNI pods, node routes, BGP/VXLAN |
| large cross-node hangs | overlay MTU too high — set to underlay − encap overhead |
| pod→Service fails | kube-proxy not programming rules — restart/fix mode |
| in-cluster DNS fails | CoreDNS health / resolv.conf / ndots — DNS playbook |
| one node's pods isolated | CNI pod crashed on that node — restart it |

## References

- [Kubernetes — Cluster Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Debug Services — Kubernetes docs](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- vault: `diagnostics/mtu-blackhole.md`, `diagnostics/dns-resolution-failure.md`, `diagnostics/k8s-node-notready.md`, `systems/container/kubernetes.md`
