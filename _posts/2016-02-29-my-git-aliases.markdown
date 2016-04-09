---
title: My git aliases
date: 2016-02-29T14:26:27-05:00
---

<small>_This post has been [translated into Russian][russian]._</small>

[russian]: http://softdroid.net/moi-psevdonimy-v-git

I have a few little aliases in [my `.gitconfig`][gitconfig] that I find useful
and that I thought I'd share.

[gitconfig]: https://github.com/kamalmarhubi/dotfiles-git/blob/master/.gitconfig


# git root: print the absolute path of the repository root directory

~~~
# Print absolute path of repo root directory
root = rev-parse --show-toplevel
~~~

This is useful in the shell if you end up `cd`ed to somewhere deeper in the
repository but want to apply a command at the root, eg for a rename:

~~~
somewhere/in/repo$ sed -i 's/oldname/newname/g' $(git root)/**
~~~


# git detach: get to a detached HEAD state on purpose

~~~
# Get to a detached HEAD state on purpose! Usage: `git detach [REF]`
detach = !sh -c 'git checkout $(git rev-parse ${1:-HEAD})' --
~~~

Usually a detached HEAD state is something you don't want to be in, but I've
wanted this in a few instances, so I added it as an alias. Eg, sometimes I want
to experiment but not actually create a branch. Running `git detach` gives me a
‘branch’ I can make changes to, but without needing to name it, and without needing
to delete it afterwards. If it turns out I do want to save the work I did, I
can always `git checkout -b new-branch-name`, or `git checkout old-branch-name
&& git merge --ff-only HEAD@{1}`.

This alias uses shell expansion by invoking `sh -c`. The bang runs a shell
command, then `sh -c` runs the rest through `sh`. The final `--` is necessary
to send any additional arguments to go to `git checkout` rather than `sh`.


# git sha1: print short sha1 of a commit

~~~
# Print short sha1; usage: `git sha1 [REF]`
sha1 = !sh -c 'git rev-parse --short ${1:-HEAD}' --
~~~

Most often used with `xsel` or `pbcopy`, as in `git sha1 | xsel -i` to copy the
current commit's short sha1 to the clipboard.

I use `sh -c` for shell expansion again, though here it's just for supplying a
default ref.


# git gh-url: get the GitHub URL for a repository

~~~
# Get the GitHub URL for a GitHub repository. Usage: `git gh-url [REMOTE]`
gh-url = "!f() { \
	if ! remote=${1:-$(git config --get \
		branch.$(git symbolic-ref --short HEAD).remote)}; \
	then \
		echo no remote specified and could not get remote for HEAD; \
		exit; \
	fi; \
	if ! remote_url=$(git config --get remote.$remote.url); \
	then \
		echo "could not get URL for remote \\`$remote\\`"; \
		exit; \
	fi; \
	case $remote_url in \
		git@github.com:*.git) \
			repo=$(echo $remote_url \
				| sed 's/git@github.com:\\(.*\\).git/\\1/');; \
		https://github.com/*) \
			repo=$(echo $remote_url \
				| sed 's+https://github.com/\\(.*\\).git+\\1+');; \
		*) \
			echo "\\`$remote\\` does not appear to have " \
				"a GitHub remote url: $remote_url"; \
			exit 1;; \
	esac; \
	echo https://github.com/$repo; \
}; \
f"
~~~

This one is pretty self-explanatory in terms of usage, but it illustrates
another pattern of git aliases: defining shell function and immediately calling
it. The general pattern is this:

~~~
my-alias = "!f() { \
	echo COMMANDS GO HERE, ESCAPING NEWLINES WITH \
		BACKSLASHES, AND TERMINATING WITH SEMICOLONS; \
}; \
f"
~~~

Because we're in a string inside a config file, there's an annoying amount of
escaping necessary, but you get the hang of it fairly quickly.
