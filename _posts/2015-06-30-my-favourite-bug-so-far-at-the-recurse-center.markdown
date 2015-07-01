---
title: My favourite bug so far at the Recurse Center!
date: 2015-06-30
---

[Yesterday's shell weirdness mystery][post]: solved! This was definitely my favourite
bug so far here at [RC], so it gets its own post!

[post]: http://kamalmarhubi.com/blog/2015/06/29/recurse-center-lab-notes/#python-shell-writing-weirdness
[rc]: https://www.recurse.com/

As a quick recap, [Sophie] and I were trying to get pipes to work in the shell
she was writing in Python. We ended up in a situation where `ls | head` and `ls
| wc` worked, but `yes | head` did not. `head` would terminate just fine, and
only ten lines of `y` were printed, but `yes` continued to run and consume lots
of CPU.

Oh, and this happened on Sophie's OS X machine, but not on my Linux one. Both were
running Python 2.7.10.

[sophie]: http://sfrapoport.github.io/

To break this down a bit more, we've got four commands we're dealing with: two
which produce data on standard output—`ls` and `yes`—and two which consume
data on standard input—`wc` and `head` [^wc-head-stdout]. We can divide these
by whether they produce or consume a bounded or unbounded amount of data:

- producers
  - `ls`: bounded
  - `yes`: unbounded
- consumers:
  - `head`: bounded
  - `wc`: unbounded

[^wc-head-stdout]:
    `wc` and `head` both also produce data on standard output, but only in
    response to data on standard input; this isn't interesting because of which
    side of the pipe we put them on

Now let's look at the pipelines we were running again:


|              | producer  | consumer  |
|:------------:|:---------:|:---------:|
| `ls | head`  | bounded   | bounded   |
| `ls | wc`    | bounded   | unbounded |
| `yes | head` | unbounded | bounded   |


Our issue comes up when we have an unbounded producer with a bounded consumer.
But only on OS X. 

So now we're all caught up, here's what happened today. Sophie messaged me this
morning saying:

> I think I found something that will help us:
> [http://www.chiark.greenend.org.uk/~cjwatson/blog/python-sigpipe.html][py-sigpipe]
>
> PS: I wouldn't have found this if you hadn't mentioned exploring SIGPIPE in
> your blog post. I read about it and said, 'I think we have a problem with
> SIGPIPE on OSX/Python!'

[py-sigpipe]: http://www.chiark.greenend.org.uk/~cjwatson/blog/python-sigpipe.html

I _love_ this because that wasn't in the part of the post about our shell bug:
it was just pure coincidence!

That page links to a [bug against Python 3.2][py32-bug]. She did some further
investigation and turned up a [bug against Python 2.7][py27-bug] about this.
Sadly, this was closed as wontfix ‘because it is too late to backport this to 2.7.’

[py32-bug]: http://bugs.python.org/issue1615376
[py27-bug]: http://bugs.python.org/issue1652

The gist of the bugs is this:

- Python sets the disposition for `SIGPIPE` to ignore, and instead checks the
  return value of all its writes for errors; this allows raising exceptions in
  Python code instead of requiring installation of a signal handler
- signal dispositions and handlers are inherited by the child after `fork`
- `execve` resets signals with handlers to their default dispositions;
  otherwise it leaves their dispositions alone 

This results in a process `execve`'d from Python starting off ignoring
`SIGPIPE`. Unless it resets its signal dispositions to the default, it will
receive errors instead of signals on writes to broken pipes. There are a couple
of fixes suggested in the bugs, but neither are applied in Python 2.7. Our
shell launches programs which ignore `SIGPIPE`.

We're getting closer!  All this sounds great, but it doesn't explain the
works-on-Linux-but-not-on-OS X part. This is where it gets really fun. To quote
myself, just a few sentences ago: ‘Python sets the disposition for `SIGPIPE` to
ignore, and _instead checks the return value of all its writes for errors_’.
That last part is really important.

At this point we realised we needed to check what `yes` was actually doing. The
version on my machine comes from GNU coreutils. We can take a look at [the loop
where it writes its output][gnu-loop]:

~~~ c
  while (true)
    {
      int i;
      for (i = optind; i < argc; i++)
        if (fputs (argv[i], stdout) == EOF
            || putchar (i == argc - 1 ? '\n' : ' ') == EOF)
          error (EXIT_FAILURE, errno, _("standard output"));
    }
~~~

[gnu-loop]: https://sources.debian.net/src/coreutils/8.23-4/src/yes.c/#L80

Look at all that error checking! From [`man 3 puts`][linux-man-3-puts], which
covers both `fputs` and `putchar`, we can see that they return `EOF` on error;
and [`error(3)`][linux-man-3-error] prints out a nice error message.

[linux-man-3-puts]: http://man7.org/linux/man-pages/man3/puts.3.html#RETURN_VALUE
[linux-man-3-error]: http://man7.org/linux/man-pages/man3/error.3.html

The [implementation of `yes` on OS X][osx-yes] is short enough that I can excerpt its entire `main`:

~~~ c
int
main(argc, argv)
    int argc;
    char **argv;
{
    if (argc > 1)
        for(;;)
            (void)puts(argv[1]);
    else for (;;)
        (void)puts("y");
}
~~~

[osx-yes]: http://www.opensource.apple.com/source/shell_cmds/shell_cmds-170/yes/yes.c

Look at those unchecked calls to `puts`! At last we have our answer! And to be
sure, I compiled the OS X version on my machine, and we got the same behaviour!
Mystery solved!

I love this bug for so many reasons. In particular, I love that it involves so
many parties. There's the Python bug. There's the OS X `yes` implementation
that doesn't check its `puts` calls' return values. And it shouldn't have to
for the pipe case since it can reasonably expect `SIGPIPE` to have its default
action of terminating the process. Where this got confusing is that the GNU
implementation _does_ check for errors, so we got the OS difference. If Sophie
had been using Linux, we never would have encountered this!

A big takeaway for me here is that it's up to us as developers to deal with
idiosyncrasies in our platform. The fact that there's an acknowledged bug in
Python doesn't mean we can just throw up our hands in a big ¯\\\_(ツ)\_/¯.

