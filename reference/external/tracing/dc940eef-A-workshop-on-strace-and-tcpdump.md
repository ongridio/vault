---
title: A workshop on strace and tcpdump
source: https://jvns.ca/blog/2016/05/06/a-workshop-on-strace-and-tcpdump/
kind: external
domain: tracing
author: Julia Evans
license: cc-by-sa
fetched_at: 2026-05-18
tags: [external, tracing]
---

> [!info] External article · imported reference
> Source: [jvns.ca](https://jvns.ca/blog/2016/05/06/a-workshop-on-strace-and-tcpdump/)
> Author: Julia Evans
> License: cc-by-sa
> Fetched: 2026-05-18

# A workshop on strace and tcpdump

This week at work, I ran a workshop on tcpdump and strace. a couple of people on Twitter asked about it so here are some notes. This is mostly just so I can reuse them more easily next time, but maybe you will also find it interesting. The notes are a bit sparse and it’s very unclear that anybody other than me will find them legible or useful, but I decided to put them here anyway.

I basically did a bunch of live demos of how to use tcpdump & strace, and then took questions & comments as people had them. I ran it in an hour, which I think was fine for people who already had some familiarity with the tools, but really aggressive if you’re learning from scratch. Will do that differently next time.

Why use tcpdump? We do a lot of web development; almost all of our services talk to each other with network requests. So a tool that can spy on network requests can be a really good general-purpose debugging tool. I’ve used this quite a bit at work and it’s been great.

I didn’t really explain what TCP was, which seemed okay.

Ask everyone to install Wireshark at the beginning of the workshop. Wireshark is really easy to install on OS X now! Nobody had trouble with this.

**step 1: laptop demo**

- start `python -mSimpleHTTPServer 8080`
- run `curl localhost:8080/404`
- now that we have some simple network traffic on my laptop, we can look at it with tcpdump!
- run `sudo tcpdump` by itself. That’s a lot of output, and it’s a little difficult to read. Don’t worry!
- talk about network interfaces, here we actually need to run `tcpdump -i lo0` or `tcpdump -i any` to see the local traffic
- talk a little bit about what the output here means, but note that it’s kind of difficult to understand
- run `tcpdump -A port 8080 -i any`. `port 8080` is the same as `src port 8080 or dst port 8080`; a really great shortcut. the pcap filter syntax can be a little difficult to remember at first, all I’ve need to really know so far is how to filter by port and IP. `-A` shows us the contents of the packets. We can see the GET request, the content type, and the response in tcpdump’s output! This is really cool!
- point out that Python’s SimpleHTTPServer is writing out each header line in a separate packet, and that this makes no sense.

**step 2: look at some QA network traffic**

- ssh to a dev machine
- run tcpdump on some relevant network traffic, talk about what that network traffic actually means and how you can tell (by looking at the port, knowing what services run on which ports, looking at the hostname the packets are going to)

**step 3: tcpdumping for performance**

this section is about how to debug performance problems using tcpdump!

- find a service in production that makes HTTP requests
- talk about packet captures in production (it’s generally safe to do, just be careful to not accidentally fill up your disk if you’re trying to do packet capture on video streaming or something, and be aware that there may be customer data in there)
- “now we’re going to get timing information for those HTTP requests!”
- run `tcpdump port $correct_port -w output.pcap`
- press ctrl+c when you feel like you have enough data
- “pcap files are like chocolate cookies – every packet analysis tool understands how to read them. So you can write the pcap file on the server and then copy it over for offline analysis”
- copy the file over to my laptop, get everyone else to copy over the file as well
- WIRESHARK TIME!!! =D =D =D
- “as you’ve seen, tcpdump output is a little difficult to interpret and search. some people are really good at that but IMO it’s easier to stick with a basic knowledge of tcpdump and do more advanced stuff in Wireshark”
- open up the pcap file in Wireshark.

Wireshark features to demo:

- searching by HTTP status (try THAT with tcpdump!)
- right clicking on a packet to get every other packet in that TCP conversation
- click statistics -> conversations in the menu and you can sort all TCP sessions by duration to find performance problems
- you can colour packets by type (TCP/UDP/whatever) to more easily visually see what’s going on
- show where the packet timings show up, and talk through how you can use this to diagnose whether the client or the server is the problem (if the client sends a packet and then the server takes 5 seconds to reply to it, that’s AWESOME EVIDENCE)
- ask other people in the room with experience for their favorite Wireshark features and tactics to get information about it. people had really great suggestions.

that’s all for tcpdump!

**step 1: system calls & how to read strace output**

- talk about what a system call is “the API for your operating system”
- run strace on a small program like `ls`
- talk through what the parts of the output mean (“this is the system call, this is a file descriptor, these are the arguments”)
- explain what happens in `strace ls` (first you have `execve(ls)`, then the dynamic linker happens, the first part of the strace output is always the same, then you see stuff that’s more specific to `ls`, and at the end you have `exit`)

**step 2: getting configuration files**

- Find a Java program which has some configuration
- “we have no idea how it’s configured! How will we ever find out?”
- STRACE OBVIOUSLY
- run `strace -e open -o strace_output.txt the_java_program` (I used a Hadoop program). `-e` means “this system call” and `-o` writes the output to a file
- it turns out that this actually doesn’t work if the Java program starts child processes – you usually want to run `strace -f`.
- Run `strace -f` instead
- grep the output file for `.xml` because practically every java program is configured with xml
- we find our configuration file! we are winners
- mention looking at calls to `write`

**step 3: attaching to a running program, and strace a CPU-only program**

run

```
while True:
    pass
```

and then attach to it with `sudo strace -p`. When you’re attaching, you always have to run as root. This program doesn’t have any system calls!

If you want another cool demo of stracing a running program, `find` is a good example, run `find /` and then attach to it with strace – it’ll show you which files `find` is looking at right now!

**step 4: stracing to understand a performance problem**

- (secretly, beforehand) make a tiny flask server that responds to `GET /hello` with ‘hi!’ after sleeping for 2 seconds
- run a small bash script that just runs curl in a loop
- the script is slow! But why is it slow?
- run `strace` on the script, and see how you can see it pause really clearly on the `wait`. Talk about other system calls you’ll often see strace pause (select)
- now is a really good time to mention that STRACING PRODUCTION PROGRAMS WILL SLOW THEM DOWN AND YOU NEED TO BE VERY CAREFUL. Sometimes you do it anyway if the process is a mess
- talk about what to do if you don’t know what a system call means (what’s `wait4`? You can run `man 2 wait4` on any system to get the man page for that system call)

**done!**

- Break for questions.
- mention that I wrote a [zine about strace](http://jvns.ca/blog/2015/04/14/strace-zine/) which is an ok basic reference

I thought this went pretty well, especially given that I prepared it only 2 hours in advance. I think I’ll do it again sometime! I want to get better at doing workshops and talks at least 2-3 times because preparing good material is so hard, and I always learn so much about the talk/workshop the first time I give it.

If you want to adapt this workshop for your cool friends who you want know about strace or tcpdump, you could! Most likely this is way too sketchy for anybody else to use but me.
