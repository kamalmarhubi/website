---
layout: post
title: "hg advent -m '04: there is no index'"
date: 2018-12-05
---

This one will be really really short. I didn't get a chance to hop into
#mercurial and try to debug hg-git.

So instead here's a very initial observation from using mercurial on a toy
repo: there is no index / staging area. Instead, everything that's changed in
the working directory will be committed.

This reminds me of part of `hg help revsets` that I didn't quote yesterday:

> The reserved name "." indicates the working directory parent. If no working
> directory is checked out, it is equivalent to null. If an uncommitted merge
> is in progress, "." is the revision of the first parent.

There's no room for a staging area concept like the index if the working
directory has a parent: it means that the working directory is the in-progress
changeset that will be created by `hg commit`. In git, that in-progress commit
is what is stored in the index. and conceptually `HEAD` is its parent.

The reason this small observation gets it's own post is that it's going to
require a mental shift in how I VCS. A huge amount of my workflow involves `git
add --patch`[^how-i-git]. A quick google suggests that you can achieve similar
things via `hg commit --interactive` (formerly [`hg record`][hg-record], an
extension!), which is a relief. But it still feels conceptually different to
me.

[hg-record]: https://www.mercurial-scm.org/wiki/RecordExtension
[^how-i-git]:  I promised a how-I-git post to go with this series. I'll try to write it this weekend, as it'll be useful to be able to point to.

That's all I have for today!
