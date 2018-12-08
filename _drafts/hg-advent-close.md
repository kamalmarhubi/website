---
layout: post
title: "hg advent -m '05: heads and `hg commit --close`'"
date: 2018-12-06
---

I found out how a couple of key parts of unnamed branches works: changesets can be marked
as closed, and mercurial can keep track of the leaf changesets that aren't
closed. This answers the big question I had of how you can possibly make this
work without getting lost in a sea of work.

This is pretty cool! It means that when a string of commits are done with, eg
because you've decided it's not a useful direction, you can tell mercurial. And
at any point, you can ask mercurial to see what open work exists.

# An example

This is contrived. I'm going to start off with the default Rust skeleton you get from `cargo new`, and translate it into a few languages.

    $ cargo new --vcs=hg hw
         Created binary (application) `hw` project
    $ cd hw
    $ cargo --quiet run
    Hello, world!
    $ hg add .
    $ hg commit -m'Add cargo skeleton'

Ok so now we have saved the initial state: hello world in English. Now let's start on a translation into Spanish:

    $ sed -i 's/Hello/Hola/' src/main.rs
    $ cargo --quiet run
    Hola, world!
    $ hg commit -m'Translate hello into spanish'

Off to a great start! Let's take a look

    $ hg log --graph --template=compact
    @  1[tip]   4ca430c10413   2018-12-06 20:56 -0500   kamal
    |    Translate hello into spanish
    |
    o  0   0345dfb2199e   2018-12-06 20:53 -0500   kamal
         Add cargo skeleton

I'm not sure if world is "mondo" or "mundo" at this point, so I'll switch to translating German. First, let's go back to the skeleton with `hg checkout`[^hg-update]

    $ hg checkout 0
    1 files updated, 0 files merged, 0 files removed, 0 files unresolved
    $ sed -i 's/world/welt/' src/main.rs
    $ cargo --quiet run
    Hello, welt!
    $ hg commit -m'Translate world into german'
    created new head

# Created new head?

Mercurial just told us we created a new head. This means there is a new
lineage, which we can see in the log:

    $ hg log --graph --template=compact
    @  2[tip]:0   472ad1d630ef   2018-12-06 21:04 -0500   kamal
    |    Translate world into german
    |
    | o  1   4ca430c10413   2018-12-06 20:56 -0500   kamal
    |/     Translate hello into spanish
    |
    o  0   0345dfb2199e   2018-12-06 20:53 -0500   kamal
         Add cargo skeleton

Because mercurial is aware of the heads, we can list the heads with `hg heads`:

    $ hg heads
    changeset:   2:472ad1d630ef
    tag:         tip
    parent:      0:0345dfb2199e
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:04:19 2018 -0500
    summary:     Translate world into german

    changeset:   1:4ca430c10413
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 20:56:46 2018 -0500
    summary:     Translate hello into spanish

Let's throw one more language into the mix, before demonstrating how we can
tell mercurial we're no long interested in one of these lines of work.

    $ hg checkout 0
    1 files updated, 0 files merged, 0 files removed, 0 files unresolved
    $ sed -i 's/Hello, world/Salamu, dunia/' src/main.rs
    $ cargo --quiet run
    Salamu, dunia!
    $ hg commit -m'Translate the whole thing into swahili'
    created new head

Now let's check the graph and the list of heads:

    $ hg log --graph --template=compact
    @  3[tip]:0   b14c88540219   2018-12-06 21:18 -0500   kamal
    |    Translate the whole thing into swahili
    |
    | o  2:0   472ad1d630ef   2018-12-06 21:04 -0500   kamal
    |/     Translate world into german
    |
    | o  1   4ca430c10413   2018-12-06 20:56 -0500   kamal
    |/     Translate hello into spanish
    |
    o  0   0345dfb2199e   2018-12-06 20:53 -0500   kamal
         Add cargo skeleton

    $ hg heads
    changeset:   3:b14c88540219
    tag:         tip
    parent:      0:0345dfb2199e
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:18:02 2018 -0500
    summary:     Translate the whole thing into swahili

    changeset:   2:472ad1d630ef
    parent:      0:0345dfb2199e
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:04:19 2018 -0500
    summary:     Translate world into german

    changeset:   1:4ca430c10413
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 20:56:46 2018 -0500
    summary:     Translate hello into spanish

Makes sense.

# Closing a head

Now it's late and my sister, who's fluent in spanish, can't tell me if it's "mondo" or "mundo" tonight. So I'm going to close that branch out to have less work in progress floating around.

    $ hg checkout 1
    1 files updated, 0 files merged, 0 files removed, 0 files unresolved
    $ hg commit --close -m'Close branch: could not get translation of world'

Now if we list the heads, that one won't be shown:

    $ hg heads
    changeset:   3:b14c88540219
    parent:      0:0345dfb2199e
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:18:02 2018 -0500
    summary:     Translate the whole thing into swahili

    changeset:   2:472ad1d630ef
    parent:      0:0345dfb2199e
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:04:19 2018 -0500
    summary:     Translate world into german

It's still there, and shows up in the graph:

    $ hg log --graph --template=compact
    @  4[tip]:1   80da88bcc1a8   2018-12-06 21:21 -0500   kamal
    |    Close branch: could not get translation of world
    |
    | o  3:0   b14c88540219   2018-12-06 21:18 -0500   kamal
    | |    Translate the whole thing into swahili
    | |
    | | o  2:0   472ad1d630ef   2018-12-06 21:04 -0500   kamal
    | |/     Translate world into german
    | |
    o |  1   4ca430c10413   2018-12-06 20:56 -0500   kamal
    |/     Translate hello into spanish
    |
    o  0   0345dfb2199e   2018-12-06 20:53 -0500   kamal
         Add cargo skeleton

Actually if we switch away from the closed head, we can see that it gets a different marker in the graph:

    $ hg checkout null
    0 files updated, 0 files merged, 4 files removed, 0 files unresolved
    $ hg log --graph --template=compact
    _  4[tip]:1   80da88bcc1a8   2018-12-06 21:21 -0500   kamal
    |    Close branch: could not get translation of world
    |
    | o  3:0   b14c88540219   2018-12-06 21:18 -0500   kamal
    | |    Translate the whole thing into swahili
    | |
    | | o  2:0   472ad1d630ef   2018-12-06 21:04 -0500   kamal
    | |/     Translate world into german
    | |
    o |  1   4ca430c10413   2018-12-06 20:56 -0500   kamal
    |/     Translate hello into spanish
    |
    o  0   0345dfb2199e   2018-12-06 20:53 -0500   kamal
         Add cargo skeleton

Instead of a letter `o`, it gets an underscore. And if we wanted to exclude it,
we can use our [newfound revset knowledge][revset-post]:

    $ hg log --graph --template=compact -r 'not closed()'
    o  3:0   b14c88540219   2018-12-06 21:18 -0500   kamal
    |    Translate the whole thing into swahili
    |
    | o  2:0   472ad1d630ef   2018-12-06 21:04 -0500   kamal
    |/     Translate world into german
    |
    | o  1   4ca430c10413   2018-12-06 20:56 -0500   kamal
    |/     Translate hello into spanish
    |
    o  0   0345dfb2199e   2018-12-06 20:53 -0500   kamal
         Add cargo skeleton

[revset-post]: hg-revsets

And if we wanted it in the list of heads, we can ask mercurial to include
closed ones

    $ hg heads --closed
    changeset:   4:80da88bcc1a8
    tag:         tip
    parent:      1:4ca430c10413
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:21:16 2018 -0500
    summary:     Close branch: could not get translation of world

    changeset:   3:b14c88540219
    parent:      0:0345dfb2199e
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:18:02 2018 -0500
    summary:     Translate the whole thing into swahili

    changeset:   2:472ad1d630ef
    parent:      0:0345dfb2199e
    user:        Kamal Marhubi <kamal@marhubi.com>
    date:        Thu Dec 06 21:04:19 2018 -0500
    summary:     Translate world into german


# It's starting to make sense

Unnamed branches have been the part of mercurial that have me most intrigued. They're really close to my current workflow in git, but I wasn't sure how it could work. Now that I've seen `hg heads` and `hg commit --close` I can actually imagine how it'd work. I'm still a ways away from knowing if it'll work for me, fingers crossed!


[^hg-update]:
    `hg checkout` is a built-in alias for `hg update`, which updates the
    working directory or switces revisions.  It's equivalent to one of the
    several meanings of `git checkout`, in this case, the `git checkout
    <commit-ish>` variant. I'm not sure I like the use of the word `update`
    here, hence my using the alias.
