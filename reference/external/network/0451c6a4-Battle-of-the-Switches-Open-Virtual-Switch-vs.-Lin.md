---
title: Battle of the Switches - Open Virtual Switch vs. Linux Bridge
source: https://kumul.us/switches-ovs-vs-linux-bridge-simplicity-rules/
kind: external
domain: network
original_date: 2015-12-18
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[kumul.us](https://kumul.us/switches-ovs-vs-linux-bridge-simplicity-rules/)
> 原始日期：2015-12-18
> 抓取日期：2026-05-16

# Battle of the Switches - Open Virtual Switch vs. Linux Bridge

I’ve gotta come right out and say it: Open Virtual Switch– or OVS– is the wrong switch for Infrastructure as a Service (IaaS).

That’s not to say it’s the wrong switch for virtual, or even as a base layer integration for Software Defined Networks…but it is absolutely the wrong solution for IaaS, whether it be for a managed service provider, an enterprise, or a university.

#### Why OVS?

Open Virtual Switch was initially conceived in a university environment, with a flow-based model providing the development primitive, and a central controller determining what those flows actually looked like. The use of OVS as a virtual switch in the IaaS market (and to be specific, in the OpenStack IaaS market) came about because the resources who supported OVS and its enterprise equivalent understandably wanted to ensure that there was an open model for integrating their services into OpenStack. Along the way, numerous decisions were made that eventually elevated OVS into the “principal” position of virtual switch. Unfortunately, along with this promotion of status came expectations that OVS would provide the best of all possible switching worlds when the anticipated era of the Software-Defined Networking (SDN) took over. The premature anticipation of SDN ubiquity in cloud made OVS a seemingly reasonable bet as “the” switching solution. Anticipation of SDN ubiquity in cloud made OVS a seemingly reasonable bet.

Now, it’s quite possible that if SDN had become the defacto standard for Cloud, the OVS-as-King solution would have made perfect sense. But the complexity of OVS had people longing for the simpler days, the days where they just had a simple bridging technology like Linux Bridge to support their Cloud solutions.

#### Why not Linux Bridge?

So why not Linux Bridge? It has been an alternate switch technology even before OVS was conceived and provides a simple model for interacting with the virtual forwarding layer in the Linux kernel. It has been around since nearly before users realized that it was possible to get a Linux machine to connect two or more physical network port together. Truisms exist for a reason, and one of the truisms in the Cloud systems space is that simple always wins over complicated. Hence, Linux Bridge, although not the new kid on the block, or the newest technology solution, may well become the winner in the Battle of the Virtual Cloud Switches. Linux Bridge, being older and simpler, may have made OVS initially more attractive. Linux Bridge, being older and simpler, may have made OVS initially more attractive. But the champions for the OVS model weren’t willing to give up without a fight. OVS proponents pointed out that Linux Bridge lacked a scaleable tunneling model. True, Linux Bridge supported GRE Tunnels, but not the newer and more scalable VXLAN model. Eventually, the titans of networking became involved in the arguments, and it is therefore probably not too surprising that a complex solution was deemed better than a simple one.

#### Why Linux Bridge will Win the Battle

But let’s remember that truism: simple wins over complicated. Having been through the convolutions required by OVS, large scale production environments are now increasingly moving over to Linux Bridge. OVS just comes with too much complexity and renders issues in the network domain a royal pain in the rear to manage. Certainly, there have been changes to Linux Bridge that have also helped close the gap between projects using OVS and Linux Bridge, including the addition of VXLAN support as a tunnel technology. But in the greater scheme, it’s the simplicity of Linux Bridge that wins the day.

In production, simple wins over complicated; Linux Bridge is increasingly the Cloud switch of choice. In production, simple wins over complicated. Linux Bridge is the #cloud switch of choice. Given that a principal tenet of an “as a service” environment is pooled, shared and elastic resources–which means some level of homogeneity, plus a degree of simplification of the resource being modeled and turned into a service–it’s not surprising that a simple network technology makes more sense than a complex one. And with the introduction of technologies that remove an entire layer of the forwarding domain (bypassing L2 segregation to focus on L3 segregation exclusively), we’ll likely see yet another shift at the simplification process that has been the driving engine behind IaaS scale and enablement.

For now, deploy your IaaS tools with a simple and functional L2 switching layer, and leverage Linux Bridge with the understanding of its simple model for services that meets much of the IaaS market’s needs. No need to complicate things. After all, as Leonardo da Vinci said, “Simplicity is the ultimate sophistication.”


Robert Starmer,

Founder and CTO


Ready to accelerate your development processes? Learn how Git enhances both collaboration and productivity with our new GitLab courses. Learn how Git, when used correctly smooths out code production, packaging, and deployment. For a limited time, qualified teams can get access to TWO free copies of our on-demand classes to try collaboration with GitLab for themselves. Learn more>> https://kumul.us/gitlab-accelerator/.