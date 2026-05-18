---
title: Observability, Tracing & Debugging — Curated Reading List
kind: index
domain: observability
fetched_at: 2026-05-18
tags: [index, observability, ebpf, tracing, cilium, kubernetes]
---

> [!info] Reading-list index · canonical observability sources
> The companion to the full-text imports under `reference/external/ebpf/`
> and `reference/external/tracing/`. Cilium, Hubble, and BPF tracing
> material that the AIOps assistant can recommend.

# Cilium / Hubble — networking observability for Kubernetes

The Cilium troubleshooting guide is a single dense page with anchored
sub-sections — most useful as a whole, but the anchors below give the
LLM a precise jump target when the operator's question maps to one
sub-area.

- [Component & cluster health — Kubernetes](https://docs.cilium.io/en/stable/operations/troubleshooting/#kubernetes)
- [Detailed status (`cilium-dbg status`)](https://docs.cilium.io/en/stable/operations/troubleshooting/#detailed-status)
- [Logs](https://docs.cilium.io/en/stable/operations/troubleshooting/#logs)
- [Component & cluster health — generic / on-host](https://docs.cilium.io/en/stable/operations/troubleshooting/#generic)
- [Observing flows with Hubble](https://docs.cilium.io/en/stable/operations/troubleshooting/#observing-flows-with-hubble)
- [Observing flows with Hubble Relay](https://docs.cilium.io/en/stable/operations/troubleshooting/#observing-flows-with-hubble-relay)
- [Cilium connectivity tests](https://docs.cilium.io/en/stable/operations/troubleshooting/#cilium-connectivity-tests)
- [Checking cluster connectivity health](https://docs.cilium.io/en/stable/operations/troubleshooting/#checking-cluster-connectivity-health)
- [Monitoring datapath state (`cilium-dbg monitor`)](https://docs.cilium.io/en/stable/operations/troubleshooting/#monitoring-datapath-state)
- [Handling drop — CT: Map insertion failed](https://docs.cilium.io/en/stable/operations/troubleshooting/#handling-drop-ct-map-insertion-failed)
- [Ensure pod is managed by Cilium](https://docs.cilium.io/en/stable/operations/troubleshooting/#ensure-pod-is-managed-by-cilium)
- [Understand the rendering of your policy](https://docs.cilium.io/en/stable/operations/troubleshooting/#understand-the-rendering-of-your-policy)
- [Policymap pressure and overflow](https://docs.cilium.io/en/stable/operations/troubleshooting/#policymap-pressure-and-overflow)
- [Understanding etcd (kvstore) status](https://docs.cilium.io/en/stable/operations/troubleshooting/#understanding-etcd-status)
- [etcd recovery behavior](https://docs.cilium.io/en/stable/operations/troubleshooting/#recovery-behavior)
- [Cluster Mesh — automatic verification](https://docs.cilium.io/en/stable/operations/troubleshooting/#automatic-verification)
- [Cluster Mesh — manual verification](https://docs.cilium.io/en/stable/operations/troubleshooting/#manual-verification)
- [Cluster Mesh — state propagation](https://docs.cilium.io/en/stable/operations/troubleshooting/#state-propagation)
- [Service Mesh — manual verification of setup](https://docs.cilium.io/en/stable/operations/troubleshooting/#manual-verification-of-setup)
- [Ingress troubleshooting](https://docs.cilium.io/en/stable/operations/troubleshooting/#ingress-troubleshooting)
- [Service Mesh connectivity troubleshooting](https://docs.cilium.io/en/stable/operations/troubleshooting/#connectivity-troubleshooting)
- [Node-to-node traffic is being dropped](https://docs.cilium.io/en/stable/operations/troubleshooting/#node-to-node-traffic-is-being-dropped)
- [Automatic log & state collection (`cilium sysdump`)](https://docs.cilium.io/en/stable/operations/troubleshooting/#automatic-log-state-collection)
- [Single-node bugtool (`cilium-bugtool`)](https://docs.cilium.io/en/stable/operations/troubleshooting/#single-node-bugtool)
- [Debugging information (`cilium-dbg debuginfo`)](https://docs.cilium.io/en/stable/operations/troubleshooting/#debugging-information)

# Performance debugging — Linux

- [Using strace to understand a 10x Java performance improvement — packagecloud](https://blog.packagecloud.io/using-strace-to-understand-a-10x-java-performance-improvement/)
- [Linux Distributions and the Timelines of their Systems — packagecloud](https://blog.packagecloud.io/linux-distributions-and-the-timelines-of-their-systems/)
- [Performance is a Science — Gustavo Duarte](https://manybutfinite.com/post/performance-is-a-science/)

# Methodology

(See also `reference/index/sre-methodology.md` for Google SRE Book Ch. 12
"Effective Troubleshooting" and Ch. 4 "Service Level Objectives".)
