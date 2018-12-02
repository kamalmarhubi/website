---
layout: post
title: hg advent init
date: 2018-12-01
---

This is the start of a daily series in the run-up to Christmas where I learn mercurial. This first post will be about what's making me want to do this in the first place.

I tried briefly a while back, but I didn't get very far. It's annoying to feel
useless using a tool that does approximately the same thing as another tool you
know well. I'm hoping that some daily and deliberate practice will get me
further!

I've been interested in mercurial from a distance for a year or so now. This is partly because it has cool features, and partly because I use git in an
idiosyncratic way that I think could fit better with mercurial. I'll start off talking about the features that initially made me interested. And I'll write a companion post about how I use git which will hopefully help this make more sense.

Bear in mind I've basically never used mercurial, so there is probably a bit of wrongness in what I'm writing here. :-)



# changeset evolution

I'm a pretty heavy user of git-rebase. When you rebase in git, there's no
connection between the rewritten commits, and the ones they're replacing.
Usually this is fine, but sometimes I'd like to be able to diff between the
now-version and the then-version of a commit. Short of some reflog magic and
making assumptions like you never change commit subjects, there's no easy way to
do this in git.

With [changeset evolution][evolution] in mercurial, changesets can point to their earlier versions. If you think of rebase-like operations as changing history, then evolution can give you the history of the history.

[evolution]: https://www.mercurial-scm.org/wiki/ChangesetEvolution

# phases

In git, there's this convention / recommendation that you don't modify visible
history. The precise meaning of visible depends on the circumstances, but in
the popular centralized workflow this would be the main branch of the central
repo.

There's nothing that enforces this, so it's a really easy mistake to make. Pushing an unrelated history up to the main branch of a project with lots of contributors breaks all of those contributors current PRs. They'll have to rebase, and if everyone is lucky there won't be conflicts.

To prevent this kind of thing, people often install update hooks. Or if using a hosted service like github, gitlab, or bitbucket, you can set an option to prevent force pushing. This saves you from modifying public history, and depending on the exact setup may also save you from accidentally pushing work in progress. But all this depends on the remote preventing you from doing damage. In mercurial, your local `hg` commands can stop you from making these mistakes.

In mercurial, changesets (commits) can be in one of three different [phases]:
- secret
- draft
- public

[phases]: https://www.mercurial-scm.org/wiki/Phases

Secret changesets can be modified but not shared. Draft changesets can be both
modified and shared. Public changesets can be shared but not modified. Commands
in mercurial enforce these restrictions: for example,`hg push` will prevent you from
pushing a secret changeset, and `hg commit --amend` will prevent you from rewriting a public changeset.

# unnamed branches

I spend a lot of time in detached head states. This might make more sense after I've written the "how I use git" post. From what I read, mercurial makes it a lot easier and safer to work with unnamed branches. You can do this in git---and I do!---but inevitably something gets a bit lost in the reflog. I want to see how mercurial handles that workflow.

# absorb

This is the feature that pushed me over the edge and made me want to really try learning mercurial. There's [a really great post][absorb-post] about it by Gregory Szorc. Like I said earlier, I rebase a lot. I also frequently have a few [stacked changes][stacked] in the works, or even out for review at the same time.

[absorb-post]: https://gregoryszorc.com/blog/2018/11/05/absorbing-commit-changes-in-mercurial-4.8/
[stacked]: https://jg.gg/2018/09/29/stacked-diffs-versus-pull-requests/

Doing little fixups from code review can be kind of annoying in that setup: you have to rebaseand shuffle things around. [Autosquash] helps a lot: you can commit with [`--fixup`][fixup] to say which commit you're fixing. But it's still kind of a pain.

[autosquash]: https://git-scm.com/docs/git-rebase#git-rebase---autosquash
[fixup]: https://git-scm.com/docs/git-commit#git-commit---fixupltcommitgt

Mercurials `absorb` supposedly makes all that pain go away! Here's its synopsis

> incorporate corrections into the stack of draft changesets

It sounds like exactly what I want! You just make changes to your working directory, and then run `hg absorb`. No committing required. It tries to find which changesets your working directory changes belong to, and then amends those changesets. And if, like me, you're worrying about ambiguous changes that could belong to more than one changeset, well don't! It leave those changes uncommitted in the working directory. And because it only works on draft changesets, it won't accidentally stick changes in the middle of public history.

# it's written in python

This one's a bit of a plus and minus. The fact that it's in python rather than something like C results in perceptible startup delays. As a quick example, in a tmpfs on my laptop, it takes under 3ms to `git init`, and slightly under 90ms to `hg init`. Even `hg --version` takes around 70ms. This is solidly in the range where you can see the lag.

The mercurial folks have a project called oxidation to rewrite parts of it in rust. Initially the `hg` command will continue to be in python, and some operations will call into rust. But I think eventually they want to invert that so that `hg` is a rust binary that calls into python. This should help bring down lag on common operations.

But putting performance aside, python is really approachable. Far more approachable than C, and I'd say somewhat more approachable than bash. This means you can customise it in ways more advanced than aliases without resorting without resorting to bash or libgit2. For example, the absorb extension is about [1000 lines of pretty readable python][absorb-src]! If it turns out it doesn't do _exactly_ what I want, I think I could prod at it until it does. 

[absorb-src]: https://www.mercurial-scm.org/repo/hg/file/tip/hgext/absorb.py

# so here's to three-and-a-bit weeks of mercurial

I'm excited to try this experiment out. And the external commitment in the blog
series makes me epsilon more likely to actually follow through! And who knows,
maybe I'll switch. I started using vim somewhat arbitrarily on 2009-01-01 as
part of a learn-more-unixy-things resolution", and here I am almost 9 years
later typing a blog post in vim. :-)

Oh and I'd love to hear your experience if you use mercurial as a git client.
If I do switch, that's where I'll end up.


<!--
---

[^details]:
    eg, I spent a non-zero amount of time thinking about: whether to spell it
    ‘Center’ or ‘Centre’; how many times to include pair programming in the
    ‘uncomfortable list’; and what symbol set to use for footnotes: numbers or  *,
    †, ‡, §, …—I eventually settled on using the built-in [footnote
    feature][kramdown-footnotes] of the [kramdown] Markdown processor used by
    Jekyll

[kramdown]: http://kramdown.gettalong.org/
[kramdown-footnotes]: http://kramdown.gettalong.org/syntax.html#footnotes
-->
