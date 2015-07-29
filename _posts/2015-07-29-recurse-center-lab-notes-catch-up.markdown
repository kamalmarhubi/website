---
title: Recurse Center lab notes catch up
date: 2015-07-29T00:01:49-04:00
---

I'm way behind on posting here. Here is a quick rundown of what I've been up to.

The biggest thing is that I managed to run [Kythe]! It took me half a week to
get it to work with a hello world program, but once I had that I was good to
go. This meant I got to play with [Bazel] again, which was nicely familiar.

[bazel]: http://bazel.io/
[kythe]: http://kythe.io/

A lot of the confusion was that remote Bazel repositories don't quite work for
C++ in the current state. My approach of referring to the Kythe indexer from a
Bazel repository with my hello world program just wasn't going to work, even
though I kept trying. In the end, I just copied the binaries into my tree and
set up the configuration needed to run them.

I gave a demo of the web UI that comes with Kythe, run against some data
structure code that had Alice written. There's a somewhat hilarious blowup in
size: 7 kB of source results in a couple of serving database tables. I also did
a hackish port of the SQLite build over to Bazel so I could run Kythe over it.
The 1.5 MB of source resulted in a ~80 MB database. The flipside is that the
API server responds to most queries in a few milliseconds.

Keeping with Kythe, I'm currently working on an alternative web UI. The
provided one is really good to understand how nodes are linked together in the
graph structure that Kythe builds up, but it's too noisy and clunky to use as a
day-to-day code exploration tool. As an example, it shows the syntactic
relationship of a variable reference being a child of the function it occurs
in.

I'm pretty excited about this project because one of the things I wanted to do
at RC was to learn JavaScript (well, ECMAScript 2015). The [Kythe web
UI][kythe-ui] is in [Clojurescript] using [Om], which I'm not currently
interested in learning, so I'll be starting my own from scratch.

Just to keep things slightly weird, I'm using [RxJS], the Reactive Extensions
for JavaScript. I'm trying to emulate the architecture used in [Elm] in place
of the [Flux] architecture that's more commonly used with React. The [Elm
architecture][elm-arch] is pleasantly functional, consisting of

- a model type `Model`;
- an action type `Action` that specifies all the kinds of actions that exist;
- an update function `update :: Action -> Model -> Model`; and
- a view function `view :: Model -> Html` that renders the model.

These are all wired up with functional reactive stuff—[Signals] in Elm,
[Observables] in RxJS. The user input events are translated into a stream of
actions. A stream of models is created by folding the update function over the
action stream. Finally, the view is kept up to date by mapping the view
function over the model stream.

I have no real idea what I'm doing, so I'm starting off with translating the
[Elm TodoMVC][elm-todomvc] implementation to this cobbled together set of
tools. This should be done fairly soon, then I'll go back to the Kythe code
browser.

[kythe-ui]: https://github.com/google/kythe/tree/master/kythe/web/ui/src-cljs/ui
[clojurescript]: https://github.com/clojure/clojurescript
[om]: https://github.com/omcljs/om
[rxjs]: https://github.com/Reactive-Extensions/RxJS
[flux]: http://facebook.github.io/flux/
[elm]: http://elm-lang.org/
[elm-arch]: https://github.com/evancz/elm-architecture-tutorial/
[signals]: http://package.elm-lang.org/packages/elm-lang/core/2.1.0/Signal
[observables]: https://github.com/Reactive-Extensions/RxJS/blob/master/doc/api/core/observable.md
[elm-todomvc]: https://github.com/evancz/elm-todomvc
