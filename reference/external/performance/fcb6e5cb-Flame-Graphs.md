---
title: Flame Graphs
source: https://www.brendangregg.com/flamegraphs.html
kind: external
domain: performance
author: Brendan Gregg
license: fair-use
fetched_at: 2026-05-18
tags: [external, performance]
---

> [!info] External article · imported reference
> Source: [www.brendangregg.com](https://www.brendangregg.com/flamegraphs.html)
> Author: Brendan Gregg
> License: fair-use
> Fetched: 2026-05-18

# Flame Graphs

# Flame Graphs

[![](FlameGraphs/cpu-mysql-crop-500.png)](FlameGraphs/cpu-mysql-updated.svg)*CPU Flame Graph*

This is the official website for flame graphs: a visualization of hierarchical data that I created to visualize stack traces of profiled software so that the most frequent code-paths can be identified quickly and accurately. They can be generated using my open source programs on [github.com/brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph) which create interactive SVGs. There are now over 100 other implementations, many open source (see the [Updates](#Updates) section) and they are also now available in most commercial profilers. My colleague on the Netflix performance engineering team, Martin Spier, created a more interactive d3 version, also widely used: [d3-flame-graph](https://github.com/spiermar/d3-flame-graph).

These pages (or posts) introduce different types of flame graphs:

1. [CPU](FlameGraphs/cpuflamegraphs.html)
2. [AI/GPU](/blog/2024-10-29/ai-flame-graphs.html)
3. [Memory](FlameGraphs/memoryflamegraphs.html)
4. [Off-CPU](FlameGraphs/offcpuflamegraphs.html)
5. [Hot/Cold](FlameGraphs/hotcoldflamegraphs.html)
6. [Differential](/blog/2014-11-09/differential-flame-graphs.html)

The example on the right is a portion of a CPU flame graph, showing MySQL codepaths that are consuming CPU cycles, and by how much.

Flame graphs can also be used for any hierarchical data. E.g., file system contents (see [instructions](/blog/2017-02-05/file-system-flame-graph.html); [comparisons with treemaps and sunbursts](/blog/2017-02-06/flamegraphs-vs-treemaps-vs-sunburst.html)). To generate them for custom time ranges for the study of variation, perturbations, and periodic activity, see [FlameScope](flamescope.html).

On this page: [Summary](#Summary),
[OSes](#OperatingSystems),
[Presentation](#Presentation),
[Variations](#Variations),
[Origin](#Origin),
[Updates](#Updates).

## Summary

The x-axis shows the stack profile population, sorted alphabetically (it is not the passage of time), and the y-axis shows stack depth, counting from zero at the bottom. Each rectangle represents a stack frame. The wider a frame is is, the more often it was present in the stacks. The top edge shows what is on-CPU, and beneath it is its ancestry. Original flame graphs use random colors to help visually differentiate adjacent frames. Variations include inverting the y-axis (an "icicle graph"), changing the hue to indicate code type, and using a color spectrum to convey an additional dimension.

Flame graphs are both a static and dynamic visualization. As a static visualization, a flame graph can be saved as an image, included in print (books), and will still convey the "big picture" as only the most frequent frames have enough width for labels. A dynamic visualization allows interactive features to aid navigation and comprehension, including:

- Mouse hover shows additional frame details in a status bar.
- Mouse click zooms the visualization horizontally, revealing function names previously elided.
- Search matches and highlights a given term and shows the "cumulative percentage" of stacks including the search term.

This visualization is fully explained in my ACMQ article [The Flame Graph](http://queue.acm.org/detail.cfm?id=2927301), also published in [Communications of the ACM, Vol. 59 No. 6](http://cacm.acm.org/magazines/2016/6/202665-the-flame-graph/abstract).

Also see my [CPU Flame Graphs](FlameGraphs/cpuflamegraphs.html) page, and the [presentation](#Presentation) below.

## Presentation

I gave an updated talk explaining flame graphs at USENIX ATC 2017 titled [Visualizing Performance with Flame Graphs](https://www.usenix.org/conference/atc17/program/presentation/gregg-flame), which is on [youtube](https://www.youtube.com/watch?v=D53T1Ejig1Q) and [slideshare](https://www.slideshare.net/brendangregg/usenix-atc-2017-visualizing-performance-with-flame-graphs) ([PDF](Slides/USENIX_ATC2017_flamegraphs.pdf))

[![](Images/youtube-USENIXATC17.png)](https://www.youtube.com/watch?v=D53T1Ejig1Q)

My first talk on flame graphs was at [USENIX LISA 2013](https://www.usenix.org/conference/lisa13/technical-sessions/plenary/gregg), which ended up as a plenary talk ([youtube](http://www.youtube.com/watch?v=nZfNehCzGdw), [slideshare](http://www.slideshare.net/brendangregg/blazing-performance-with-flame-graphs), [PDF](Slides/LISA13_Flame_Graphs.pdf)):

## Operating Systems

Some operating system profilers now have built-in support for flame graphs:

- Linux: perf (perf script report flamegraph)
- Windows: WPA, PerfView

Flame graphs can also be generated from any profile data that contains stack traces, including from the following profiling tools:

- Linux: perf, eBPF, SystemTap, and ktap
- FreeBSD: DTrace
- Mac OS X: Instruments
- Windows: Xperf.exe

Once you have a profiler that can generate meaningful stacks, converting them into a flame graph is usually the easy step.

There are also numerous profiling products and companies that now support flame graphs. See the Updates section below.

## Variations

**Icicle charts** are flame graphs upside down. Some people prefer it that way. My flamegraph.pl creates them using --inverted. I prefer the standard "flame" layout, where the y-axis is counting stack depth upwards from zero at the bottom. I'm also used to scanning them top-down to look for plateaus. But for very deep stacks the flame graph layout (with a GUI that starts at the top) often means the initial view may be mostly empty (a few thin interrupt stacks) forcing the developer to scroll down to find the bulk of the profile. For developers who prefer reading root-to-leaf anyway, an icicle layout instead means that the starting point is always on screen without needing to scroll. For that reason, many flame graph implementations use the icicle layout by default instead. Others use the flame graph layout but begin showing the bottom so that the root frames are on screen. I don't have a strong opinion about this, do whichever you prefer! Preferably include a toggle so that the end user can pick their preferred layout.

**Flame charts** were first added by Google Chrome's WebKit Web Inspector ([bug](https://bugs.webkit.org/show_bug.cgi?id=111162)). While inspired by flame graphs, flame charts put the passage of time on the x-axis instead of the alphabet. This means that time-based patterns can studied. Flame graphs reorder the x-axis samples alphabetically, which maximizes frame merging, and better shows the big picture of the profile. Multi-threaded applications can't be shown sensibly by a single flame chart, whereas they can with a flame graphs (a problem flame charts didn't need to deal with, since it was initially used for single-threaded JavaScript analysis). Both visualizations are useful, and tools should make both available if possible (e.g., TraceCompass does). Some analysis tools have implemented flame charts and mistakingly called them flame graphs.

**Sunburst layout** using radial coordinates for the x-axis, a flame graph can be turned into a hierarchical pie chart. The Google Web Inspector team [prototyped](https://bugs.chromium.org/p/chromium/issues/detail?id=452624) them. I also discussed them vs flame graphs in my [comparison](/blog/2017-02-06/flamegraphs-vs-treemaps-vs-sunburst.html) post.

## Origin

I invented flame graphs when working on a MySQL performance issue and needed to understand CPU usage quickly and in depth. The regular profilers/tracers had produced walls of text, so I was exploring visualizations. I first traced CPU function calls and visualized it using Neelakanth Nadgir's time-ordered visualization for callstacks, which itself was inspired by Roch Bourbonnais's CallStackAnalyzer and Jan Boerhout's vftrace. These look similar to flame graphs, but have the passage of time on the x-axis. But there were two problems: the overhead of function tracing was too high, perturbing the target, and the final visualization was too dense to read when spanning multiple seconds. I switched to timed sampling (profiling) to solve the overhead problem, but since the function flow is no longer known (sampling has gaps) I ditched time on the x-axis and reordered samples to maximize frame merging. It worked, the final visualization was much more readable. Neelakanth and Roch's visualizations used completely random colors to differentiate frames. I thought it looked nicer to narrow the color palette, and picked just warm colors initially as it explained why the CPUs were "hot" (busy). Since it resembled flames, it quickly became known as flame graphs.

I described more detail of the original performance problem that led to flame graphs in my ACMQ/CACM article (link above). The flame graph visualization is really an adjacency diagram with an inverted icicle layout, which I used to visualize profiled stack traces.

## Updates

Flame graphs were released in Dec 2011. Not long afterwards (updated in 2012):

More Flame Graph news (updated Apr 2013):

- I wrote some documentation for [Linux Kernel Flame Graphs](FlameGraphs/cpuflamegraphs.html#perf), generated using either perf and SystemTap for the profile data.
- Mark Probst developed [Flame Graphs for Instruments](http://schani.wordpress.com/2012/11/16/flame-graphs-for-instruments/) on Mac OS X.
- Sam Saffron has developed [Flame graphs in Ruby MiniProfiler](http://samsaffron.com/archive/2013/03/19/flame-graphs-in-ruby-miniprofiler), and shows examples of amazingly deep stacks.
- Bruce Dawson has an excellent post on [Summarizing Xperf CPU Usage with Flame Graphs](http://randomascii.wordpress.com/2013/03/26/summarizing-xperf-cpu-usage-with-flame-graphs/) on Microsoft Windows. It includes examples for Visual Studio and Outlook, and a stack folding script to process the Xperf output.
- Google Chrome's performance analysis tool, WebKit Web Inspector, introduced "[Flame Charts](https://bugs.webkit.org/show_bug.cgi?id=111162)", inspired by flame graphs. These are a similar visualization, but the x-axis is time, instead of the alphabet. Check out the [screenshot](https://bug-113052-attachments.webkit.org/attachment.cgi?id=194532), which includes a mouse-over popup that links to the source code. (Update: there is a bug to add flame graphs to Chrome, in addition to flame charts: [Chromium 452624](https://code.google.com/p/chromium/issues/detail?id=452624).)
- Tim Bunce has been improving and adding features to Flame Graphs, and has included them in Perl's best profiler, Devel:::NYTProf, for profiling Perl. See his post on [NYTProf v5 – Flaming Precision](http://blog.timbunce.org/2013/04/08/nytprof-v5-flaming-precision/).

More Flame Graph news (updated Aug 2013):

- I wrote a document summarizing four techniques for generating [Memory Leak (and Growth) Flame Graphs](FlameGraphs/memoryflamegraphs.html), which visualize stacks with byte counts, instead of the traditional CPU sample Flame Graphs. I also colored them green to indicate that they are a different type.
- John Graham-Cumming showed how CloudFlare was using SystemTap-generated flame graphs for optimizing their [Lua WAF](http://blog.cloudflare.com/cloudflares-new-waf-compiling-to-lua).
- Yichun Zhang showed how [Off-CPU Time Flame Graphs](http://t.co/tusuJkqKXF) (PDF) can solve issues of blocking time. See [Off-CPU Analysis](offcpuanalysis.html) for why this is important.
- Igor Soarez wrote [How To Make Flame Graphs](http://blog.crowdprocess.com/post/58797416513/how-to-make-flame-graphs) for node.js analysis, showing all steps involved.
- [Paul Irish](https://plus.google.com/+PaulIrish/posts/3nqbozQF3VN) and [Umar Hansa](https://plus.google.com/111696301786817020074/posts/7PjohGxGECZ) posted an awesome demo of using FlameCharts to investigate time in V8, which includes zooming in and clicking on functions to go to code, and Addy Osmani posted a longer [video tutorial](http://addyosmani.com/blog/devtools-flame-charts/). While these aren't Flame Graphs, they show developments in a related visualization: a time-series version that retains sequence and ordering.

More Flame Graph news (updated Jun 2014):

- I gave a talk at USENIX/LISA13 titled [Blazing Performance with Flame Graphs](https://www.usenix.org/conference/lisa13/technical-sessions/plenary/gregg), which covered them in enormous detail. See the [slides](http://www.slideshare.net/brendangregg/blazing-performance-with-flame-graphs) and [video](http://www.youtube.com/watch?v=nZfNehCzGdw). At the last minute this talk became a Plenary when another speaker became ill and had to cancel.
- Joab Jackson wrote an article about flame graphs for [PCWorld](http://www.pcworld.com/article/2062160/usenix-flame-graph-shows-system-performance-in-a-new-light.html), [InfoWorld](http://www.infoworld.com/d/business-intelligence/flame-graph-shows-system-performance-in-new-light-230520?page=0,1), and [CIO](http://www.cio.com/article/742883/USENIX_Flame_Graph_Shows_System_Performance_in_a_New_Light?page=2&taxonomyId=600010).
- I wrote documents on [Off-CPU Flame Graphs](FlameGraphs/offcpuflamegraphs.html), and [Hot/Cold Flame Graphs](FlameGraphs/hotcoldflamegraphs.html), based on what I had documented earlier for my USENIX talk.
- I wrote a post showing how the Google lightweight-java-profiler can be used to create [Java Flame Graphs](http://www.brendangregg.com/blog/2014-06-12/java-flame-graphs.html). This is only Java code time, and not other CPU consumers. See my later work with Java (-XX:+PreserveFramePointer) for a different and more complete profile.
- [Vladimir Kirillov](https://twitter.com/darkproger) has developed [eflame](https://github.com/proger/eflame) (github), a Flame Graph profiler for Erlang. Blocking calls are in blue, see the screenshot.
- [Trevor Norris](https://twitter.com/trevnorris) posted instructions for creating [Linux Node.js](https://gist.github.com/trevnorris/9616784) CPU flame graphs, by using [perf_events](perf.html) and --perf-basic-prof. See the [example](https://twitter.com/trevnorris/status/445720118207606784).
- [Luca Canali](https://twitter.com/LucaCanaliDB) provides many examples of [Flame Graphs for Oracle](http://externaltable.blogspot.com/2014/05/flame-graphs-for-oracle.html) databases, and explains what they mean.
- Julien Charbon posted some CPU flame graphs on FreeBSD, to analyze [TCP stack lock contention with short-lived connections](https://lists.freebsd.org/pipermail/freebsd-net/2014-May/038810.html).
- Gabriel posted [Profiling and optimising with Flamegraph](http://www.gabriel.urdhr.fr/2014/05/23/flamegraph/), explaining all the steps with some really good examples.

More Flame Graph news (updated Dec 2014):

More Flame Graph news (updated Jun 2015):

- Shawn Sterling posted about his excellent talk at Linuxfest Northwest 2014: [Getting Started With Flamegraph](http://systemtemplar.org/blog/2015/04/07/getting-started-with-flamegraph/), which includes both the slides and video (turn the volume up).
- Scott Lystig Fritchie gave a great talk on [profiling Erlang code](http://www.snookles.com/erlang/ef2015/) including flame graphs at Erlang Factory San Francisco 2015.
- I submitted tickets for a flame graph / flame chart toggle for [Google Chrome (Issue 452624)](https://code.google.com/p/chromium/issues/detail?id=452624) and [Firefox (bug 1123495)](https://bugzilla.mozilla.org/show_bug.cgi?id=1123495). I'd love to see both implemented. The chromium ticket prompted interesting discussion and prototypes.
- In my SCALE13x talk (2015), I previewed [mixed-mode Java flame graphs](http://www.slideshare.net/brendangregg/scale2015-linux-perfprofiling/57) using Linux perf_events ("perf"). This uses a JVM frame pointer patch that has become the -XX:+PreserveFramePointer option in both JDK9 ([JDK-8068945](https://bugs.openjdk.java.net/browse/JDK-8068945)) and JDK8u60 ([JDK-8072465](https://bugs.openjdk.java.net/browse/JDK-8072465)). For the first time, we can see all CPU consumers in one visualization! See slides 40 to 57 for more details, and my post about it (when I write it!).
- Francesco Mazzoli post about [Flame graphs for GHC time profiles with ghc-prof-flamegraph](https://www.fpcomplete.com/blog/2015/04/ghc-prof-flamegraph), including a tutorial for making flame graphs from the existing GHC (Haskell) profiler output.
- Robin Moffatt posted about [Analysing ODI performance with Flame Graphs](http://www.rittmanmead.com/2015/04/analysing-odi-performance-with-flame-graphs/), for understanding load plans in the Oracle Data Integrator.
- Flame graphs were [demoed](https://twitter.com/bleen18/status/589886547262058499) at d3NYC (Drupal) 2015, but I haven't found slides online yet.
- Cor-Paul Bezemer has been investigating flame graph differentials with his [Flamegraphdiff](http://corpaul.github.io/flamegraphdiff/) software, which shows the difference from A to B using three flame graphs simultaneously. This was also the subject of a SANER2015 [paper](http://ieeexplore.ieee.org/xpl/login.jsp?tp=&arnumber=7081872&url=http%3A%2F%2Fieeexplore.ieee.org%2Fiel7%2F7066219%2F7081802%2F07081872.pdf%3Farnumber%3D7081872) ($), and [talk](http://www.slideshare.net/corpaulbezemer/saner-2015-era-track).
- M. Isuru Tharanga Chrishantha Perera wrote a translator for [Java Flight Recorder](https://github.com/chrishantha/jfr-flame-graph) profiles, for making flame graphs.
- Jan Stępień developed [Pluggable Flame Graphs for Clojure](https://github.com/jstepien/flames/) ([example](https://stepien.cc/~jan/flames-01.svg)).
- Evan Hempel has created a [python-flamegraph](https://github.com/evanhempel/python-flamegraph) profiler for Python, which generates the folded stack output suitable for making into flame graphs. See the README on github for instructions.
- Franck Pachot posted another example of using CPU flame graphs for identification of an [Oracle database bug](http://www.dbi-services.com/index.php/blog/entry/flame-graph-for-quick-identification-of-oracle-bug).
- Strongloop have included a nice looking [Node.js flame graphs module](https://strongloop.com/strongblog/transaction-tracing-node-js-root-cause) for their Arc Profiler.
- [@yoheia](https://twitter.com/yoheia) posted a detailed flame graph post showing [Linux kernel profiling (in Japanese)](http://d.hatena.ne.jp/yohei-a/20150706/1436208007).
- Zoltan Majo fixed [JDK-8068945](https://bugs.openjdk.java.net/browse/JDK-8068945) for Java 9 and [JDK-8072465](https://bugs.openjdk.java.net/browse/JDK-8072465) for Java 8 update 60 build 19 (or later, download as early access [here](https://jdk8.java.net/download.html)). This adds -XX:+PreserveFramePointer, which allows Linux perf_events to sample full stacks for making flame graphs. This began with a prototype patch I developed and submitted: ([A hotspot patch for stack profiling](http://mail.openjdk.java.net/pipermail/hotspot-compiler-dev/2014-December/016477.html)). Great to see this functionality make it into the JVM!
- Min Zhou has developed [PerfJ](https://github.com/coderplay/perfj), for automating the collection of Java flame graphs, using the frame pointer patch (see previous item).

More Flame Graph news (updated Dec 2015):

- Myself and Martin Spier posted about [Java in Flames](http://techblog.netflix.com/2015/07/java-in-flames.html) ([PDF](/Articles/Netflix_Java_in_Flames.pdf)) for the Netflix Tech Blog, showing Java mixed-mode flame graphs using the new -XX:+PreserveFramePointer JVM option.
- Jonathan Perkin used memory flame graphs for [Reducing RAM usage in pkgin](http://www.perkin.org.uk/posts/reducing-ram-usage-in-pkgin.html).
- Isuru Perera postabout about [Java CPU Flame Graphs](http://isuru-perera.blogspot.com/2015/07/java-cpu-flame-graphs.html) with an example making use of Min's PerfJ.
- Ben Sandler from Uber has posted [go-torch](https://github.com/uber/go-torch), a flame graph profiler for go-lang programs.
- Andi Kleen demonstrated [Generating flame graphs with Processor Trace](http://halobates.de/blog/p/329), a feature from modern Intel CPUs for very high frequency sampling.
- I wrote about the new [flame graph search](http://www.brendangregg.com/blog/2015-08-11/flame-graph-search.html) feature. The matched percentage is very handy, so I don't have to mouse over many tiny frames and add them up manually.
- Eben Freeman posted about [Profiling Python in Production](https://nylas.com/blog/performance/), which includes a python profiler and basic d3 flame graphs.
- Will Sewell mentioned flame graphs in his [Top tips and tools for optimising Haskell](https://blog.pusher.com/top-tips-and-tools-for-optimising-haskell/) post, and linked to the work for [GHC flame graphs](https://www.fpcomplete.com/blog/2015/04/ghc-prof-flamegraph#flame-graphs-for-ghc-time-profiles) by Francesco Mazzoli.
- Bo Lopker created [djdt-flamegraph](https://github.com/23andMe/djdt-flamegraph) (github) for getting a flame graph of current requests in the Django Python web framework.
- NodeSource have flamegraphs in their [NSolid](https://nodesource.com/products/nsolid) Node.js analysis product. The graphics look very nice, and they also have treemaps and sunbursts (both of which I think are usually less effective than flame graphs, but I don't mind having the option).
- Alex Ciminian has been developing a [d3-flame-graphs](https://github.com/cimi/d3-flame-graphs) library (github) for d3, implemented in CoffeeScript. Check out the demo, it looks really nice so far.
- Martin Spier (a colleague at Netflix) has been developing [d3-flame-graph](https://github.com/spiermar/d3-flame-graph), as a d3 library implemented in JavaScript. This will be integrated into our open source [Vector](http://techblog.netflix.com/2015/04/introducing-vector-netflixs-on-host.html) instance analysis tool. It has zoom transitions!
- I gave a [Java Mixed-Mode Flame Graphs](http://www.brendangregg.com/blog/2015-11-06/java-mixed-mode-flame-graphs.html) talk at JavaOne 2015, including all the latest updates.
- David Calavera wrote about [Docker flame graphs for Go](https://medium.com/@calavera/docker-flame-graphs-f9523e98d57d#.m2xz1vl18).
- Carol Nichols wrote detailed instructions for [Rust Profiling with Instruments and FlameGraph on OSX: CPU/Time](http://carol-nichols.com/2015/12/09/rust-profiling-on-osx-cpu-time/).

More Flame Graph news (updated Jun 2016):

- Apekshit Sharma wrote a post about [Saving CPU! Using Native Hadoop Libraries for CRC computation in HBase](https://blogs.apache.org/hbase/entry/saving_cpu_using_native_hadoop).
- Mike Huang wrote a Netflx tech blog post to show how he helped with [Saving 13 Million Computational Minutes per Day with Flame Graphs](http://techblog.netflix.com/2016/04/saving-13-million-computational-minutes.html).
- Srdjan Marinovic and Ryan Day created [goprofui](https://github.com/wirelessregistry/goprofui) includes a cpu.go profiler for golang and flame graphs.
- I wrote an article for ACMQ, [The Flame Graph](http://queue.acm.org/detail.cfm?id=2927301), which defines flame graphs, describes their origin, explains how to interpret them, and discusses possible future developments.
- David Mark Clements has developed [0x](https://github.com/davidmarkclements/0x), a new interactive flame graph profiler for Node.js processes on both Linux and OS X. See the [demo](http://davidmarkclements.github.io/0x-demo/).
- [Qt Creator 4.0.0](https://blog.qt.io/blog/2016/05/11/qt-creator-4-0-0-released/?utm_content=32276554&utm_medium=social&utm_source=twitter) now includes [flame graphs](https://doc.qt.io/qtcreator/creator-qml-performance-monitor.html#visualizing-statistics-as-flame-graphs) along with timeline and statistical views. That's how it should be done: different views of the same data you can switch between.
- Antonio Pérez developed [4gl-flamegraph](https://github.com/skarcha/4gl-flamegraph) for processing the profiler output of Genero by 4Js (Four Js).
- Ty Overby published a [flamegraph profiling tool for rust](https://github.com/TyOverby/flame).
- The [vprof Python package](https://github.com/nvdv/vprof) provides interactive visualizations for profiling, including flame graphs.
- Alastair Butler and Kei Yoshimoto published the paper [Large scale semantic representation with flame graphs](http://www.anlp.jp/proceedings/annual_meeting/2015/pdf_dir/C1-5.pdf) (PDF), where they used them to visualize semantically annotated corpora, which has been parsed into a hierarchy.
- Jonathan Newbrough developed the [Gumshoe Load Investigator](https://github.com/dcm-oss/gumshoe) for Java analysis, initially for internal use in the Dell Cloud Manager, which makes good use of flame graphs.
- Mahesh Dathrika published [Igniting Node.js Flame](http://www.ebaytechblog.com/2016/06/15/igniting-node-js-flames/), showing how they use the v8-profiler at eBPF to create Node.js flame graphs.
- Maciek Lesiczka is building an ASP.NET profiler that includes flame graphs, and introduced it in [netric.io - lightweight ASP.NET profiler](http://macieklesiczka.github.io/2016/06/29/netric_io/).
- Nitsan Wakart covered Java flame graphs and various Java profiling challenges in his Devoxx UK talk [Extreme Profiling: Digging Into Hotspots by Nitsan Wakart](https://www.youtube.com/watch?v=7PkkxDaFDj8) (youtube).
- Maciek Lesiczka has been developing [netric.io - lightweight ASP.NET profiler](http://macieklesiczka.github.io/2016/06/29/netric_io/), an ASP.NET profiler that includes flame graphs.
- My article [The Flame Graph](http://cacm.acm.org/magazines/2016/6/202665-the-flame-graph/abstract) was published in CACM.

More Flame Graph news (updated Dec 2016):

- Rhys Hiltner from Twitch blogged about [Go's march to low-latency GC](https://blog.twitch.tv/gos-march-to-low-latency-gc-a6fa96f06eb7#.yit1viugb), which included flame graph analysis of GC time.
- Dmytro Semenov on the ebay tech blog posted [Mastering the Fire](http://www.ebaytechblog.com/2016/07/14/mastering-the-fire/), about how they can create flame graphs on production servers at any time with one click of a button.
- Olivier Cano posted about [Measuring Snap performance](https://medium.com/intel-sdi/measuring-snap-performance-7cde2b315a62#.m8t6nxb8i) which includes flame graphs for golang.
- Kay Ousterhout wrote about [Generating Flame Graphs for Apache Spark](https://gist.github.com/kayousterhout/7008a8ebf2babeedc7ce6f8723fd1bf4) using Java Flight Recorder.
- Adam Perry published [Rust Performance: A story featuring perf and flamegraph on Linux](http://blog.adamperry.me/rust/2016/07/24/profiling-rust-perf-flamegraph/), which also has great examples of using perf.
- Bruce Dawson published a post on [ETW Flame Graphs Made Easy](https://randomascii.wordpress.com/2016/09/05/etw-flame-graphs-made-easy/) and the new support in Windows Performance Analyzer ([WPA](https://msdn.microsoft.com/en-us/windows/hardware/commercialize/test/wpt/graphs#flame_graphs)).
- Aviem Zur from PayPal engineering wrote about [Spark in Flames - Profiling Spark Applications Using Flame Graphs](https://www.paypal-engineering.com/2016/09/08/spark-in-flames-profiling-spark-applications-using-flame-graphs/).
- Cédric Champeau (I think) has added automatic Flame graph generation for [Gradle's perf tests](https://twitter.com/gradle/status/773836017677795328). Makes a lot of sense, as does auto generation for software versions and builds (given a load generator).
- Bill Smith from indeed engineering posted [A Funny Thing Happened on the Way to Java 8](http://engineering.indeedblog.com/blog/2016/09/job-search-web-app-java-8-migration/), where code cache issues are investigated with flame graphs (I've also found a code cache issue this way!).
- Bert Hubert included flame graphs in his detailed analysis [Optimizing optimizing: some insights that led to a 400% speedup of PowerDNS](https://hackernoon.com/optimizing-optimizing-some-insights-that-led-to-a-400-speedup-of-powerdns-5e1a44b58f1c#.wnny5y5fc).
- Luca Canali has posted more flame graphs, this time [Apache Spark 2.0 Performance Improvements Investigated With Flame Graphs](http://db-blog.web.cern.ch/blog/luca-canali/2016-09-spark-20-performance-improvements-investigated-flame-graphs), and even includes some perf stat analysis.
- Mahmoud Hatem posted about [perf_events : Off/On/Mixed CPU flamegraph extended with oracle wait events](https://mahmoudhatem.wordpress.com/2016/09/23/perf_events-offonmixed-cpu-flamegraph-extended-with-oracle-wait-events/), showing both on- and off-CPU flame graphs for an Oracle workload.
- Evan Klitzke posted [pyflame: Uber enginering's ptracing profiler for python](https://eng.uber.com/pyflame/), as well as the source on [github](https://github.com/uber/pyflame). Python is tricky to profile as it's interpreted only, so basic frame pointer-based stack walking only identifies interpreter frames.
- I posted about [Linux 4.9's Efficient BPF-based Profiler](/blog/2016-10-21/linux-efficient-profiler.html), an important feature that will make flame graph profiling much more efficient.
- Jerome Terry posted instructions for [Generating Java Mixed Mode Flame Graphs](http://blog.jerometerry.com/2016/12/generating-java-mixed-mode-flame-graphs.html) summarizing all the latest steps.
- Joel Fernandes posted [ARMv8: Flamegraph and NMI Support](http://www.linuxinternals.org/blog/2016/12/31/nmi-perf-armv8/).

More Flame Graph news (updated Jun 2017):

More Flame Graph news (updated Dec 2017):

- My USENIX ATC 2017 talk was posted: [Visualizing Performance with Flame Graphs (youtube)](https://www.youtube.com/watch?v=D53T1Ejig1Q) ([sildes](https://www.slideshare.net/brendangregg/usenix-atc-2017-visualizing-performance-with-flame-graphs)): this is my updated flame graphs talk.
- I posted [Coloring Flame Graphs: Code Hues](http://www.brendangregg.com/blog/2017-07-30/coloring-flamegraphs-code-type.html), explaining how I've been improving them
- Oracle added flame graphs to [Oracle Developer Studio Performance Analyzer](http://www.oracle.com/technetwork/server-storage/solarisstudio/documentation/o11-151-perf-analyzer-brief-1405338.pdf) (PDF).
- Mark Price wrote about [Using Flame Graphs to Analyze & Determine Cassandra Performance Under Stress](https://medium.com/linode-cube/using-flame-graphs-to-analyze-determine-cassandra-performance-under-stress-part-2-of-3-b2f9eebd9c5) (Part 2 of 3), which includes Java thread pool names from jstack to group towers.
- LinkedIn has enhanced flame graphs to do [Common Issue Detection for CPU Profiling](https://engineering.linkedin.com/blog/2017/09/common-issue-detection-for-cpu-profiling?utm_content=buffer21674&utm_medium=social&utm_source=twitter.com&utm_campaign=buffer), and applied it to several common JVM issue. This is a good idea.
- Alexey Ivanov wrote [Optimizing web servers for high throughput and low latency](https://blogs.dropbox.com/tech/2017/09/optimizing-web-servers-for-high-throughput-and-low-latency/), which is a grand tour of many Linux performance tools including flame graphs and bcc/BPF. Check it out.
- Nitsan Wakart gave an excellent talk [Exploring Java Perf Flamegraphs](https://2017.javazone.no/program/56179b136b91458a843383e13fd2efa1) at JavaZone 2017.
- Mark Price added [Java Heap Allocation Flamegraphs](https://epickrram.blogspot.co.uk/2017/09/heap-allocation-flamegraphs.html) to grav toolkit, by using the USDT object-alloc probe.
- I included flame graphs in my Kernel Recipes 2017 talk: [Using Linux perf at Netflix (youtube)](https://www.youtube.com/watch?v=UVM3WX8Lq2k) ([slides](https://www.slideshare.net/brendangregg/kernel-recipes-2017-using-linux-perf-at-netflix)).
- Miel Donkers posted [The JVM on Fire – Using Flame Graphs to Analyse Performance](https://blog.codecentric.de/en/2017/09/jvm-fire-using-flame-graphs-analyse-performance/), and compared a flame graph to JProfiler tree view.
- Nudge APM appear to have included flame graphs in their [profiling](https://www.nudge-apm.com/features/#profiling) section of their analyzer (for Java, PHP, .NET).
- Adam Sitnik developed flame graphs for [Windows PerfView](https://github.com/Microsoft/perfview/pull/440), which has been [merged](https://twitter.com/SitnikAdam/status/950729146627391489) (yay).
- Martin Spier added flame graphs to [pprof's web UI](https://github.com/google/pprof/issues/166).
- Konrad Malawski posted [Automatic FlameGraph generation from JMH Benchmarks using (SBT) JMH Extras (plain Java too)](http://kto.so/2017/12/10/automatic-flamegraph-generation-from-jmh-benchmarks-using-sbt-jmh-extras-plain-java-too/).
- Alexander Yakushev posted [Profiling tool: async-profiler](http://clojure-goes-fast.com/blog/profiling-tool-async-profiler/), showing how it can be used to generate flame graphs for Clojure apps on the JVM.
- Ruth Grace Wong gave the talk [FlameGraph That: Self-Service Profiling with SaltStack and Rundeck](https://www.youtube.com/watch?v=smW9VXOdAIg) at Salt Conf 17 (thanks!).
- Milian Wolff's [Hotspot - the Linux perf GUI for performance analysis](https://github.com/KDAB/hotspot) tool has flame graphs (I'm not sure exactly when it merged.)

More Flame Graph news (updated Dec 2018):

- Jon Hadad wrote about [Analyzing Cassandra Performance with Flame Graphs](http://thelastpickle.com/blog/2018/01/16/cassandra-flame-graphs.html).
- Amir Langer gave a talk (in Hebrew) about [Flame Graphs and the JVM at eBay](https://www.youtube.com/watch?v=Hw3G272Ztb0) (youtube).
- Datadog say they added flame graphs for [Java performance](https://www.datadoghq.com/dg/apm/java-performance) but they look like flame charts. I'm not sure yet.
- Brendan Ryan wrote about [Profiling Go Applications with Flamegraphs](http://brendanjryan.com/golang/profiling/2018/02/28/profiling-go-applications.html) using Uber's go-torch library.
- Kiran posted [How to use linux perf tools and save dollars - Part 2](http://techunravel.blogspot.com/2018/03/how-to-use-flame-graph.html), including flame graphs of perf samples.
- Vladimir Agafonkin developed [Flamebearer](https://twitter.com/mourner/status/980828722692489218), for lightweight responsive graphs for Node.js and v8.
- Trace Compass added [flame graphs as well as flame charts](https://twitter.com/DavisTurlis/status/978449942200102912), the first tool to support both (yay).
- Myself and my Netflix colleague Martin Spier published [FlameScope](https://medium.com/netflix-techblog/netflix-flamescope-a57ca19d47bb), a new performance analysis tool that allows profiles to be visualized as subsecond-offset heat maps, and then time ranges to be selected and then visualized as a flame graph. It's also on github [Netflix/flamescope](https://github.com/Netflix/flamescope).
- Sadiq Jaffer, Richard Warburton wrote about [Always-on production Flame Graphs for Java](https://www.opsian.com/blog/always-on-production-flame-graphs/).
- Alexandru Olaru posted [Squeeze node.js performance with flame graphs](https://www.alxolr.com/articles/squeeze-node-js-performance-with-flame-graphs).
- Ivan Babrou posted a case study of [https://blog.cloudflare.com/tracing-system-cpu-on-debian-stretch/](Tracing%20System%20CPU%20on%20Debian%20Stretch), including flame graphs (and bcc/eBPF tools).
- Marcus Hirt developed [jmc-flame-view](https://github.com/thegreystone/jmc-flame-view) and posted [instructions](http://hirt.se/blog/?p=989) for getting it going: this adds flame graphs to Java Mission Control.
- Jamie Wong has developed [SpeedScope - Interactive Flamegraph Explorer](http://jamie-wong.com/post/speedscope/), which supports "time order" (flame charts), "left heavy", which are flame graphs with an additional sort so the largest frames move left, and "sandwich" ("middle-out" merge). Great to see all three types in one GUI.
- Sebastián Reca (Muun) showed at at Kaizen how flame graphs helped with [Scaling Bitcoin](https://www.youtube.com/watch?v=nwSuctrzV7Y&t=2913s) (youtube video).

More Flame Graph news (updated Oct 2019):

More Flame Graph news (updated Oct 2020):

More Flame Graph news (updated Nov 2021):

Thanks to everyone who has written about flame graphs, developed them further, and shared their results! I'll update this page from time to time with more news.
