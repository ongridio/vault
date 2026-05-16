---
title: How to Run and Read a Traceroute | InMotion Hosting
source: https://www.inmotionhosting.com/support/website/how-to/read-traceroute
kind: external
domain: network
author: Carrie Smaha
original_date: 2025-06-11
fetched_at: 2026-05-16
bookmark_title: How to Read a Traceroute | InMotion Hosting
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[www.inmotionhosting.com](https://www.inmotionhosting.com/support/website/how-to/read-traceroute)
> 作者：Carrie Smaha
> 原始日期：2025-06-11
> 抓取日期：2026-05-16

# How to Run and Read a Traceroute | InMotion Hosting

**Traceroute** (or **tracert** on Windows) is a essential network diagnostic tool that maps the exact path data packets take from your local computer to a destination server, such as your website hosted on an InMotion Hosting VPS or dedicated server. It measures the round-trip time (latency) at each intermediate router (“hop”) along the route and helps identify where slowdowns, packet loss, or connectivity issues occur.

When your website loads slowly, times out, or becomes unreachable, support teams often request a **ping** and **traceroute** report. Understanding how to run and interpret these tools empowers you to quickly determine whether the problem lies with your local network/ISP, an intermediate network, or the hosting server itself.

This thorough guide explains how traceroute works, provides platform-specific instructions, teaches accurate interpretation of results, covers common issues, and includes best practices for effective troubleshooting on InMotion Hosting servers as of 2026.

## How Traceroute Works

Traceroute operates by sending packets (typically ICMP on Windows or UDP on Linux/macOS) with incrementally increasing **Time To Live (TTL)** values in the IP header.

- The TTL starts at 1. The first router decrements it to 0, drops the packet, and returns an
**ICMP Time Exceeded**message containing its IP address. - The process repeats with TTL=2, TTL=3, etc., until the packets reach the destination or hit the maximum hop limit (usually 30).
- For each hop, traceroute sends
**three probes**by default and records the**round-trip time (RTT)**in milliseconds for each response.

This creates a detailed “map” of the network path, revealing latency at every step. Unlike a simple **ping** (which only tests the final destination), traceroute pinpoints the exact location of delays or failures.

**Key limitations**:

- Results can vary over time due to dynamic routing.
- Some routers/firewalls block or deprioritize ICMP/UDP responses for security, resulting in asterisks (
`*`

) that do not always indicate real problems. - Traceroute measures diagnostic packet latency, which may differ slightly from actual web traffic (TCP) due to routing policies or processing priorities.

## When to Use Traceroute

Run a traceroute when you experience:

- Slow website loading
- Intermittent timeouts or 502/504 errors
- High latency compared to other sites
- Connection problems that ping alone cannot diagnose

Combine it with **ping** for a complete picture. InMotion Hosting provides a free Traceroute Parser tool to help analyze results automatically.

## How to Run a Traceroute (and Ping)

Always run both **ping** and **traceroute** to the same target (e.g., your domain or server IP).

InMotion Hosting has a free tool that can automatically and consistently analyze the results of a traceroute test. If you run the traceroute test as described below, you can use our traceroute parser tool here to see if you may be experiencing a connectivity problem.

### On Windows (tracert)

- Press
**Windows key + R**, type cmd, and press Enter to open Command Prompt. - Run the ping test (let it complete 4–5 replies or press Ctrl+C to stop):
`ping example.com`

- Run the traceroute:
`tracert example.com`


(Replace example.com with your actual domain or IP address.)

To copy results: Right-click in the Command Prompt window, select **Mark**, highlight the text, and press Enter.

### On macOS or Linux (traceroute)

- Open
**Terminal**(Applications → Utilities → Terminal on macOS). - Run the ping test (stop after 4–5 replies with Ctrl+C):
`ping example.com`

- Run the traceroute:
`traceroute example.com`


**Useful options**:

`traceroute -n example.com`

— Disable DNS resolution for faster, cleaner output.`traceroute -I example.com`

— Use ICMP mode (sometimes bypasses blocks).`mtr example.com`

— Install**mtr**(My Traceroute) for a continuous, real-time version that combines traceroute and ping (highly recommended for ongoing issues).

### Advanced Alternative: MTR (My Traceroute)

MTR runs traceroute repeatedly and displays live statistics including packet loss percentage, average latency, and standard deviation. It is superior for spotting intermittent problems.

**Install on Linux/Ubuntu:**

`sudo apt install mtr`


**On AlmaLinux:**

`sudo dnf install mtr`


**Run:**

`mtr example.com`


Press **q** to quit.

## How to Read and Interpret Traceroute Output

Here is a sample Windows tracert output:

```
Tracing route to example.com [10.10.242.22]
over a maximum of 30 hops:
1 <1 ms <1 ms <1 ms 172.16.10.2
2 * * * Request timed out.
3 2 ms 2 ms 2 ms vbchtmnas9k02-t0-4-0-1.coxfiber.net [216.54.0.29]
4 12 ms 13 ms 3 ms 68.10.8.229
5 7 ms 7 ms 7 ms chndbbr01-pos0202.rd.ph.cox.net [68.1.0.242]
6 10 ms 8 ms 9 ms ip10-167-150-2.at.at.cox.net [70.167.150.2]
7 10 ms 9 ms 10 ms 100ge7-1.core1.nyc4.he.net [184.105.223.166]
8 72 ms 84 ms 74 ms 10gr10-3.core1.lax1.he.net [72.52.92.226]
9 76 ms 76 ms 90 ms 10g1-3.core1.lax2.he.net [72.52.92.122]
10 81 ms 74 ms 74 ms 205.134.225.38
11 72 ms 71 ms 72 ms www.inmotionhosting.com [192.145.237.216]
```


As you can see, there are several rows divided into columns on the report. Each row represents a “hop” along the route. Think of it as a check-in point where the signal gets its next set of directions. Each row is divided into five columns. A sample row is below:

` 10 81 ms 74 ms 74 ms 205.134.225.38`


Let’s break this particular hop down into its parts.

| Hop # | RTT 1 | RTT 2 | RTT 3 | Name/IP Address |
|---|---|---|---|---|
| 10 | 81 ms | 74 ms | 74 ms | 205.134.225.38 |

| Column | Description |
|---|---|
Hop # | Sequential number of the router in the path (1 = your local router/gateway). |
RTT 1 / RTT 2 / RTT 3 | Round-trip times (in milliseconds) for the three probe packets. Lower and more consistent values are better. |
Hostname / IP | Resolved hostname (if available) and/or IP address of the router at that hop. |

### Key Interpretation Guidelines

**Low, consistent latency**(under 50–100 ms within the same country) is normal in early hops.**Sudden large jumps**in latency (e.g., from 20 ms to 200+ ms) that persist in subsequent hops indicate a bottleneck or congestion starting at that specific router.**Asterisks (**: The hop did not reply within the timeout period. Common reasons include:`* * *`

) or “Request timed out”- Firewalls or routers configured not to respond to traceroute probes (harmless in many cases).
- High load on the router (ICMP responses are low priority).
- Actual packet loss or routing blackholes.

**Timeouts at the beginning**: Usually normal (local router or ISP gateway often blocks ICMP).**Timeouts at the end**: May indicate the destination firewall blocks diagnostic packets, but normal web traffic (HTTP/HTTPS) can still work.**High latency across many hops**: Could point to transoceanic routing, peering issues, or widespread congestion.**Packet loss pattern**: If loss appears at one hop and continues, the problem likely starts there.

**Rule of thumb for continental US routing**:

- Under 150 ms total is generally acceptable.
- Consistent increases toward the destination suggest a problem near the target (possibly InMotion network or upstream provider).
- Problems in the first 3–5 hops → Contact your ISP or check your local network.
- Problems in the middle → Intermediate carrier/peering issue (often temporary).
- Problems near the final hops → Likely on the hosting provider side.

## Common Traceroute Issues and What They Mean

**High latency in the first few hops**: Local Wi-Fi, router, or ISP problem. Try wired connection or restarting your router.**Many asterisks in the middle**: Normal for some backbone routers; not always indicative of loss unless accompanied by latency spikes.**Traceroute stops early with repeated timeouts**: Destination may block probes, or there is a routing loop/firewall. Test with different tools (MTR, TCP traceroute).**Different results from ping vs. traceroute**: Traceroute uses different packet types; firewalls may treat them differently.**Varying paths on repeated runs**: Internet uses dynamic routing; minor variations are normal.**Extremely high RTT (500+ ms)**: Possible satellite links, international routing, or severe congestion.

## Best Practices and Tips for Effective Troubleshooting

- Run tests from multiple locations (your home, mobile hotspot, another network) to isolate the issue.
- Use InMotion Hosting’s Traceroute Parser tool after running your test.
- Include your current public IP address when submitting results to support.
- For ongoing issues, use
**MTR**instead of one-off traceroute. - Test to both your domain and the raw server IP.
- Avoid running during peak hours only—compare off-peak vs. peak results.
- On InMotion VPS/dedicated servers, high latency near the final hops may relate to server load, PHP-FPM, or network configuration—combine with server-side diagnostics (e.g.,
`htop`

, logs).

If the traceroute shows clean low-latency paths all the way to InMotion’s network but your site is still slow, the issue is likely application-related (e.g., database queries, unoptimized code, or PHP-FPM `max_children`

limits) rather than network connectivity.

## Next Steps with InMotion Hosting Support

When opening a support ticket:

- Provide both
**ping**and**traceroute**(or MTR) results. - Describe symptoms (slow loading, timeouts, specific error messages).
- Note the time of day the issue occurs and whether it affects all visitors or only some locations.

Properly interpreted traceroute results significantly speed up diagnosis. Most connectivity problems are resolved by identifying the problematic hop and coordinating between your ISP and hosting provider.

For InMotion Hosting customers, these tools work seamlessly with cPanel, AlmaLinux, or Ubuntu-based VPS environments. If you need help analyzing a specific traceroute output, paste it into a support ticket along with the details above. This approach helps isolate issues quickly and keeps your website performing optimally.