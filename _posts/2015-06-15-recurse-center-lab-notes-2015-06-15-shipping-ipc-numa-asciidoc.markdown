---
title: "Recurse Center lab notes 2015-06-15: shipping, IPC, NUMA, AsciiDoc"
date: 2015-06-15
permalink: /blog/2015/06/15/recurse-center-lab-notes
---

Today I joined the ‘shipping’ checking group, instigated by [Nat Welch][nat].
The idea is to work towards actually shipping something, which I really want to
get better at while at RC.

[nat]: http://natwelch.com/

I decided on a simple utility to list IP addresses. This came out of my writing
on [how `ifconfig` works][ifconfig-post]. [Mark Dominus][dominus] said a
utility that just listed IP addresses would be handy, and he's never come
across one. I did a little poking around the googles and stackoverflows, and
most answers involved `ifconfig`, `grep`, and either `sed` or `awk`. A few
listed the Linux only `ip addr show` which still has a lot of cruft in its
output. I also came across another Linux only option, `hostname
--all-ip-addresses`, which prints them one per line, ignoring loopback and link
local addresses.

[ifconfig-post]: {% post_url 2015-06-11-ifconfig-how-does-it-even %}
[dominus]: http://plover.com/~mjd/

The last one is pretty close to what I would like, but it's missing some handy
flags to specify IPv4 or IPv6, which interface, or whether to include link
local or loopback addresses. It's also Linux only, which a standalone utility
could work around. I'm aiming to have a fairly polished first version for Linux
systems this week, possibly even dabbling in Debian packaging.

Of course, right after deciding that, I got distracted following up on some
links [Anil Madhavapeddy][anil] gave me last Thursday while he was at RC. It
turns out the IPC performance is of a lot of interest to virtualization people,
and he had this [big set of benchmarks][ipcbench]. I also read through slides
from his FOSDEM 2012 talk entitled _The Wild West of UNIX I/O_. This goes into
a lot of details around non-uniform memory architecture (NUMA) and how it
affects latency and throughput between core in a many core machine. It's
incredibly fascinating that the interconnections between the cores can be
inferred from measuring IPC performance.

[anil]: http://anil.recoil.org/
[ipcbench]: https://github.com/avsm/ipc-bench

Even more interesting to me was the little bit at the end on Fable I/O, an
‘ongoing attempt at a “new” sockets API for high-performance data’. Sadly, the
‘ongoing’ part of this no longer seems valid: the best I could find was a
[`libfable`][libfable] repository that hasn't seen any updates in a couple of
years. This got me a bit excited, and I started reading up a bit more on NUMA
support in Linux. I came across a [_Linux Weekly News_ article][lwn-numa-sched]
from 2012 on NUMA aware scheduling support. So many things to think about:

- the idea home NUMA nodes for processes
- moving physical pages between nodes while leaving virtual addresses intact
- grouping processes together into NUMA groups that will always share a home
  node
- the performance gains possible by understanding and exploiting hardware
  details

[libfable]: https://github.com/ms705/libfable
[lwn-numa-sched]: http://lwn.net/Articles/486858/

On the performance note, a quote from the LWN article stands out:

> Without the NUMA balancing patches, over time, the benchmark ended up with
> just as many remote memory accesses as local accesses - allocated memory was
> spread across the system. With the NUMA balancer, 86% of the memory accesses
> were local, leading to a significant speedup.

NUMA: serious business.

After a while I was able to settle down again from all this IPC excitement and
think about how to approach the list-IP-addresses utility. I decided to go with
some [README-driven development][readme], and drafted up a README. After a bit,
I changed the aim to writing a manpage. This was driven by a secret desire to
use [AsciiDoc] (see also: [AsciiDoctor]), as I've been wanting to check it out as a Markdown alternative
for a while. I knew it had good support for manpage generation. In fact, the
[`git` docs][git-docs] are in AsciiDoc.

[asciidoc]: http://asciidoc.org/
[asciidoctor]: http://asciidoctor.org/
[readme]: http://tom.preston-werner.com/2010/08/23/readme-driven-development.html
[git-docs]: https://github.com/git/git/blob/master/Documentation/

So, current status:

- I have some flags being parsed
- can list IPv4 addresses
- manpage needs some rewording and reworking
- the tool needs a name. I had `lsip`, after `lsof`, but it looks wrong. I'm
  now thinking `lsaddr`. I am bad at names.
