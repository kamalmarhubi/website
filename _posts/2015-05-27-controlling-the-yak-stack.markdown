---
title: Controlling the yak stack
date: 2015-05-27
---
<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">A-yak-shaving I will go, a-yak-shaving I will go&#10;I&#39;ll update the CI and then the build&#10;Then implement my feature and push to the source repo</p>&mdash; Kamal Marhubi (@kamalmarhubi) <a href="https://twitter.com/kamalmarhubi/status/603663049313132545">May 27, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

Today, I was trying to make a [small improvement][gh-issue] to
[pycapnp], the Python implementation of [Cap'n Proto][capnp]. This isn't
a very big change: it's mostly adding a few lines at the end of a big
conditional to make the library a better Python citizen.

But the library has [tests][pycapnp-tests] and [continuous integration][pycapnp-ci] set up.
These are both awesome things to have, except for the part where they
don't work for me. Here is my current yak stack[^yak-stack]:

- the [tox] tests don't run properly on Linux, so [I
  modified][commit-tox] the `tox.ini` to remove a setting I think is
  OS X specific, fixing tox on my machine
- I need to make sure the CI passes, so I fork the repo on GitHub, sign
  up for Travis CI, and add my fork to Travis
- the [Travis build fails][build-fail] because my build is using the new container
  based infrastructure, which doesn't allow setuid programs like `sudo`
  to run, so I started trying to port the CI build to the new
  infrastructure

I think the third step was where I could have stopped. If I'd tried
finding out how to opt out of the container infrastructure, I might have
been able to move on.

At least that is a good starting point for tomorrow.



[gh-issue]: https://github.com/jparyani/pycapnp/issues/66
[pycapnp]: https://github.com/jparyani/pycapnp
[capnp]: https://capnproto.org/
[pycapnp-tests]: https://github.com/jparyani/pycapnp/tree/develop/test
[pycapnp-ci]: https://travis-ci.org/jparyani/pycapnp
[tox]: http://tox.testrun.org/
[commit-tox]: https://github.com/kamalmarhubi/pycapnp/commit/965301e9b42e664d43de5f8ea7f7f69648e0b7da
[build-fail]: https://travis-ci.org/kamalmarhubi/pycapnp/jobs/64316653
[container-infra]: http://docs.travis-ci.com/user/workers/container-based-infrastructure/
[commits-ci]: https://github.com/kamalmarhubi/pycapnp/commits/travis-apt-addon

<br />

---


[^yak-stack]: Remember: yak stacks, like x86 stacks, grow down.
