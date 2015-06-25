---
title: "Recurse Center lab notes 2015-06-24: playing with mmap"
date: 2015-06-24
permalink: /blog/2015/06/24/recurse-center-lab-notes
---

Today I wrote some code! None to show, just experiments. I'm investigating
`mmap` and its interactions with `mprotect` and `ftruncate`. I got some good
responses to yesterday's email thinking about approaches to a memory mapped
message builder for Cap'n Proto. In particular [a message from Paul
Pelzl][paul-message] expanded on my memory protection / `SIGSEGV`-based idea
and suggested playing tricks with `MAP_FIXED` to repeatedly remap the file as
it grows.

I started doing some investigation to see what happens when you mix `mmap`,
`mprotect`, and `ftruncate`. Here's some stuff I found out:

- it is possible to create a non-zero length mapping with `mmap` on a
  zero-length file, eg, one just created by `touch` for the purpose
- on Linux, attempts to access any of the memory result in `SIGBUS`
- a `SIGBUS` handler can call `ftruncate` to extend the backing file; the
  access will then succeed!

[paul-message]: https://groups.google.com/d/msg/capnproto/kLQOsxjkjxM/u8iNvbd6c6wJ

This suggests an even simpler approach than what emerged on the mailing list:

1. `mmap` a huge amount of address space backed by the target file, which
   starts off empty
2. allocate an initial segment of 4GB—the maximum in Cap'n Proto's encoding
3. have a `SIGBUS` handler call `ftruncate` to extend the file whenever an
   attempt is made to reach beyond the end

An initial page can be set aside for a segment table to allow compatibility
with the existing message readers. On closing of the file, the segment size can
be written in at whatever length was actually set aside on disk. There would
likely be an opportunity to shrink the segment down so it only occupies as much
space as was actually used; in this case the file could be truncated to that
size.

This approach has a few nice features:

- it's compatible with the existing flat array and stream based message readers
- it does not require sparse file support
- only one system call needs to be made in the signal handler

I still need to do a bit more research. In particular, I'm wondering:

- will this work on non-Linux systems? The [Linux `mmap` manpage][mmap-linux]
  mentions the `SIGBUS` behaviour, as does [the NetBSD one][mmap-netbsd], but
  [the FreeBSD][mmap-freebsd] and [OS X][mmap-osx] manpages do not; [OpenBSD's
  manpage][mmap-openbsd] says it will give a `SIGSEGV` instead, and helpfully
  points out that POSIX says this situation should be a `SIGBUS`.
- how do I keep track of which regions belong to which files? The signal handler
  has the address that called the fault, but I'll have to have some way to look
  it up.
- how should the filis file lookup structure interact with threads? Is it a
  global table for which we incur some synchronisation cost? Is it a
  thread-local table, and we require that a message only be written to by the
  creating thread?

I'm hoping to answer at least the first question tomorrow, mostly for
curiosity's sake. I'm happy to be Linux-only for now; I'm sure that if this
doesn't work on other platforms, one of the other approaches will. After
testing this on someone's Mac, I can start on a proof of concept message
builder, which will hopefully inform the design more. 

[mmap-linux]: http://man7.org/linux/man-pages/man2/mmap.2.html
[mmap-freebsd]: https://www.freebsd.org/cgi/man.cgi?query=mmap&sektion=2
[mmap-osx]: https://developer.apple.com/library/mac/documentation/Darwin/Reference/ManPages/man2/mmap.2.html
[mmap-netbsd]: http://netbsd.gw.com/cgi-bin/man-cgi?mmap++NetBSD-current
[mmap-openbsd]: http://www.openbsd.org/cgi-bin/man.cgi/OpenBSD-current/man2/mmap.2?query=mmap&arch=i386

[^rc-explainer]:
    Conversations with people at RC make me realise I'll need to take some
    time to explain what I'm doing in a dedicated post some time soon!
