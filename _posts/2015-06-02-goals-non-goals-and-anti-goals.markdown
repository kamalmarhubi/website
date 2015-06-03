---
title: Goals, non-goals, and anti-goals
date: 2015-06-02
---

When working on a thing—being at the Recurse Center, say—knowing what you want
to get out of it helps with direction. Identifying clear goals keeps your
focused. For example, I know that I want to know more about [inter-process
communication in Linux][linux-ipc]. I also want to be able to develop for the
browser, which means learning JavaScript and CSS.

[linux-ipc]: http://kamalmarhubi.com/blog/2015/06/01/a-list-of-linux-ipc-mechanisms/

Knowing what some non-goals are can also help. For me, exploring new
programming languages is a non-goal, as I've done enough of it in the past.
Making that explicit simplifies turning down a good chunk of activities at the
Recurse Center that would otherwise distract me from my goals.

But there's another category: anti-goals. These are things I specifically do
not want to work onspend time on while at the Recurse Center. I've identified a
couple of these so far: installing [Kubernetes] on my [CoreOS] cluster, and tinkering with
Google's [Bazel] build system.

[bazel]: http://bazel.io/
[kubernetes]: http://kubernetes.io/
[coreos]: https://coreos.com/

The distinction between non-goals and anti-goals is a bit unclear to me. In the
context of self-directed learning, I have identified a couple of potential
criteria. One is that a non-goal is still educational and could be useful,
while an anti-goal is more like busywork and configuration; the other is that
anti-goals are more tempting to work on. This makes them more likely to
interfere with my goals by drawing me away.

Putting this helps clarify why I had such an unsatisfying first week. I started
off with working on a small patch to someone else's project. It seemed simple
enough, but it turned out to need some [fussing with build and testing
configuration][yaks]. This put most of the time I spent closer to anti-goals
than to goals. Realising this after lunch today, I decided to simply dump that
patch—however close to done it seems—and move on to exploring IPC.

[yaks]: http://kamalmarhubi.com/blog/2015/05/27/controlling-the-yak-stack/

<br />

---

_Thanks to Danielle Sucher, David Albert, and Olivia Jackson for conversations
that set me on the way to this categorisation. I feel like my next few days
will be better than my last few for it!_
