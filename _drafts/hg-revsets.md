---
layout: post
title: "hg advent -m '03: revsets :o'"
date: 2018-12-03
---

Today I'm going to follow some advice given to me in [a lobste.rs comment][comment] on the first post: read [`hg help revsets`][revsets]. (Thanks @ngoldbaum!)

[comment]: https://lobste.rs/s/f9n7lk/hg_advent_init#c_g7ap6x
[revsets]: https://www.mercurial-scm.org/repo/hg/help/revsets

Full disclosure: because of the hg-git setback, I haven't started using mercurial in any real way yet. But I'm told that my clone-single-commit-repo should have worked, so I hope to hop onto #mercurial on freenode and help / get help debugging. But the result is that this post is still based on things I read rather than things I experienced, though it's stuff I'm reading from mercurial's output rather than "the internet".

Anyway, I get the feeling this post is going to result in some knowingly smug expressions on mercurial regulars' faces, together with some kind of "why can't everyone just get what I'm talking about when I say revsets?" thought. (Sorry, I can't help.)

This post will just be excerpts from `hg help revsets` together with my reactions, thoughts, musings, and so on. This help section begins with a very humble

> Mercurial supports several ways to specify revisions.

but then very quickly proceeded to blow my mind. Let's get started!


# Specifying single revisions

There are a few standouts in even this really basic section.

> A plain integer is treated as a revision number. Negative integers are
> treated as sequential offsets from the tip, with -1 denoting the tip, -2
> denoting the revision prior to the tip, and so forth.

From this I glean that `0` refers to the first changeset in the repo, which is handy. There's no easy way to refer to this in git though most use cases I have for that are curiosities.

> A 40-digit hexadecimal string is treated as a unique revision identifier.

This is much like git. Unambiguous prefixes work, too. The identifier is unique and unambiguous across multiple clones, while the revision numbers could be different. So it seems like revision numbers are meant to be local shorthand.

> The reserved name "null" indicates the null revision. This is the revision
> of an empty repository, and the parent of revision 0.

This one is great. There's no way to talk about the parent of the first commit in git. And once upon a time, you couldn't rebase under it. I still have a habit of starting repos off with an empty commit via `git commit --allow-empty -m 'Root commit (empty)'` even though it's not needed. I sometimes think about creating the One True Empty Commit and cloning from it, which is exactly what `null` in mercurial is.

> Finally, commands that expect a single revision (like "hg update") also
> accept revsets (see below for details). When given a revset, they use the
> last revision of the revset. A few commands accept two single revisions
> (like "hg diff"). When given a revset, they use the first and the last
> revisions of the revset.

This sounds reasonable, but once you get to "seeing below for details", it starts to make you think of all kinds of possibilities.


# Specifying multiple revisions

This is where it gets really fancy and mind-expanding.

> Mercurial supports a functional language for selecting a set of revisions.
> Expressions in this language are called revsets.
> 
> The language supports a number of predicates which are joined by infix
> operators. Parenthesis can be used for grouping.

Now, let's take a look at a few of those predicates!

## `adds()`, `modifies()`, `removes()`, `file()`

These all take a pattern and include changesets that adds, modifies, removes files matching the pattern. `file()` is a shorthand for any of those effects.

## `author()`, `date()`, `desc()`, ...

These allow specifying changesets by metadata: author, date (takes an
interval), and description. There are a bunch of others that look at more fields.

## `merge()`, `tag()`, `branchpoint()`

These are predicates related to the repo structure: merge changesets, tagged changesets, and changesets with more than one child.

## `first()`, `last()`

These take the first or last revisions in a revset.

## `parents()`, `children()`, `ancestors()`, `descendants()`, `commonancestors()`

These ones actually take revsets and build a new revset. For example, in mercurial you could refer to the equivalent of `git merge-base A B C D E` as `last(commonancestors(A | B | C | D | E))`.


# Putting it together

Here's a quick example: who other than me modified `file.txt` in the last week?

    hg log -r 'modified(file.txt) && not author(kamal) && date(-7)'

# Revset aliases

You can define an alias for a revset, which you can then use anywhere a revset is expected. This seems really useful! For example, an `unreleased` all changes since the last tagged release could be defined as

    [revsetalias]
    unreleased = not(ancestors(tag()))

You can even define aliases that take arguments! This is the example from the revsets help:

    [revsetalias]
    issue(a1) = grep(r'\bissue[ :]?' ## a1 ## r'\b|\bbug\(' ## a1 ## r'\)')

It's a bit ugly because of the `##` string concatenation operator, but this defines an `issue()` alias that looks for a few different syntaxes of mentioning bugs or issues. You can then throw this in a revset and get, for example, the changesets in the last week related to issue 1234:

    hg log -r 'issue(1234) && date(-7)'


# Until next time

I'm starting to get bored of reading now, so I hope tomorrow I can figure out what's up with hg-git. This revset algebra seems like it would fit my brain quite nicely, but I can't know that for sure yet.
