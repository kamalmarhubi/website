---
title: "Recurse Center lab notes 2015-07-07: back to mmap; other people's code; things I'm not meant to do while at RC"
date: 2015-07-07
permalink: /blog/2015/07/07/recurse-center-lab-notes
---

I went back to `mmap` today. I am finally attempting to put together what I
picked up from experimenting in the last while towards writing a memory mapped
file message builder for Cap'n Proto. Current status:

- I've created a new class with a constructor `MmapMessageBuilder(int
  fd)` that maps the file descriptor
- Things mostly magically work because I'm supplying an alternate
  implementation of an interface
- I am not yet handling the resizing of the backing file in any way
- I am not yet including a segment table at the front of the file

The last one means I can't get the files read back in with the existing
deserializers. I was totally able to see it in the bytes though!

~~~
00000000  00 00 00 00 00 00 01 00  01 00 00 00 57 00 00 00  |............W...|
00000010  08 00 00 00 01 00 04 00  7b 00 00 00 02 00 00 00  |........{.......|
00000020  21 00 00 00 32 00 00 00  21 00 00 00 92 00 00 00  |!...2...!.......|
00000030  29 00 00 00 17 00 00 00  39 00 00 00 22 00 00 00  |).......9..."...|
00000040  c8 01 00 00 00 00 00 00  35 00 00 00 22 00 00 00  |........5..."...|
00000050  35 00 00 00 82 00 00 00  39 00 00 00 27 00 00 00  |5.......9...'...|
00000060  00 00 00 00 00 00 00 00  41 6c 69 63 65 00 00 00  |........Alice...|
00000070  61 6c 69 63 65 40 65 78  61 6d 70 6c 65 2e 63 6f  |alice@example.co|
00000080  6d 00 00 00 00 00 00 00  04 00 00 00 01 00 01 00  |m...............|
00000090  00 00 00 00 00 00 00 00  01 00 00 00 4a 00 00 00  |............J...|
000000a0  35 35 35 2d 31 32 31 32  00 00 00 00 00 00 00 00  |555-1212........|
000000b0  4d 49 54 00 00 00 00 00  42 6f 62 00 00 00 00 00  |MIT.....Bob.....|
000000c0  62 6f 62 40 65 78 61 6d  70 6c 65 2e 63 6f 6d 00  |bob@example.com.|
000000d0  08 00 00 00 01 00 01 00  01 00 00 00 00 00 00 00  |................|
000000e0  09 00 00 00 4a 00 00 00  02 00 00 00 00 00 00 00  |....J...........|
000000f0  09 00 00 00 4a 00 00 00  35 35 35 2d 34 35 36 37  |....J...555-4567|
00000100  00 00 00 00 00 00 00 00  35 35 35 2d 37 36 35 34  |........555-7654|
00000110  00 00 00 00 00 00 00 00  00 00 00 00 00 00 00 00  |................|
*
00001000
~~~

Writing out the segment table mostly requires trawling through someone else's
code base. The Cap'n Proto code is fairly clear and well documented, especially
the user interfaces. Some key innards are not as well documented, however. In
particular, the [arena abstraction][capnp-arena] used to look after the
allocated segments could do with a short overview. I'm most of the way to
understanding how it fits together, but it's been slower than I would have
liked.

[capnp-arenal]: https://github.com/sandstorm-io/capnproto/blob/1702050903e1038acf0556c6eabcf8f99702690d/c%2B%2B/src/capnp/arena.h#L197

The process of understanding the code has been slowed by needing to learn the
[`kj`][kj] library that Cap'n Proto makes heavy use of. I'm trying to find out
the boundaries of what `kj` aims to be, but so far it looks to include:

- replacements for some standard library classes, eg, [`kj::Own`][kj-own]
  is used where you might use[ `std::unique_ptr`][std-unique_ptr], and
  there is a separate [stream abstraction][kj-io] that is used instead of
  [the standard one][std-io]
- an event loop—the C++ standard library still has nothing here; it may
  in 2017 though!
- some templated array classes that seem to be a pointer-length pair;
  there is some macro magic to allow allocating such an array on the
  stack

[kj]: https://github.com/sandstorm-io/capnproto/tree/master/c%2B%2B/src/kj
[kj-own]: https://github.com/sandstorm-io/capnproto/blob/1702050903e1038acf0556c6eabcf8f99702690d/c%2B%2B/src/kj/memory.h#L99
[kj-io]: https://github.com/sandstorm-io/capnproto/blob/master/c%2B%2B/src/kj/io.h

[std-unique_ptr]: http://en.cppreference.com/w/cpp/memory/unique_ptr
[std-io]: http://en.cppreference.com/w/cpp/io

An overview of this library would also be really nice to have. My life
would be a bit easier if this was all STL stuff.

All this frustration has me trying to use Google's [Kythe code indexing
project][kythe] on Cap'n Proto. This in turn has me messing around with their
[Bazel build tool][bazel].  Both of these tools are up there on my
not-while-at-RC list, but I'm giving myself a very short pass to see if I can
port the build over.  Building these tools is pretty slow going...

[kythe]: http://www.kythe.io/
[bazel]: http://bazel.io/

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">current status: waiting for llvm to compile :/</p>&mdash; Kamal Marhubi (@kamalmarhubi) <a href="https://twitter.com/kamalmarhubi/status/618613903593996289">July 8, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
