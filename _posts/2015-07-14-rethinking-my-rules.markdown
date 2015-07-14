---
title: Rethinking my rules
date: 2015-07-14
permalink: /blog/2015/07/14/rethinking-my-rules
---

After a conversation with [Julia] and [Greg] about how to approach the
Recurse Center, I'm revisiting my not-at-RC project list. I wrote about
this early on in [Goals, non-goals, and anti-goals][goals-post]. The
gist of it is that I wanted to avoid spending time on things that were
mostly ‘busywork and configuration’. The list of not-at-RC projects was
vaguely:

- [Kubernetes], [CoreOS], and other containery and clustery stuff
- [Bazel], Google's recently open sourced build tool
- [Kythe], Google's recently open sourced source code indexer
- [Prometheus], a monitoring and alerting system

[julia]: http://jvns.ca/
[greg]: https://blog.gregbrockman.com/
[goals-post]: http://kamalmarhubi.com/blog/2015/06/02/goals-non-goals-and-anti-goals/
[kubernetes]: http://kubernetes.io/
[coreos]: https://coreos.com/
[bazel]: http://bazel.io/
[kythe]: http://kythe.io/
[prometheus]: http://prometheus.io/

There's a common thread here of wanting a Google-away-from-Google, at
least in terms of the developer experience. This might be misguided,
because not all problems are Google-sized. The developer experience of a
great code search, unified build system, and cluster scheduler is really
great though! The main problem is I'm not sure how many of these things
require scale to be worthwhile.

Back to the conversation that made me think of changing my rules. Greg
said it was valuable to explore and learn to use tools to give you a
better understanding of how to judge similar things. That resonated with
me: if I want to have informed opinions on things in this space, I have
to use some of them!

With all that in mind, here's what I'm working on at the moment:

- fixed an issue I'd been having with my CoreOS cluster, so that I no longer
  get complaints about failed systemd units on login
- reading up on what's changed in Kubernetes since I read the original design
  docs when it was first released. There's a lot of new concepts in there now,
  and they're [gearing up for a 1.0 next week][kubernetes-1.0][^launch-site],
  so this seems like a good time to dig in!
- I installed [Sandstorm] on a new server in GCE. Eventually I plan to move
  this to the Kubernetes cluster.

[kubernetes-1.0]: http://kuberneteslaunch.com/
[sandstorm]: https://sandstorm.io/

Of course, bringing in a new theme means something has to go to make room. I'm
probably going to drop the priority on learning JavaScript / ES15. Unlike with
the systems programming I was doing, I don't have a strong and clear goal in
mind there. I do expect to get back to `mmap` and my other friends in that
world though. I'll treat Kubernetes and CoreOS as a little break for now, and
then figure out a way to interleave the different kinds of work.

This should be fun!

<br>

---

[^launch-site]:
    Yes, apparently they got a domain just for the launch.
