---
title: How Can the Packet Size Be Greater than the MTU?
source: http://packetbomb.com/how-can-the-packet-size-be-greater-than-the-mtu/
kind: external
domain: network
author: Sridhar
original_date: 2014-08-18
fetched_at: 2026-05-16
bookmark_title: How Can the Packet Size Be Greater than the MTU? | PacketBomb
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[packetbomb.com](http://packetbomb.com/how-can-the-packet-size-be-greater-than-the-mtu/)
> 作者：Sridhar
> 原始日期：2014-08-18
> 抓取日期：2026-05-16

# How Can the Packet Size Be Greater than the MTU?

#### Leave a Comment:

#### (36) comments

Thanks Kary! Another great insight! Oh hey, I recommended packetbomb to a guy on reddit in /r/networking who was looking for some help with a file server performance issue. Hope you don’t mind. Thanks again!

ReplyNice post, especially pinpointing where the packets are picked up and why that is too soon to be exact.

I wrote a blog post that covered the same topic, at http://blog.packet-foo.com/2014/05/the-drawbacks-of-local-packet-captures/

In my humble opinion captures should never be taken on client or server unless you can live with the drawbacks and are aware of them. So I would not complain about LSO or LRO, CRC errors etc. if doing local captures, because that’s just what happens if it is done that way.

Also, I would never write a script to break up packets into MSS sizes. When the source (local capture) is already “artificial” it can only get worse by assuming things that may not have happened that way on the wire. E.g. you can only guess the timings etc. But again, if you can live with the drawbacks, go ahead :-)

Cheers,

Jasper

P.S.: there are tons of guys out there that think they know all about TCP, but give them one simple sequence to track and they fail every single time.

ReplyThanks, Jasper. I dig your site. It’s fantastic.

For the particular issue I was troubleshooting, breaking it up into separate packets was ok, but you’re right, in general not a very good idea.

Listen up, people! When I talk about the packet pros, Jasper is one of them. I’ve seen him present at Sharkfest and he knows his stuff. Make sure you subscribe to his site!

ReplyThanks, your site is very cool, too. I’ll keep coming back ;-)

ReplyHello Jasper,

The link you’re providing is useful a well… And probably right… one thing only is with Middleware guys (I’m from), it’ll be useful to capture the TLS “secrets” as the conversation is running… hence be capable to uncrypt SSL/TLS conversation(s)… I don’t even know how to do it without capturing a libcurl-client SSL-TLS keys… and give it to WireShark to decode SSL/TLS traffic. So in my humble opinion, to strictly follow instructions of https://jimshaver.net/2015/02/11/decrypting-tls-browser-traffic-with-wireshark-the-easy-way/ and https://www.youtube.com/watch?v=hh9SRJpK5hI in order to actually beeing able to decode the whole conversation…

ReplyHi Kary,

Nice text! I think that it is possible to disable TCPoffload in the NIC.

Regards,

Mauricio.

ReplyCool post!

It’s really i was looking for in my trouble.

And answer to the question for disable it in Linux:

ethtool -K vlan563 tso off

To disable it on NIC “vlan563”.

And if you want to see actual status of the TCP offload for the vlan563:

ethtool -k vlan563

Nice article to help clear doubts. It would be great if the pkt dumps mentioned above are attached somewhere so that user can themself see how packets under TSO. One thing not clear to me is when TSO is done, are the original TCP options being copied to all the segments or some changes are done in the same.. I am trying to understand how TSO and MPTCP co-exist (if they).

Thanks again.

Krishna

You could confirm this by doing a capture on both the client and a spanned port of the client interface :)

ReplyHello Kary

Thanks for posting, very useful information. I have looked at a handful of Wireshark traces now and have seen ‘TCP Segment of reassembled PDU’ by the way what does PDU stand for Physical Data Unit?

So basically are you saying is if this offload behaviour is in action, it is impossible to deduce any thing sensible from the TCP sequence/acknowledgement numbers in he normal fashion? or am I misunderstanding that point?

Yes, I would be very interested in the Perl script please (I will likely turn it into a PowerShell script as I am working on Windows)

One last question please

Lets say I have to capture on a Windows Server (as the Cisco guys will not setup a span port for me. I then turn off, offloading on the Windows NIC, If the host at the other end of the connection (storage appliance for example) has offloading enabled, will I also have issues with Seq/Ack numbers.

Thanks very much

Ernest

Wondering how you draw this picture: http://packetbomb.com/wp-content/uploads/2014/08/libpcap-trpy.png

What’s the font and what’s the tool used?

ReplySo, supposing that I don’t have access to the network switch. If I use a third machine with a soft switch between the sender and receiver and configure the soft switch to dump all frames to the hard drive, do you think that would be any different? I’m not even sure if the soft switches available today have that feature. Ignoring, of course, if the traffic is such that it can be captured in software without losing frames.

ReplyThis post saved my life!

It’s well written and easy to understand.

Thanks very much!

One point: I can’t believe that the guy whose seeing the capture for 15 years had never had to follow the sequence numbers…

I came here several months ago and even I’m already doing this….

Anyways, thanks a lot!

ReplyVery nice content Kary. I have a question though.

I have captured the traffic from the vyatta, which is the FW in front of my server. I still see the packages there that are greater than the MTU. Is it possible that the vyatta is also doing the reverse LRO as you explained above?

On the dump I am seeing packages with 2764 bytes, although the MTU is set to 1400 on the interface of my server. How is that possible?

Thanks.

ReplyNice explanation, but I have one question. Exactly what # within the Wireshark capture represents the MTU size.

I’ve been banging my head try to pinpoint which figure is the MTU.

Thank you so much,

Will

Reply[…] http://packetbomb.com/how-can …https://ask.wireshark.org/que …Transmission over the network must not exceed MTU. […]

ReplyHi

I am getting large packets as you described above so as you recommended I changed the setting to the following:

ethtool -k eth0 | grep tcp-segmentation-offload

I am still getting large packets. Am I doing something wrong?

I would love to get the script you mentioned above to see if it will solve the problem. If you could email it to me I’d really appreciate it.

Thank you,

Ben

Hi Kary, thanks to you i’m getting better at packet analysis but except this one (: Hope you will answer my question below.

My question is;

Captured an iso download from web while i study on packet analysis… On wireshark i’m seeing, server sends 2946, 2774, 9698, 13026 bytes packets (headers included)… MTU is 1500 and LRO is disabled on my laptop. Large packets size are varying from 2.9k to 13k of bytes. Don’t fragment flag is set on packets. And packets from server appears in microseconds while 3-way handshake shows 35 ms rtt on wireshark and ping shows average 280+ ms rtt… What possibly cause this stuation? What am i possibly missing?

Thanks in advance…

Hi again, it seems that “generic-receive-offload” was on… Turned it off and big packets gone. (:

ReplyWhat happens to the IP Identifier? Will it be the same for all the packets that are segmented on the NIC level?

What Happens to the Sequence Numbers and Ack numbers ?

Thanks a lot for this article, it clarified something for a client of mine.

ReplyOh boy. So I’m looking at a packet capture here that was taken off of a vmware server inside a large cloud service provider and WireShark is telling me there are packets up to almost 15,000 bytes, I’m thinking WTH! I come across your explanation and I had forgotten all about this. I don’t even remember the last time I saw this, probably because 99.9999% of our captures are done with TAP’s or SPAN’s. Definitely made some notes in onenote and saved the link! Thanks for the explanation Kary!

ReplyHi Kary, I hope you are well. I’m an ex-colleague of yours (and we met in SFO). You might remember me. You mentioned that perl script to break up the packets to the MSS size. I’d be really interested in seeing that. Meanwhile, all the best to you and Packetbomb.

ReplyHi Darren, yes, of course. I’ll never forget beating my head against a throughput issue and you casually asked if it was just hitting the license bandwidth limit. It never occurred to me to think of that and of course that’s what it was. I’ll have to look and see if I still have that script. I must warn you and anyone else, it’s not something that should be used for anything important or trusted in any way, more of an academic exercise.

ReplyThanks Kary much appreciated. Great to hear from you. And for your great presentations at SharkFest Wireshark Developer and User Conference.

Cheers, Daren

ReplyHello,

thanks a lot ! We were scratching our heads as to why on the emitting end the packet was >1500 bytes, but chunked in smaller packets at the receiving end when running wireshark on the server and the client.

While IP fragmentation was set to do not fragment, and was not used anyway.

And the server seemed happy to receive ACKs about TCP packets it did not sent ? We felt we were missing something in wireshark.

Used “ethtool -K bond1 tso off” and then I could see on wireshark the actual TCP fragmenting going on as expected on the server (and switched it back on right after)

So thanks again for this great explanation !

ReplyThank you sir!

I am 2/3 to my CCNP cert so I understand this stuff, yet I don’t know the console commands, nor the various strategies to utilize to efficiently parse the data.

Question– What is the best literature to read to master the Wireshark tool as well as its various methods for strategic application?

Thank you!

-Matt

Reply