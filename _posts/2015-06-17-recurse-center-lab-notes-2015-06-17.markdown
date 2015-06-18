---
title: Recurse Center lab notes 2015-06-17
date: 2015-06-17
permalink: /blog/2015/06/17/recurse-center-lab-notes
---

Not a lot to report today: I overslept and only got in at 11, and then left at
6 to go ringing. I did bring another recurser—Jess—along with me, so that was
cool. I think she'll be back, too! I left ringing early to go and checkout the
[Keyboardio Model 01][keyboardio] in midtown. It's a very attractive piece of
hardware, and has me tempted to order one. I'll need a second keyboard at some
point, right?

[keyboardio]: http://www.keyboard.io/

On the [`lsaddr`][lsaddr] front, I figured out how to make `lfind` go. I don't understand
why, but I'll take my win for now. I'm also parsing `/proc/net/if_inet6` to get
IPv6 addresses, though not yet including them in the output. I mostly pushed
characters around in source files:

[lsaddr]: https://github.com/kamalmarhubi/lsaddr

~~~
$ git log --since=yesterday --reverse --oneline
a08386b Fix typo in manpage
16fbff4 Change int to size_t in a couple of places
69300ac Use designated initializers instead of statements
614ef75 Rename ipv4_fd to sockfd and use AF_INET6
8d9aacc Check interface arguments exist
d21ecec Run clang-format
8f9fb9c Properly handle user specified interfaces
6d75545 Comment the repeated ioctl(SIOCGIFCONF)
3f03089 Improve handling of unspecified arguments
88a2eb5 Reformat struct initializer
8ea9224 Properly handle the IP version flags (IPv4 only)
bf08afa Move manpage doctype to a flag to a2x
cef15c4 Change manpage source file suffix to .1.txt
~~~

Tomorrow's work on `lsaddr` will be to finish off IPv6 listing first thing in
the morning, and then do some restructuring to make option handling easier:

- get list of interfaces from /proc/net/dev
  - `int list_interfaces(char ***interfaces, size_t *num_interfaces);`
- check against that list instead of using `ioctl`
- have a `struct address { char [ADDRESSBUFLEN] addr, char [IFNAMSIZ]
  interface, enum {ipv4 = 4, ipv6 = 6} ip_version }`
- have an array of `struct address` sorted by interface, version, addr
- internal list of interfaces with pointers into list of `struct address`

Other random things:

- I read [Introducing the Journal][journald], the post introducing `journald`.
  I'm new to this `systemd` world so it was interesting to get some background.
  The ‘frequently asked questions’ are pretty terrible though, in that
  intending-to-be-funny-but-mostly-condescending way that's a bit too common in
  tech.
- I went to yoga in the early afternoon, which was a good idea. I need to go
  more often. Physical activity is good.

[journald]: https://docs.google.com/document/pub?id=1IC9yOXj7j6cdLLxWEBAGRL6wl97tFxgjLUEHIX3MSTs
