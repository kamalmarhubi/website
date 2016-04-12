---
title: "Rust + nix = unix systems programming <3"
date: 2016-04-12T18:26:50-04:00
---

Lately I've been writing lots of Rust, and I've been particularly interested in systems programming on unix. I've been contributing to a library called [nix], whose mission is to provide Rust-friendly bindings to *nix APIs. In this blog post, I hope to convince you that you might want to reach for Rust and nix the next time you need to do some unix systems programming, especially if you aren't fluent in C.

[nix]: https://github.com/nix-rust/nix

## First off, what is systems programming?

The term systems programming can mean all sorts of things, and depends a lot on context and who you're talking to. For this blog post, here's my incredibly precise definition:

> Systems programming is programming where you spend more time reading man pages than reading the internet.

(As an aside, this means that systems programming is programming you can comfortably do on the subway if you live in one of those places that doesn't have connectivity in the tunnels.)

Here are a few examples of things that fall under this definition:

  - managing processes (eg, `fork(2)`, `exec(2)`, `waitpid(2)`, `signal(2)`, `kill(2)`)
  - working with files (eg, `open(2)`, `ftruncate(2)`, `unlink(2)`, `read(2)`, `write(2)`)
  - network programming (eg, `socket(2)`, `setsockopt(2)`, `listen(2)`, `sendfile(2)`)
  - Linux containers (eg, `clone(2)`, `unshare(2)`, `pivot_root(2)`, `mount(2)`)
  - interacting with hardware (eg, `ioctl(2)`, `mmap(2)`)


## fork(2) and kill(2): an example of how badly things can go!

Here's a fairly inoccuous looking C program that uses [`fork(2)`][fork] and [`kill(2)`][kill] to spawn and kill a process:

~~~
#include <signal.h>
#include <unistd.h>

int main(void) {
        pid_t child = fork();
        if (child) {  // in parent
                sleep(5000);
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

Putting the two togeter, that program could really ruin your day. If the `fork()` call fails for some reason[^fork-failures], we store `-1` in `child`. Later, we call `kill(-1, SIGKILL)`, which kill all your processes, and most likely hose your login. Not even `screen` or `tmux` will save you![^rachel-fork-citation]

It's a pretty scary failure mode, and neither the library nor the language do anything at all to prevent you from having a terrible day.

[^rachel-fork-citation]:
    I first came across this issue in [a post on Rachel by the Bay][rachel-fork]—which incidentally is a great blog!

[rachel-fork]: http://rachelbythebay.com/w/2014/08/19/fork/

[^fork-failures]:
    There are two main ways `fork(2)` can fail:

      - the system is out of memory
      - you're at your process limit

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


Some extra badness comes from C's way of treating all non-zero integral values as truthy in conditions, so our `if (child)` check takes the true branch even when `fork()` failed!

The combination of `-1` as failure value from `fork(2)` and as a special value to `kill(2)` is unfortunate and makes this example especially bad. But other functions treat a `pid` value of `-1` in a special way too. Even if we didn't call `kill(2)` this could still turn out badly. For example, if we called [`waitpid(2)`][waitpid] instead, we'd end up either blocking execution waiting for termination of a child that doesn't exist, or reaping a child that some other thread is waiting for. While they won't ruin our system in quite the same way as `kill(-1, sig)`, neither are failure modes that should be so easy to end up in!


## How nix and Rust help with the `fork(2)` / `kill(2)` problem

The nix wrapper for `fork(2)` does two things to make it much easier to avoid accidentally killing all your processes. Both take advantage of [Rust's enums][book-enums], which are effectively tagged unions.

[book-enums]: https://doc.rust-lang.org/stable/book/enums.html

### The Result type

The biggest thing Rust does to help with all this is having a `Result` type that's used to represent the return from functions that can fail. It looks like this:

~~~
#[must_use]
pub enum Result<T, E> {
    Ok(T),
    Err(E),
}
~~~

Breaking this down: `Result` is generic over the return type `T` and the error type `E`. A `Result` value is either `Ok` and contains a return value of type `T`, or it's `Err` and contains an error value of type `E`. The `#[must_use]` attribute tells the compiler to warn you if you ignore a `Result` return value.

All together, what this means is that the error and non-error cases are very clearly separated, and that the compiler will tell you if you ignore the return value of a function that might fail.

Rust has some idioms for dealing with `Result` values. The one we'll rely on in this post is [`Result::expect()`][result-expect]. It unwraps the `T` from an `Ok` result, or panics with a given error message on an `Err` result. It's a handy way to just crash the program with a semi-useful error message when an error happens. It's good for prototyping or for quick and dirty programs.

The Rust book has [a chapter][book-error] with a fantastic and detailed look at different ways to approach error handling in Rust programs. I highly recommend reading it!

[result-expect]: http://doc.rust-lang.org/std/result/enum.Result.html#method.expect
[book-error]: https://doc.rust-lang.org/stable/book/error-handling.html


### Enums

For the success case, nix's `fork()` makes a really great use of Rust's enums, and returns a `ForkResult` type instead of just a plain `pid_t`. The `ForkResult` enum looks like this:

~~~
pub enum ForkResult {
    Parent {
        child: pid_t
    },
    Child
}
~~~

A Rust enum variant can carry some data with it. In this instance, a  `ForkResult` is either `Parent` and carries the child's pid, or `Child` in which case there's no additional data to convey. Rust has [a pattern matching syntax][book-patterns] for easily checking which variant an enum value is.

[book-patterns]: https://doc.rust-lang.org/stable/book/match.html#matching-on-enums


## Putting it together

In Rust, the `fork()` and `kill()` example would look like this:

~~~
extern crate nix;

use std::thread;
use std::time::Duration;

use nix::sys::signal::{kill, SIGKILL};
use nix::unistd::ForkResult::*;
use nix::unistd::fork;

fn main() {
    match fork().expect("fork failed") {
        Parent{ child } => {
            thread::sleep(Duration::from_secs(5));
            kill(child, SIGKILL).expect("kill failed");
        }
        Child => loop {},  // until killed
    }
}
~~~

The `main()` function is pretty much the same length as the C version, but it's much more correct![^compiler-warned]

[^compiler-warned]:
    Fun fact: when I first wrote this example, I called [`execvp(2)`][execvp] in the child instead of looping, and I forgot to call `expect()` on its return value. Woops! Thankfully, the compiler warned me about ignoring a `Result` return value:

    ~~~
    $ cargo build
       Compiling fork-rs v0.1.0 (file:///tmp/fork-rs)
    src/main.rs:18:13: 18:60 warning: unused result which must be used, #[warn(unused_must_use)] on by default
    src/main.rs:18             execvp(&CString::new("/bin/ls").unwrap(), &[]);
                               ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    ~~~

    [execvp]: http://pubs.opengroup.org/onlinepubs/9699919799/functions/exec.html


## Better than C

SUMMARY HERE WILL GET FILLED IN SOON

<br />

---


