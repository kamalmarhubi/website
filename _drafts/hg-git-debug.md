---
layout: post
title: "hg advent -m '06: debugging hg-git'"
date: 2018-12-06
---

I've been delinquent in this series, sorry about that! Again, flying got the best of me: I was busy during the day on Friday, and then flew overnight. Saturday was recovery, and now it's Sunday. I'm not flying again before Christmas, so here's hoping I can keep up the daily habit for the rest of the series!

In the [second post][hg-extensions], I lamented my inability to use hg-git. [Over on lobsters][lobsters-comment], Jordi---a fellow Montrealer---said the bug I was seeing was confusing to the assembled minds of #mercurial, and that the clone should have worked. This was encouraging, but I haven't had time to get back to this until today. But now I think I've figured out the root cause!

[hg-extensions]: http://kamalmarhubi.com/blog/hg-extensions/
[lobsters-comment]: https://lobste.rs/s/utiigb/hg_advent_m_02_extensions#c_2hfhfi

# phases and filtered repos

In the [first post][hg-advent-init], I called out [phases] as one of the features that got me interested in mercurial. I excitedly included the following in my `.hgrc` before ever running mercurial:

    [phases]
    new-commit = secret

[phases]: https://www.mercurial-scm.org/wiki/Phases
[hg-advent-init]: http://kamalmarhubi.com/blog/hg-advent/#phases

Pretty self-explanatory: it tells mercurial to put new commits in the secret phase. This prevents accidental pushing of work in progress: you'd have to explicitly push a revision that is secret for it to get sent out.

This next bit is going to be a bit hand-wavy, as I didn't spend much time looking at the mercurial internals. But my skimming leads me to believe that there's a way to get different views of a repo, eg including or excluding secret or closed or obsolete changesets. Looking up a changeset that's filtered out in the view raises an error whose error is exactly one of the ones I was seeing:

    except (error.FilteredIndexError, error.FilteredLookupError):
        raise error.FilteredRepoLookupError(_("filtered revision '%s'")
                                            % pycompat.bytestr(changeid))

([source][filtered-error])



[filtered-error]: https://www.mercurial-scm.org/repo/hg/file/4.8/mercurial/localrepo.py#l1281

# how hg-git clones a git repo

With hg-git: running `hg clone` against a git repo does two things:

1. clone the git repo as a git repo and store it under the .hg directory
2. import all the git commits as mercurial changesets, storing some metadata to allow back-and-forth translation

Roughly, the second step runs through all the git commits from the roots upwards, and imports them one by one. To make this easier, let's just think about a completely linear history. Going root commit upwards means each commit being imported should have had all its parents imported first. That's important since the imported mercurial changeset for a git commit needs to refer to the imported changeset for the parent git commit. 

While doing the import, hg-git maintains a couple of maps: one from git commit sha1s to hg changeset ids, and another that is the inverse.

# where it all goes wrong

Because I had set `phases.new-commit` to `secret`, the changesets hg-git creates are all in the secret phase.  But from what I can tell, the mercurial repo object that hg-git uses filters out secret commits.

So here's what happens:

1. hg-git imports the root commit with sha1 `ababab` as a root changeset with id `121212` whose phase is secret
2. hg-git goes to import that next commit whose sha1 is `cdcdcd`, and for this it needs the mercurial changeset id of the parent, `ababab`
3. hg-git looks up the `ababab` in the git-sha1-to-hg-commit-id map, and finds `121212`
4. hg-git then looks up this changeset id in the mercurial repo object
5. the repo is filtering this revision out since its secret, so it results in that `FilteredRepoLookupError`
6. boom

Now hg-git has a couple of options related to phases: [`hggit.usephases`][usephases] and [`git.public`][public], but these affect other aspects of the import. Neither of them change either the phase of new commits, or the filter on the repo object.

[usephases]: https://bitbucket.org/durin42/hg-git#markdown-header-hggitusephases
[public]: https://bitbucket.org/durin42/hg-git#markdown-header-gitpublic


# what's next

I've [filed an issue][issue] against hg-git to figure out what the right fix is, and will hopefully submit a patch once that's figured out.

But in the meantime, I've just temporarily set `phases.new-commit` to `public` when doing the `hg clone` and can actually clone git repos. This means I can much more easily use mercurial day-to-day, so this series should hopefully get more grounded in real use.

[issue]: https://bitbucket.org/durin42/hg-git/issues/266/hg-git-interacts-poorly-with-phasesnew
