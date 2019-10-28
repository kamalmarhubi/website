---
layout: post
title: "What happened to my hg advent series?"
date: 2019-10-27
---

Last December, I started a series called `hg advent`. It was meant to in the spirit of advent of code: 24
daily posts of me learning how to use
mercurial. But it stopped at the forebodingly titled sixth post—[debugging
hg-git]—which ended with a hopeful sounding sentence

> This means I can much more easily use mercurial day-to-day, so this series
> should hopefully get more grounded in real use.

This is a short post about what went wrong!

[debugging hg-git]: /blog/hg-git-debug/

# git interop was critical

I don't have time to start a green-field project just to learn a new version
control system. I needed to be able to use mercurial on repos I work with
regularly, which all use git. I decided the most likely way to succeed was to
do this on my work repo.

At the time, it looked like the only realistic option was [hg-git]. hg-git works
by cloning the git repo, then applying the commits to a mercurial repo. It does
this in a way that lets it go from a git commit to the corresponding mercurial
changeset and back again. You can then work locally using mercurial, and it'll
translate your work back to git commits whenever you push.

[hg-git]: https://hg-git.github.io/

# hg-git just didn't work for me

I ran into a few issues right off the bat with hg-git. [It didn't play nice][phase-issue] with
setting the default phase for new commits to `secret`. There's an easy
workaround that I decided I could live with: don't do that. Instead either set
the default phase to draft, or set it to public during a pull. This is annoying
but easy to script or alias around.

[phase-issue]: https://bitbucket.org/durin42/hg-git/issues/266/hg-git-interacts-poorly-with-phasesnew

Much harder to work around was the import process just failing. The initial
import was also really slow, much slower than I expected for running against a
five your would repo with under a hundred thousand commits. I left it running
overnight twice, and each time it failed after some unknown number of hours. At
this point, I decided that I wasn't likely to be able to rely on hg-git, and
kind of gave up.

# revisiting mercurial with hgit?

A few weeks ago, I came across an [RFC code submission][hgit-rfc] for mercurial to have an extension
that works directly on git repositories. This gets around the slow and error
prone import process in hg-git by simply not doing it. I don't know anything
about mercurial's internals, so I have no idea where the realy hairy semantic
issues are, but it's promising at least.

It clearly states it's not production code, so for now I'll check in on it
occasionally, and who knows: maybe I'll get to actually try out mercurial!

[hgit-rfc]: https://phab.mercurial-scm.org/D6734
