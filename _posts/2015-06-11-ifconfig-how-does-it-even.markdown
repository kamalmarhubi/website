---
title: "ifconfig: how does it even?"
date: 2015-06-11
---

I can already [measure the latency][ipc-latency-post] of TCP sockets over the
loopback interface. I want to compare this to TCP sockets connecting to one of
the ‘real’ addresses the machine has, to see if it's any different. I could see
this being either the same as the loopback interface, or being slower. I'm way
below the level I have any real knowledge of at this point, so there's only one
way to find out.

[ipc-latency-post]: /blog/2015/06/10/some-early-linux-ipc-latency-data/

But rather than hardcode in the IP addresses, or take them on the command line,
I want the benchmark to find them itself. One thing I want out of these
benchmarks is for them to build on any Linux system, and run without needing
machine-specific arguments.  When _I_ find out what IP addresses my machine
has, I use `ifconfig`. But how does `ifconfig` do it? I was about to Google the
answer when I realised this would be a perfect time use `strace`, and so I did!

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">I just straced ifconfig to find out how it finds out which interfaces exist! /cc <a href="https://twitter.com/b0rk">@b0rk</a></p>&mdash; Kamal Marhubi (@kamalmarhubi) <a href="https://twitter.com/kamalmarhubi/status/608735834905415680">June 10, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Here's the quick summary of what happens:

1. create sockets of both `AF_INET` and `AF_INET6` families
2. read `/proc/net/dev` to get a complete list of interfaces; we'll need this
   to get the addresses for `AF_INET6` socket, as well as to get the list of
   interfaces that don't have an `AF_INET` address
3. use the `SIOCGIFCONF` `ioctl` to get a list of addresses for `AF_INET`
4. loop through the interface names that came from `/proc/net/dev` and for each
   one
   - read `/proc/net/if_inet6` to get the IPv6 address for the interface, if
     any
   - use a series of `ioctl` calls to get data about the interface

To see in more detail, take a look at [this gist][gist] for an annotated
`strace` of `ifconfig` and `ifconfig -s`.

[gist]: https://gist.github.com/kamalmarhubi/1dfc1fa302916e21975d
