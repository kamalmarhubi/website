---
layout: post
title: "hg advent -m '02: extensions'"
date: 2018-12-03
---

Ok, we're not off to a great start here. First off, this is a day late. For future reference: it's not a great idea to start a daily blog post series from the airport. Jet lag can get you before you've got into the routine.

But far more annoying to me: I had planned to write about how to make [hg-git] go, and excitedly explain how this blog post is being committed and pushed from a mercurial fork of [my site's repo][website-repo]. There was going to be a digression into how it was a bit of a pain to make HTTP auth work for Github private repos, but that I figured it out in the end.

[hg-git]: https://bitbucket.org/durin42/hg-git/overview
[website-repo]: https://github.com/kamalmarhubi/website

Unfortunately, I have simply failed to figure out hg-git at all. I can't get it to work even for locally cloning a repo with a single commit:

    $ git init git-repo
    Initialized empty Git repository in /tmp/git-repo/.git/
    $ cd git-repo/
    $ echo 'A git repo' > README
    $ git add README
    $ git commit -m'README'
    [master (root-commit) e148665] README
     1 file changed, 1 insertion(+)
     create mode 100644 README
    $ cd ..
    $ hg clone git-repo hg-git-repo
    importing git objects into hg
    abort: filtered revision '911b5ec9fbffb99638eb10be7edb803b4898bad9'!
    $ cd ../git-repo
    $ echo "\nDoesn't work." >> README
    $ git add README
    $ git commit -m'Update README'
    [master 6b0ae1d] Update README
     1 file changed, 1 insertion(+)
    $ cd ..
    $ hg clone git-repo hg-git-repo
    importing git objects into hg
    abort: you appear to have run strip - please run hg git-cleanup
    $ echo ":'("
    :'(


So that'll have to wait.

Instead I'll just write something about extensions in general, purely based on things I've read on the internet. 


# extensions are really cool

There are [a lot of features][extensions-list] in mercurial that are not "core", but instead are bundled extensions. This includes at least a couple of the features I called out in the [first post][hg-advent-init]. As a specific example, the feature I called out as pushing me over the edge, `hg absorb`, is an extension.

[hg-advent-init]: http://kamalmarhubi.com/blog/hg-advent/
[extensions-list]: https://www.mercurial-scm.org/wiki/UsingExtensions#Extensions_bundled_with_Mercurial

It does some some fancy stuff that would be hard to make work as well in a `git-absorb` "extension" command, for example modifying history without every touching the working directory, and relying on a bespoke algorithm to find which commits changes belong with. I'm sure I could cobble something together in [bash or perl][gitcore-post][^no-perl] to achieve a similar effect, but it would almost certainly use rebase under the hood, and so have to change the working tree.

[gitcore-post]: http://kamalmarhubi.com/blog/2016/10/07/git-core/
[^no-perl]: I don't actually know perl.

I think mercurial being in Python is at least partly responsible for things like absorb being possible to add in extensions. It lets you use internal APIs without a recompile: just drop a file somewhere and tell mercurial where to find it. And those internal APIs are more accessible than git's for being in Python.

Oh, and hg-git, the thing that lets you---but not me, apparently---use mercurial as a git client? That's an extension too. :-)


# extensions are a papercut

[Once upon a time][rust-issues-post], the rust community used the issue label `I-papercut` to refer to

> Minor issues, nice-to-haves. Some times these are ergonomics issues which in aggregate are quite important.

[rust-issues-post]: https://internals.rust-lang.org/t/how-the-rust-issue-tracker-works/3951

I really like metaphor: you won't die from a single papercut, but it will get in your way. And worse, it could turn you off whatever caused it.

The fact that so many features in mercurial are in extensions is a papercut. It makes a mess of the first-run experience, especially for users coming from git. The out-of-the-box feature set is a bit limited---again, especially for git users---and I've never seen a mercurial guide that didn't suggest enabling _several_ extensions. To me that's a really strong suggestion that something is wrong.

As an example: in the first post, I said I use rebase a lot. Can you rebase in mercurial? Of course, just enable [the extension][rebase]. How about using [meld] to see a diff? Yup, there's [an extension][extdiff]. Colour output in diffs? The answer [used to be an extension][color-ext], but thankfully that's built-in now.

[rebase]: https://www.mercurial-scm.org/wiki/RebaseExtension
[meld]: http://meldmerge.org/
[color-ext]: https://www.mercurial-scm.org/wiki/ColorExtension
[extdiff]: https://www.mercurial-scm.org/wiki/ExtdiffExtension

On a related note, the syntax for enabling an extension is itself a papercut: I find it simultaneously confusing and ugly. Here's what the rebase docs have as how to enable it:

> Enable the extension in the configuration file (e.g. .hg/hgrc):
>
>    [extensions]
>    rebase =

All I can think is "`rebase` equals what, damnit?!" Yes, I now know that the right hand side is the path to a python module (`.py` file, or directory containing `__init__.py`), and that since `rebase` is distributed with mercurial, it's in the default search path, so you don't need to say where to find it. But what's wrong with something like

    [extensions]
    rebase = on  # or True, or anything more yes-like than the empty string


# moving on

Day 3 or 4 will hopefully involve me actually using mercurial. I'm going to give myself a bit more time to figure out hg-git, since it's the only way I can see to fit mercurial into my daily workflow. If I can't, I'll just start doing some mini project or other using mercurial for version control.

Until then![^day-3]

[^day-3]: It's already pretty late on day 3, so "then" had better be soon if I want to keep up.
