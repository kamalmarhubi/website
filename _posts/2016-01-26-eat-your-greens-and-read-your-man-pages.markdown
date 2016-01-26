---
title: Eat your greens and read your man pages
date: 2016-01-26T10:19:54-05:00
---

I've been working on a little tool to let unprivileged users run commands in
the context of a container image's filesystem on Linux.  I'm really bad at
names, so let's call it `containy-thing`. That also evokes about the right
level of completeness: it's quite far from done.

Anyway, this is a little field report from working on it. Its moral is: read
your man pages.

<br>

-----

<br>

`containy-thing` uses a few Linux tricks to do its job, the main ones being
[user][user-ns] and mount [namespaces]. It takes a directory to use as the
container root filesystem, and a command to run, and runs the command. If
you've used docker,

[user-ns]: http://man7.org/linux/man-pages/man7/user_namespaces.7.html
[namespaces]: http://man7.org/linux/man-pages/man7/namespaces.7.html

~~~
$ containy-thing /path/to/rootfs /bin/ls -l
~~~

would be similar to

~~~
# docker run my-image /bin/ls -l
~~~

(Note the `$` vs `#` prompts! The main point of this project is that you can
run it as an unprivileged user; docker needs superuser privileges.)

I was trying to test it out and had an extracted root filesystem for an
Ubuntu-derived image. I attempted to run `/bin/ls -l`, but it kept failing.
Investigating a bit, I found that the `execve(2)` system call to start
`/bin/ls` was failing with `EACCES`.


# read your man pages

I've done bits and pieces of systems programming, and the man pages are pretty
great most of the time. The man page for [`execve(2)`][execve] definitely falls
under ‘most of the time’, and is pretty exhaustive. It lists the circumstances
that could result in `EACCES`:

~~~
       EACCES Search permission is denied on a component of the path prefix of
              filename or the name of a script interpreter.  (See also
              path_resolution(7).)

       EACCES The file or a script interpreter is not a regular file.

       EACCES Execute permission is denied for the file or a script or ELF
              interpreter.

       EACCES The filesystem is mounted noexec.
~~~

[execve]: http://man7.org/linux/man-pages/man2/execve.2.html#ERRORS

I went through these one by one:

* checked that all path components (namely, `/bin`) had the search permission
  (`a+x`) set (`/bin/ls` is not a script, and so there was no script
  interpreter to check)
* checked that `/bin/ls` was there, and was a file
* checked that `/bin/ls`/ had the execute permission (`a+x`)
* double checked the mount to be sure it wasn't `noexec`, which disallows
  execution of all files on the filesystem

Nothing came up. I did this a bunch of times, and still nothing came up.


# no really, read your man pages

Eventually, I finally realized what the problem was. The third reason for
`EACCES` was

~~~
       EACCES Execute permission is denied for the file or a script or ELF
              interpreter.
~~~

but I was reading

~~~
       EACCES Execute permission is denied for the file or a script       
              interpreter.
~~~

I didn't even stop to think that something could be wrong with the ELF
interpreter. I only even vaguely understand what an ELF interpreter does! But
after wasting probably a day and a bit trying to figure this out, I finally
took a look. The interpreter for x86-64 programs on Linux is at
`/lib64/ld-linux-x86-64.so.2`, at least on Debian and Ubuntu.... and its
permissions were wrong.
