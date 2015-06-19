---
title: Recurse Center lab notes 2015-06-18
date: 2015-06-18
permalink: /blog/2015/06/18/recurse-center-lab-notes
---

Lots of progress on [`lsaddr`][lsaddr] today! Unlike yesterday, most of these
commits are writing code rather than poking at it:

~~~
$ git log --since=yesterday --reverse --oneline
0a695b0 Get IPv6 addresses from /proc/net/if_inet6
683d9c2 Fix typo in Makefile
8d61702 Add debug target to Makefile
686995b Add --list-interfaces option
8f2d3c4 Sort output of --list-interfaces
db95d00 Comment why we replace colon with null
ccf04d5 Introduce struct str_list to remove need for triple pointers
2209807 Use struct str_list in struct args
99cc8db Check interfaces exist using entries from /proc/net/dev
~~~

[lsaddr]: https://github.com/kamalmarhubi/lsaddr

Some highlights: I now feel like I've got this `lfind` thing under control: I'm
using it to filter non-existent interfaces from the user's arguments, as well
as to make sure I only output IP addresses for the specified interfaces. I also
implemented a `--list-interfaces` option that does what it says. All this
information is pretty scattered and disorganised. Here are my sources:

`/proc/net/dev`
: contains a list of interfaces and some other data in a fairly
  parse-unfriendly layout—two header rows, and the interface names have a colon
  appended

`/proc/net/if_inet6`
: contains a list of IPv6 addresses along with some other data in a more
  parse-friendly format, except that the IP addresses are given as 128-bits
  hex-encoded, no colons and no zero-contractions—not ready for human
  consumption

`ioctl(SIOCGIFCONF)`
: a system call that needs to be made on a socket file descriptor to get a list
  of interfaces with IPv4 addresses as `struct sockaddr_in`

So far `--include-loopback` and `--include-link-local` don't do anything. I
started some refactoring to make them easier to impelment. I should be able to
get this wrapped up pretty soon!

I also gave a presentation on writing manpages with [AsciiDoc]. The tl;dr was
that it's really easy! Take a look at the [AsciiDoc source][manpage-source] for
the `lsaddr` manpage. It starts off:

~~~
lsaddr(1)
=======

NAME
----
lsaddr - list active IP addresses

SYNOPSIS
--------
[verse]
*lsaddr* [ *-46* ] [*--include-loopback*] [*--include-ipv6-link-local*] [ _interface_ ... ]
*lsaddr --list-interfaces*

DESCRIPTION
-----------
List IP addresses of the specified __interface__s. By default, lists all IPv4
and IPv6 addresses, with the loopback interfaces and IPv6 link-local omitted.
~~~

And here's a screenshot of the formatted output in my terminal:
![screenshot]

[screenshot]: /static/lsaddr-manpage.png

[asciidoc]: http://asciidoc.org/
[manpage-source]: https://github.com/kamalmarhubi/lsaddr/blob/master/lsaddr.1.txt

That's it for now. I should be getting back to IPC stuff soon!
