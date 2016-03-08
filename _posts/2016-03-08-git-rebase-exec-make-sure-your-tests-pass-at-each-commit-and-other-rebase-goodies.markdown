---
title: "git rebase --exec: make sure your tests pass at each commit! (and other rebase goodies)"
date: 2016-03-08T14:38:55-05:00
---


I learned about the `--exec` option to `git rebase` recently, and I needed to
share! The basic structure is this:

~~~
git rebase --interactive --exec "cmd" some-ref
~~~

This brings up the usual interactive rebase todo list, but with `exec "cmd"`
added after each commit. Saving the todo list as-is will run `cmd` at each
commit between `some-ref` and the current `HEAD`. You can use the short option
`-x`:

~~~
git rebase -ix "cmd" some-ref
~~~

The rebase will stop any time the `cmd` exits unsuccessfully, giving you a
chance to inspect why it failed.


## making sure the tests always pass!

I like to have my pull requests broken up into small commits, eg one that does
a small refactor, one that adds a type, another that uses the new type. In the
run up to actually opening the pull request, I make lots of small
changes—style, documentation, comments, and so on.

These finishing touches affect all the commits, so I do lots of little rebases.
There's a risk of one of the intermediate commits being broken—not passing
tests, not building, having bad style or lint errors. If I was feeling extra
diligent, sometimes I'd manually step through the commits by marking them all
for `edit` in `git rebase -i`. But now it's super easy to get right!

You can even specify `--exec` (`-x`) multiple times, and all the commands will
get run:

~~~
git rebase -i -x "run-tests" -x "run-linter" master
~~~


## other goodies in rebase

After finding out about `--exec`, I wanted to be sure I wasn't missing any
other life changing options or features, so here's a quick tour of things I
discovered.


### the options in the rebase todo list

The `--exec` option wouldn't have surprised me if I ever looked at the options
that pop up in my editor *every time* I use `git rebase -i`! Here they are as
of git version 2.1.4, which is what I'm running:

~~~
# Commands:
#  p, pick = use commit
#  r, reword = use commit, but edit the commit message
#  e, edit = use commit, but stop for amending
#  s, squash = use commit, but meld into previous commit
#  f, fixup = like "squash", but discard this commit's log message
#  x, exec = run command (the rest of the line) using shell
~~~

I don't feel like I was missing out *too* much though. I make good use of
`fixup` and `reword`, as well as the basics: `pick`, `edit`, and `squash`.
`fixup` is great if you make lots of little work-in-progress commits: you can
just squash them and throw away their messages. I find `reword` is useful for
expanding on commit messages in the run up to opening a pull request.


### git rebase --edit-todo

When you run `git rebase -i`, your editor pops up with a list of commits for
you to mess with. I sometimes get my choices wrong, and end up doing `git
rebase --abort` and starting over. It turns out I don't need to! `git rebase
--edit-todo` will bring up the same list and let you modify it.


### git commit --squash, --fixup, and git rebase --autosquash

I didn't know about these at all, but I'm excited to try and get them into my
workflow! I'll just quote the man page because it describes how to use them
very well:

~~~
--autosquash, --no-autosquash
    When the commit log message begins with "squash! ..." (or "fixup!
    ..."), and there is a commit whose title begins with the same ...,
    automatically modify the todo list of rebase -i so that the commit
    marked for squashing comes right after the commit to be modified,
    and change the action of the moved commit from pick to squash (or
    fixup). Ignores subsequent "fixup! " or "squash! " after the first,
    in case you referred to an earlier fixup/squash with git commit
    --fixup/--squash.
~~~

The `--squash` and `--fixup` options to `git commit` take a commit as an
argument, and formats the commit message for use with `--autosquash`.

To make it all even handier, you can set `rebase.autoquash` to `true` in your
gitconfig. [I just did][commit]!

[commit]: https://github.com/kamalmarhubi/dotfiles-git/commit/dee6b4912c077bd06404937854ba053c16f8b880


### git rebase --autostash

The `--autostash` option will stash before starting a rebase, and pop the stash
afterwards. If you ever end up doing a dance like

~~~
git stash
git rebase -i master
git stash pop
~~~

then the `--autostash` option will save you 12 key presses each time. Or you
can set `rebase.autostash` to `true` in your configuration, and it will happen
by default, saving you a whopping *24* key presses!

The man page warns that ‘the final stash application after a successful rebase
might result in non-trivial conflicts’, so I've not set this in my `gitconfig`.
I'll see how useful the option seems first.


### git rebase --no-ff

This forces the rebase to create new commits instead of fast-forwarding over
the unchanged ones. I've manually approximated this in the past to tickle the
continuous integration into running again if I think the failures were spurious
or flakes. This could be useful for repositories where you're not the owner,
and so don't have access to the CI system's retry button.


### changing the merge strategy

The `--strategy` (`-s`) option lets you say what merge strategy to use, and the
`--strategy-option` (`-X`) option lets you set options on the strategy. I just
figured out that, together with `--autosquash`, this can let me do another
pre-pull-request-tidy task: making sure all the intermediate commits are
properly formatted.

If you have some command `format-cmd` that formats your code, then something
like this will format all the commits:

~~~
git rebase -i \
-x "format-cmd && git commit -a --allow-empty --fixup=HEAD" \
--strategy-option=theirs \
origin/master
~~~

Since I now have `rebase.autosquash` set in my `gitconfig`, this results in a
nice tidied up set of commits. I still have to tweak it a bit, as I had to `git
rebase --continue` on an empty diff at some point, but it's looking pretty
great! I will almost certainly turn it into an alias soon.


## sometimes reading docs really pays off

I'm pretty happy with this exploration of `git rebase`.  I'm pretty sure what I
just picked up here with `git rebase` will make some things a *lot* smoother in
the future. It makes me think I set aside a bit of time each week to read some
documentation for tools I use a lot. When I do, I'll write it up!
