---
title: Recurse Center lab notes 2015-06-22
date: 2015-06-22
permalink: /blog/2015/06/22/recurse-center-lab-notes
---

Today was slightly slow. I re-remembered why it's a good idea to leave
things incomplete at the end of the day or week: it gives you somewhere
to start. I consider [`lsaddr`][lsaddr] to be more or less feature
complete at this point, at least for a ‘1.0’. That was the state on
Friday, and is still the state today. If something was left, I would
have had an immediate place to start this morning.

[lsaddr]: https://github.com/kamalmarhubi/lsaddr

I wasn't feeling like going back to IPC benchmarking immediately, so I
ended up deciding to run a version of my [Let's Build a
Shell!][workshop] workshop. I've been wanting to do this for a while,
since I think it's a pretty good way to get a deeper understanding of
how our programs are run. I need to file some issues to improve the
documentation a little, but I remain pretty pleased with how it's set
up. Good job past me!

[workshop]: https://github.com/kamalmarhubi/shell-workshop

After the workshop, I spent a bit of time deciding what to do next. I
think I've settled on adding a memory mapped message builder to [Cap'n
Proto][capnp]'s C++ library. This was requested in a [recent mailing
list posting][capnp-mmap-thread]. It'll get me poking at the innards of
Cap'n Proto a little, which would be good for one of my potential bigger
project ideas for my time at RC: a shared memory transport for [Cap'n
Proto RPC][capnp-rpc].

[capnp]: https://capnproto.org/
[capnp-mmap-thread]: https://groups.google.com/d/topic/capnproto/kLQOsxjkjxM/discussion
[capnp-rpc]: https://capnproto.org/rpc.html
