---
title: Rust: it's like Haskell and C++ had a baby... kinda
---

I watched Rust from the outside for over two years, thinking one day I'd get
around to learning it. I was especially interested because it looked like it
took stuff I liked from Haskell—algebraic data types, type classes (traits)—and
stuff I liked from C++11/14—control over allocation and copies, notions of
ownership—and put them into one language. I had a conversation with a colleague
where we agreed it was like Haskell and C++ had a baby. How could I *not* like it?

I've finally started actually writing Rust. Luckily, it turns out I *do* like
it. But I've been thinking about how closely it feels like Haskell and C++, or
in particular how *unlike* them it feels. I'm going to try and get down some of
why I think that is.


# How Rust is unlike Haskell

From a language-feature-checklist point of view, it definitely sounds like there's a lot in
common. Eg, they both have algebraic data types, pattern matching, and type classes. But writing Rust feels almost nothing like writing Haskell to me.


## Look

There are some really obvious superficial difference. Rust is more like Java or
C++ than it is like Haskell. It's got curly braces, and angular brackets. It
emphasises methods with a distinguished receiver over free functions. Functions
are not curried, and there's no partial applications, so there tend to be more
parentheses. Putting all that together with the prevalent method chaining style,
code ends up taking more vertical space.

Here's a quick dot product example[^dot-product] that shows how things look different. First,
in Rust:

[^dot-product]:
    This example comes from [Niko Mastakis' series on implementing parallel
    iterators], which is a great read!

[parallel-iter]: http://smallcultfollowing.com/babysteps/blog/2016/02/19/parallel-iterators-part-1-foundations/

~~~rust
vec1.iter()
    .zip(vec2.iter())
    .map(|(i, j)| i * j)
    .sum()
~~~

then in Haskell:

~~~haskell
sum . map (uncurry (*)) $ zip vec1 vec2

-- or, a more succinctly:

sum $ zipWith (*) vec1 vec2
~~~


## Feel

### Memory management

Rust has a clear distinction between stack and heap allocation, and values and
references. There's an overall preference to avoiding allocations where
possible, which is felt throughout the standard library and ecosystem. This is
something that I think only comes up in performance-critical Haskell code.

The emphasis of stack allocation

RAII

Eg, MutexGuard vs MVar.

### Type system

Traits are like type classes, except:
- no higher kinded types
- coherence!
- no overlapping instances allowed (changing soon: specialization)

fundeps?

### Conventions
Conventions:

Error handling: pervasive use of `Either` (`Result` in Rust) for signally
errors. This is helped by the `From` conversion trait, `Error` trait, and `try`
macro.

NB I haven't used Haskell since ~2008-2009, so anything I saw about idioms and
conventions is likely to be out of date. Still, despite Rust being a much
younger language and community, it seems to have a stronger set of conventions.
It might be better to say the community prefers to have conventions to letting
language features be used in whatever way. A few ways this shows up:

- conversion method names (TODO: LINK)
- conversion traits, pervasively used
- error handling


# How Rust is unlike C++

Syntactically:

Braces, dots, double colons, angle brackets, ampersands, asterisks. Pretty
familiar territory!

Expression oriented: `if/else` is an expression, `match` (souped up `switch`) is
an expression. Blocks are expressions, leading to a preference to leave off
`return` if returning from the end of a function.

`match` does pattern matching with exhaustiveness checking.

Closer to semantics:

No implicit conversions

Instead, if you want to accept a convertible type, there are conversion traits
and you can accept `Into<T>`.

Type system:

immutable by default!!

ownership is elevated to a language construct

traits are like concepts lite (?)
traits enable some uses of templates, eg for containers. but there is no
specialization (yet!)

methods accept a `self` paramter that allows clarity on `const` and non-`const`
methods.

methods can consume `self`, which is not possible in C++ (no borrow checker to
enforce)

Explicitness:

accepting an arg by ref requires syntax at the call site

(Maybe refer to goog style guide around non-const ref?)

Algebraic data types
Pattern matching

More traits (huh what did I mean?)

Error handling: return type rather than exceptions

Library niceties:

C++ is missing some really basic types in its libray that would be great to
have as standard. Here are a couple of examples:

- [`&[T]`] and [`&str`][str]: packaged pointer / length for arrays and strings: built in
from the start! C++17 will hopefully have `string_view` for strings. Google has
[`StringPiece`][string-piece]. Cap'n Proto's `kj` library has
[`ArrayPtr`][array-ptr] and [`StringPtr`][string-ptr] for this. These are *so*
important for avoiding allocations, and it's great to have an ecosystem-wide
standard for it.

- [`Option`][option] type for explicit nullability.

[option]: http://doc.rust-lang.org/std/option/enum.Option.html
[slice]: http://doc.rust-lang.org/std/primitive.slice.html
[str]: http://doc.rust-lang.org/std/primitive.str.html
[array-ptr]: https://github.com/sandstorm-io/capnproto/blob/3d58431c3edaee0b3436528390385e25ee53ef45/c%2B%2B/src/kj/common.h#L1127
[string-ptr]: https://github.com/sandstorm-io/capnproto/blob/096ed204894565010a3ca3ac6c8f0695c5de9150/c%2B%2B/src/kj/string.h#L47

[string-piece]: https://code.google.com/p/chromium/codesearch#chromium/src/base/strings/string_piece.h

Conventions:

C++ can end up being a different language in each project or organization that
uses it. I'll speak mostly about ‘modern C++’, being C++11, C++14, ignoring blah
blah.

Rust has clear preference for signalling errors through return values.
Conversions are explicit.

Error handling: C++ is "all over the place": use exception / not. Return values
(StatusOr), sentinel values, others. Rust just uses `Result`. Panicking is
available, which uses the exception handling machinery. However it's not meant
to be used for "normal" errors, instead there's a preference to use `try` and
`Result`.
