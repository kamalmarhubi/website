---
title: "Recurse Center lab notes 2015-06-29: fun with signals; confusing shell-writing"
date: 2015-06-29
permalink: /blog/2015/06/29/recurse-center-lab-notes
---

# Signals

I finished skimming the chapter on signals in [Advanced Programming in the UNIX
Environmnent][apue]. This took me a few attempts over the last few days, and I
fell asleep reading it at least once. One big takeaway for me was the
distinction between synchronous and asynchronous signals. In a nutshell:

[apue]: http://www.apuebook.com/about3e.html

**synchronous signals**
: these signals are a direct result of execution; examples include

  - `SIGFPE` (stands for *f*loating *p*point *e*xception), which is sent on
    various errors in mathematical functions, such as dividing by zero
  - `SIGPIPE` (stands for... *pipe*), which is sent if there's a write to a pipe
    with no readers
  - `SIGSEGV` (stands for *seg*mentation *v*iolation), which is sent
    to a thread if it tries to access memory in a way it's not allowed to, eg,
    dereferencing a null pointer, reading read-protected memory like kernel memory
  - `SIGBUS` (*bus* error, whatever that is meant to mean[^sigbus]!), which is sent on
    some types of memory access issues, eg, improper alignment, or—the one
    that's exciting to me—attempting to access memory mapped beyond the end of a
    memory mapped file[^sigbus-openbsd]

[^sigbus]:
    I think this is meant to be related to the memory bus, which makes sense for
    unaligned accesses, but not for the memory mapped file situtuation.

[^sigbus-openbsd]:
    From man pages and some experimentation, `SIGBUS` is sent for this
    situation on Linux, FreeBSD, OS X, and NetBSD; OpenBSD's documentation says a
    `SIGSEGV` is sent instead. Feel free to [test it out][signal-test] on your system! On mine:
    
    ~~~
    $ curl https://gist.githubusercontent.com/kamalmarhubi/d8a3b3754bc2f4f62899/raw/333a1c8ebce4a3c7718e740ea4d175d22ca206fd/mmap_signal_test.c | cc -xc -o mmap_signal_test -
    $ ./mmap_signal_test
    Received signal while accessing 0x7facd3268000: Bus error
    Going to ftruncate
    Read: 0
    Wrote and read: 1
    $ █
    ~~~

[signal-test]: https://gist.github.com/kamalmarhubi/d8a3b3754bc2f4f62899

**asynchronous signals**

: these signals can get sent independently of the running process or thread;
  they just show up. Examples include:
  - `SIGINT` (*interrupt*), which is sent when you interrupt a process from the
    keyboard with `^C` at the terminal
  - `SIGKILL`, which is sent when you `kill -9` something to kill it with fire;
    the process terminates, and the signal cannot be caught, blocked or ignored
  - `SIGSTOP`, which is sent when you stop a process with `^Z` at the terminal;
    the process is stopped, and will continue when it's sent a `SIGCONT`
    (*cont*inue)
  - `SIGHUP` (*h*ang*up*), which is sent if the controlling terminal ‘hangs up’,
    ie, is closed—this one is a fun anachronism which serves to us that we're
    still using [programs that pretend to be terminals][terminal-emulator],
    which are in turn mostly pretending to be [teletype machines][tty], and that
    everything sort of pretends to be connected over a phone line

[tty]: https://en.wikipedia.org/wiki/Teleprinter
[terminal-emulator]: https://en.wikipedia.org/wiki/Terminal_emulator


# Python shell writing weirdness

Sophie asked me for some help with the mental model for file descriptors and
pipes and `dup2` and what happens when you fork and other such fun things. There
is a really strange bug we couldn't figure out: `ls | wc` or `ls | head` would
terminate as expected, but `yes | wc` or `yes | head` would not; `yes` would
just keep on running and use loads of CPU, so it was not idle. On OS X only (2
machines tested). On Linux, it worked as expected, with `yes` terminating and
complaining about the broken pipe. We gave up eventually, and I still haven't
got any idea what's wrong. I really wish [`procfs`][procfs] existed on OS X!

[procfs]: http://man7.org/linux/man-pages/man5/procfs.5.html

I wrote a minimized equivalent of our code. Please run it and let me know what
happens! On my machine:

~~~
$ curl https://gist.githubusercontent.com/kamalmarhubi/65c2ea1063e479f0c16b/raw/2459725829891b156f61ec5fc808587846ec07be/pipe_weirdness.py > pipe_weirdness.py
$ chmod +x pipe_weirdness.py
$ ./pipe_weirdness yes head
y
y
y
y
y
y
y
y
y
y
yes: standard output: Broken pipe
yes: write error
$ █
~~~

I'm really interested in ideas on why `yes` wouldn't terminate!

<br />

-----
