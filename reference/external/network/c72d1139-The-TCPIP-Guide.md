---
title: The TCP/IP Guide
source: http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm
kind: external
domain: network
original_date: 2005-09-20
fetched_at: 2026-05-16
bookmark_title: The TCP/IP Guide - TCP Operational Overview and the TCP Finite State Machine (FSM)
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[tcpipguide.com](http://tcpipguide.com/free/t_TCPOperationalOverviewandtheTCPFiniteStateMachineF-2.htm)
> 原始日期：2005-09-20
> 抓取日期：2026-05-16

# The TCP/IP Guide

| ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||
|
TCP Operational Overview and the TCP Finite State Machine (FSM)(Page 2 of 3)
In the case of TCP, the finite state machine can be considered to describe the “life stages” of a connection. Each connection between one TCP device and another begins in a null state where there is no connection, and then proceeds through a series of states until a connection is established. It remains in that state until something occurs to cause the connection to be closed again, at which point it proceeds through another sequence of transitional states and returns to the closed state. The full description of the states,
events and transitions in a TCP connection is lengthy and complicated—not
surprising, since that would cover much of the entire TCP standard.
For our purposes, that level of detail would be a good cure for insomnia
but not much else. However, a Table 151 briefly describes each of the TCP states in a TCP connection, and also describes the main events that occur in each state, and what actions and transitions occur as a result. For brevity, three abbreviations are used for three types of message that control transitions between states, which correspond to the TCP header flags that are set to indicate a message is serving that function. These are: A*SYN:**synchronize*message, used to initiate and establish a connection. It is so named since one of its functions is to synchronizes sequence numbers between devices.A*FIN:**finish*message, which is a TCP segment with the*FIN*bit set, indicating that a device wants to terminate the connection.An*ACK:**acknowledgment*, indicating receipt of a message such as a*SYN*or a*FIN*.
Again, I have not shown every possible transition, just the ones normally followed in the life of a connection. Error conditions also cause transitions but including these would move us well beyond a “simplified” state machine. The FSM is also illustrated in Figure 210, which you may find easier for seeing how state transitions occur.
Tap tap… still awake? Okay, I guess even with serious simplification, that FSM isn't all that simple. It may seem a bit intimidating at first, but if you take a few minutes with it, you can get a good handle on how TCP works. The FSM will be of great use in making sense of the connection establishment and termination processes later in this section—and conversely, reading those sections will help you make sense of the FSM. So if your eyes have glazed over completely, just carry on and try coming back to this topic later.
Home - Table Of Contents - Contact Us The TCP/IP Guide (http://www.TCPIPGuide.com)Version 3.0 - Version Date: September 20, 2005 © Copyright 2001-2005 Charles M. Kozierok. All Rights Reserved. Not responsible for any loss resulting from the use of this site. |