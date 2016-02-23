---
title: "rust-bisect: tracking down the Rust nightly that changed some behavior"
date: 2016-02-23T16:36:47-05:00
---

After a bit more tinkering that expected, I'm finally (almost[^almost]) releasing my first
Rust crate!

[rust-bisect][gh-repo] helps track down when a change—usually a bug!—was
introduced into Rust. Rather than `git bisect` directly on the Rust repository,
it uses nightly builds to speed up the process. This is faster in at least a
couple of ways:

- at over 100 pull requests merged per week, there are far more commits
  to bisect than there are nightly builds
- to run an individual test, all rust-bisect needs to do is download the
  nightly: no slow Rust build at each step!

This was a really fun project to work on! I made a few changes to related
crates, and even had to solve a stereotypical [algorithms interview
problem][bisect]. And of course, there's something extremely satisfying about
seeing it bisect and actually find the right nightly...

[bisect]: https://github.com/kamalmarhubi/rust-bisect/blob/master/src/bisect.rs

# Try it out!

If you want to give it a shot, check out [the repository][gh-repo]—especially the
[example]!  I'd *really* love to hear if this is useful to you. And please [open
issues][issues] if you come across any bugs!

[crates.io]: https://crates.io/
[gh-repo]: https://github.com/kamalmarhubi/rust-bisect
[multirust-rs]: https://github.com/Diggsey/multirust-rs
[mr-release-issue]: https://github.com/Diggsey/multirust-rs/issues/54
[example]: https://github.com/kamalmarhubi/rust-bisect#example
[issues]: https://github.com/kamalmarhubi/rust-bisect/issues

<br>

---

[^almost]:
    It's not on [crates.io] yet because it's using some unreleased changes to
    [multirust-rs]... [soon though][mr-release-issue]!
