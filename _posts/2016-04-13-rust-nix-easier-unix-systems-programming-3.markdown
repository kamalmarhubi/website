---
title: "Rust + nix = easier unix systems programming <3"
date: 2016-04-13
---

Lately I'm writing lots of [Rust], and I'm particularly interested in systems programming on unix. I've been using and contributing to a library called [nix][nix][^not-that-nix], whose mission is to provide ‘Rust friendly bindings to *nix APIs’.

In this blog post, I hope to convince you that you might want to reach for Rust and nix the next time you need to do some unix systems programming, especially if you aren't fluent in C. It's no harder to write, you won't have to write more code, and it makes it much easier to avoid a few classes of mistakes.

[rust]: https://www.rust-lang.org/
[nix]: https://github.com/nix-rust/nix
[^not-that-nix]:
    Confusingly for some, the library has nothing to do with the [Nix package manager][nix-pkg-mgr], [NixOS], or any of the related projects.

[nix-pkg-mgr]: https://nixos.org/nix/
[nixos]: https://nixos.org/


## First off, what is systems programming?

The term systems programming can mean all sorts of things, and depends a lot on context and who you're talking to. For this blog post, here's my incredibly precise definition:

> Systems programming is programming where you spend more time reading man pages than reading the internet.

(As an aside, this means that systems programming is programming you can happily do on the subway if you happen to be in one of those places that doesn't have connectivity in the tunnels, like New York. I definitely did this when taking the Q train back from the [Recurse Center][rc] late at night!)

[rc]: https://www.recurse.com/

Here are a few examples of things that fall under this definition:

  - managing processes (eg, [`fork(2)`][fork], [`execve(2)`][exec], [`waitpid(2)`][waitpid], [`signal(2)`][signal], [`kill(2)`][kill])
  - working with files (eg, [`open(2)`][open], [`ftruncate(2)`][ftruncate], [`unlink(2)`][unlink], [`read(2)`][read], [`write(2)`][write])
  - network programming (eg, [`socket(2)`][socket], [`setsockopt(2)`][setsockopt], [`listen(2)`][listen], [`sendfile(2)`][sendfile])
  - Linux containers (eg, [`clone(2)`][clone], [`unshare(2)`][unshare], [`pivot_root(2)`][pivot_root], [`mount(2)`][mount])
  - interacting with hardware (eg, [`ioctl(2)`][ioctl], [`mmap(2)`][mmap])

[fork]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html
[exec]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/exec.html
[waitpid]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/waitpid.html
[signal]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/signal.html
[kill]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/kill.html

[open]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/open.html
[ftruncate]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/ftruncate.html
[unlink]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/unlink.html
[read]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/read.html
[write]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/write.html


[socket]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/socket.html
[setsockopt]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/setsockopt.html
[listen]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/listen.html
[sendfile]: http://man7.org/linux/man-pages/man2/sendfile.2.html

[clone]: http://man7.org/linux/man-pages/man2/clone.2.html
[unshare]: http://man7.org/linux/man-pages/man2/unshare.2.html
[pivot_root]: http://man7.org/linux/man-pages/man2/pivot_root.2.html
[mount]: http://man7.org/linux/man-pages/man2/mount.2.html

[mmap]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/mmap.html
[ioctl]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/ioctl.html


## fork(2) and kill(2): an example of how badly things can go

Here's a fairly innocuous looking C program that uses [`fork(2)`][fork] and [`kill(2)`][kill] to spawn and kill a process:

~~~
#include <signal.h>
#include <unistd.h>

int main(void) {
        pid_t child = fork();
        if (child) {  // in parent
                sleep(5);
                kill(child, SIGKILL);
        } else {  // in child
                for (;;);  // loop until killed
        }

        return 0;
}
~~~

This program compiles with no errors or warnings, not even with `-Wall -Wextra -Werror`. I recommend you don't run it though, and here's why. From the POSIX specification for [`fork(2)`][fork]:

> Upon successful completion, fork() shall return 0 to the child process and shall return the process ID of the child process to the parent process. Both processes shall continue to execute from the fork() function. Otherwise, -1 shall be returned to the parent process, no child process shall be created, and errno shall be set to indicate the error.

[fork]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/fork.html

And from the specification for [`kill(2)`][kill]:

> If pid is -1, __sig shall be sent to all processes__ (excluding an unspecified set of system processes) for which the process has permission to send that signal.

[kill]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/kill.html

Putting the two together, that program could really ruin our day. If the `fork()` call fails for some reason[^fork-failures], we store `-1` in `child`. Later, we call `kill(-1, SIGKILL)`, which tries to kill all our processes, and most likely hose our login. Not even `screen` or `tmux` will save us![^rachel-fork-citation]

It's a pretty scary failure mode, and neither the library nor the language do anything at all to prevent us from having a terrible day.

[^rachel-fork-citation]:
    I first came across this issue in [a post on Rachel by the Bay][rachel-fork]—which incidentally is a great blog!

[rachel-fork]: http://rachelbythebay.com/w/2014/08/19/fork/

[^fork-failures]:
    There are two main ways `fork(2)` can fail:

      - the system is out of memory
      - we're at our process limit

    Either of these can happen, and code should be ready if they do!


## Why fork and kill go so terribly together

I believe the main issue here is that the C library forces us to try and stick several meanings into one value. For `fork(2)`, the return value is conveying *three* different things all at once:

- whether or not the call succeeded (return value `-1`)
- if it succeeded, whether or not we are in the child (return value `0`)
- if we are the parent, what the child's PID is (strictly positive return value)

That's a lot of information for one poor little `pid_t`—usually a 32-bit integer—to convey!

In the case of `kill(2)`, the `pid` parameter conflates several different behaviors. From [the POSIX specification][kill-desc] again:

> - If pid is greater than 0, sig shall be sent to the process whose process ID is equal to pid.
> - If pid is 0, sig shall be sent to all processes (excluding an unspecified set of system processes) whose process group ID is equal to the process group ID of the sender, and for which the process has permission to send a signal.
> - If pid is -1, sig shall be sent to all processes (excluding an unspecified set of system processes) for which the process has permission to send that signal.
> - If pid is negative, but not -1, sig shall be sent to all processes (excluding an unspecified set of system processes) whose process group ID is equal to the absolute value of pid, and for which the process has permission to send a signal.

[kill-desc]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/kill.html#tag_16_286_03


Some extra badness comes from C's way of treating all non-zero integral values as truthy in conditions, so our `if (child)` check takes the true branch even when `fork()` failed and returned `-1`!

The combination of `-1` as failure value from `fork(2)` and as a special value to `kill(2)` is unfortunate and makes this example especially bad. But other functions treat a `pid` value of `-1` in a special way too, so even if we didn't call `kill(2)` this could still turn out badly. For example, if we called [`waitpid(2)`][waitpid] instead, we'd end up either blocking execution waiting for termination of a child that doesn't exist, or reaping a child that some other thread is waiting for. While they won't ruin our system in quite the same way as `kill(-1, sig)`, neither are failure modes that should be so easy to end up in!


## How nix and Rust help with the fork / kill problem

Here's what this example would look like in Rust + nix:

~~~
extern crate nix;

use nix::sys::signal::*;
use nix::unistd::*;

fn main() {
    match fork().expect("fork failed") {
        ForkResult::Parent{ child } => {
            sleep(5);
            kill(child, SIGKILL).expect("kill failed");
        }
        ForkResult::Child => {
            loop {}  // until killed
        }
    }
}
~~~

We'll go over it in detail below, but for now we'll just notice that the structure and length are really similar to the C version. But it's much safer, and won't go on a process killing rampage!

The nix wrapper for `fork(2)` does two things to make it much easier to avoid accidentally killing all our processes. Both use [Rust's enums][book-enums], which are effectively tagged unions.

[book-enums]: https://doc.rust-lang.org/stable/book/enums.html


### Separating the parent and child returns with an enum

For the success case, nix's `fork()` makes a really great use of a custom enum. Instead of returning just a plain `pid_t`, it returns a `ForkResult` type. The `ForkResult` enum looks like this:

~~~
pub enum ForkResult {
    Parent {
        child: pid_t
    },
    Child
}
~~~

We can read this definition as saying that a `ForkResult` is either `Parent` or `Child`. If it's `Parent` then it contains a `pid_t` value named `child`, while if it's `Child` then it contains no value. Rust has [a pattern matching syntax][book-patterns] for easily checking which variant an enum value is. If we have a variable `fork_result`, we can pattern match on it like this:

~~~
    match fork_result {
        ForkResult::Parent { child } => {
            // stuff to do if we're in the parent
        }
        ForkResult::Child => {
            // stuff do do if we're in the child
        }
    }
~~~

[book-patterns]: https://doc.rust-lang.org/stable/book/match.html#matching-on-enums


### Separating the success and failure cases with Result

The other big thing Rust does to help is having a [`Result`][std-result] type that's used to represent the return from functions that can fail. Similar to how `ForkResult` separated the parent and child cases, the built-in `Result` type separates successes from failures. It looks like this:

~~~
#[must_use]
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
~~~

[std-result]: http://doc.rust-lang.org/std/result/enum.Result.html

We can read this definition as saying that a `Result<T, E>` is either `Ok` to indicate success, or `Err` to indicate failure. If it's `Ok`, it contains a `T` value, while if it's `Err` it contains an `E` value. For a specific case, you'd set `T` to be the successful return type, and `E` to be the type of error that can happen. And the `#[must_use]` attribute tells the compiler to warn us if we ignore a `Result` return value.[^compiler-warned]

[^compiler-warned]:
    Fun fact: when I first wrote the example for this post, I forgot to check the return value from `kill()`. Woops! But the compiler helpfully warned me that I was ignoring a `Result` return value:

    ~~~
    $ cargo build
       Compiling fork-rs v0.1.0 (file:///home/kamal/projects/talks/2016-03-24-rc/fork-rs)
    src/main.rs:13:13: 13:34 warning: unused result which must be used, #[warn(unused_must_use)] on by default
    src/main.rs:13             kill(child, SIGKILL);
                               ^~~~~~~~~~~~~~~~~~~~~
    ~~~



For nix's `fork()` function, the return type is `Result<ForkResult, Errno>`: our happy case is the `ForkResult` type we talked about earlier. Our sad case is an `Errno` value, which is simply an integer the OS uses to tell us why our call failed.

We *could* match on the return value of `fork()` directly like this:

~~~
    match fork() {
        Ok(ForkResult::Parent { child }) => {
            // stuff to do if we're in the parent
        }
        Ok(ForkResult::Child) => {
            // stuff do do if we're in the child
        }
        Err(errno) => {
            // stuff to do if there was an error
        }
    }
~~~

However, Rust has some idioms useful for dealing with `Result` values that make code a little bit tidier. The one we'll rely on in this post is [`expect()`][result-expect]. It unwraps the success value from an `Ok` result, or panics with a given error message if called on an `Err` result. It's a handy way to just crash the program with a semi-useful error message when an error happens. That's pretty much perfect for prototyping, or for quick and dirty programs.

With `expect()`, our match only has to consider the success cases:

~~~
    match fork().expect("fork failed") {
        ForkResult::Parent { child } => {
            // stuff to do if we're in the parent
        }
        ForkResult::Child => {
            // stuff do do if we're in the child
        }
    }
~~~

If `fork()` failed, our program will exit with an error message that looks something like this:

~~~
thread '<main>' panicked at 'fork failed: ENOMEM', ../src/libcore/result.rs:709
~~~

This tells us what failed, and why. For our tiny program, that's enough to see what went wrong. For a more complicated program, we could ask for a backtrace by setting `RUST_BACKTRACE=1` in the environment.

If you want to find out more about error handling in Rust, The Rust book has [a chapter][book-error] with a fantastic and detailed look at different approaches. I highly recommend reading it!

[result-expect]: http://doc.rust-lang.org/std/result/enum.Result.html#method.expect
[book-error]: https://doc.rust-lang.org/stable/book/error-handling.html


## Join us!

I've really been enjoying doing this kind of programming in Rust. So much that I became a maintainer for nix! We've been exploring a few ways of using Rust's features to help make systems programming safer and easier to not mess up.

If this kind of thing interests you too, come help out! We have [good first bug label][first-bug] on our issue tracker, as well as a [mentored bug label][mentor]. We'd love to have your input and your help!

[first-bug]: https://github.com/nix-rust/nix/issues?q=is%3Aissue+is%3Aopen+label%3AE-good-first-bug
[mentor]: https://github.com/nix-rust/nix/issues?q=is%3Aissue+is%3Aopen+label%3AE-mentor

<br />

---

<small>*Thanks to Ant6n Dubrau, Bryan Newbold, Dan Luu, Julia Evans, and Mathieu Guay-Paquet for feedback on drafts of this post.*</small>
