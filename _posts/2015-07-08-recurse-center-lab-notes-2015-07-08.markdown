---
title: Recurse Center lab notes 2015-07-08
date: 2015-07-08
permalink: /blog/2015/07/08/recurse-center-lab-notes
---

Yesterday's tinkering with Bazel and Kythe seeped into today. I attempted to
run Kythe's indexers on its own code base. This was a multi-stage process that
took something like 15 hours of CPU time and generated 90+ GB of intermediate
data. The end result was disappointing: I get a 500 error in the provided web
service.

It was definitely fun to use Blaze / Bazel again though! While Kythe didn't
work out, I'm vaguely tempted to use Bazel on some personal stuff, but it needs
more build rules before that becomes non-painful. I need to take a slightly
closer look at what is available already. It looked like someone was adding
build rules for Rust... 

The rest of the day was taken up by being non-productive, followed by doing
some more ES6, followed by napping for too long. We'll try again tomorrow!
