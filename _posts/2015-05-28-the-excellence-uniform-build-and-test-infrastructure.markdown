---
title: The excellence of uniform build and test infrastructure
date: 2015-05-28
---

I made some small amount of progress on [yesterday's yaks][yaks]. I have
almost got to the point where think that the tests work on both OS X
and Linux; I just need to get some time on a Mac to be sure.

[yaks]: http://kamalmarhubi.com/blog/2015/05/27/controlling-the-yak-stack/

The experience so far has made me realise how powerful it is to have
uniform build and test infrastructure. At Google, an improvement as
small as I'm making would take not much longer than it takes to write
the code. Everything is in [one reository][monorepo], and all tests are
run with the same [tool][bazel]. As a result, you simply make your
change, run the tests, and then send it for review.

[monorepo]: http://danluu.com/monorepo/
[bazel]: http://bazel.io/

The [project][pycapnp] I'm changing has a test suite. For local testing,
you run either `py.test` to run with the current Python environment, or
`tox` to run a [collection of environments][tox-ini]. In the CI build,
the tests are run with `py.test` under a [set of environments]
[travis-yml] which defined by Travis CI's infrastructure.

[pycapnp]: https://github.com/jparyani/pycapnp
[tox-ini]: https://github.com/jparyani/pycapnp/blob/790bdce72ab3f2b6c203f00eaf07e6d605aa631a/tox.ini#L2
[travis-yml]: https://github.com/jparyani/pycapnp/blob/790bdce72ab3f2b6c203f00eaf07e6d605aa631a/.travis.yml#L6-11

This difference between local tests and tests under CI is quite
frustrating, especially given that the CI runs take many minutes to get
scheduled and run. I can get to the Works On My Machine™ state, but have
the CI build fail because the environment is different. To say I spent
most of yesterday and today refreshing CI build pages wouldn't be too
inaccurate!

Even more, Travis CI recently changed its default from virtualized
infrastructure to containerized infrastructure. The differences seem to
be:

- containerized infrastructure schedules and runs quicker
- containerized infrastructure won't run setuid programs such as
  sudo, which is used in this project's [environment set up]
  [setup-travis].

However, they only applied this new default to repositories created
after some [cutoff date][cutoff].[^sudo-detection] This meant that the
exact same commit that [passes][success] in the upstream CI build [would
fail][failure] in a new clone.[^refactor] Certainly not intuitive!

I'm really interested to see how different projects approach this
problem. Having such a uniform infrastructure is a luxury that not many
can afford!

[setup-travis]: https://github.com/jparyani/pycapnp/blob/aa7d5303193b13880728035c298c635e4fdcbe1c/buildutils/setup_travis.sh#L7-11
[cutoff]: https://github.com/travis-ci/travis-core/blob/7a360299c19011cbd3c0f2bf099a16600048e210/lib/travis/model/job/queue.rb#L88
[success]: https://travis-ci.org/jparyani/pycapnp/builds/61183652
[failure]: https://travis-ci.org/kamalmarhubi/pycapnp/builds/64316652

<br />

---

[^sudo-detection]:
    The attentive reader may wonder about the `sudo_detected` method
    call there: shouldn't that have prevented this from being a
    problem?  Sadly, it looks as though this detection simply looks at
    invocations directly specified in `.travis.yml`, and makes no
    attempt to find indirect invocations.

[^refactor]:
    I'm not even going into how at Google, all projects would
    have been tested with and without the new configuration. It would
    then be [fairly easy][global-change] to send an automated change
    [updating the configuration][sudo-required] to any project that was
    newly failing with the new CI configuration.

[global-change]: http://danluu.com/monorepo/#cross-project-changes
[sudo-required]: https://github.com/jparyani/pycapnp/commit/790bdce72ab3f2b6c203f00eaf07e6d605aa631a
