---
layout: post
title: Eliminating branches in Rust for fun... but not much profit
date: 2015-09-15
---

Last week, I nerd-sniped myself after reading a [blog post][bench-post] about
performance of parser combinators in Rust, and the associated [Reddit
discussion][reddit-thread].

[bench-post]: http://m4rw3r.github.io/parser-combinator-experiments-part-3/
[reddit-thread]: https://www.reddit.com/r/rust/comments/3k0d0d/parser_combinator_experiments_part_3_performance/

The benchmark in the post was parsing a bunch of HTTP requests. The focus was
on improving performance of a function determining if a character formed part
of a token. In the post, it was changed from

~~~
fn is_token(c: u8) -> bool {
    c < 128 && c > 31 && b"()<>@,;:\\\"/[]?={} \t".iter()
                           .position(|&i| i == c).is_none()
}
~~~

which iterates over a list of characters, to

~~~
fn is_token(c: u8) -> bool {
    // roughly follows the order of ascii chars: "\"(),/:;<=>?@[\\]{} \t"
    c < 128 && c > 32 && c != b'\t' && c != b'"' && c != b'(' && c != b')' &&
        c != b',' && c != b'/' && !(c > 57 && c < 65) && !(c > 90 && c < 94) &&
        c != b'{' && c != b'}'
}
~~~

which unrolls the loop into a big boolean conjunction.

On my machine, the benchmark with the original version takes about 106
microseconds to parse a text file containing about 50 HTTP requests. The newer
version takes 73 microseconds, which is about 30% faster.

After reading this, I started thinking something like "I know about CPUs, and
branches are bad because of mispredictions or something", and "LOOK `&&` AT
`&&` ALL `&&` THOSE `&&` BRANCHES!", and imagining fame and fortune for making
it even faster.

This led me down a path of trying to replace that function with a
straight-through series of bitwise operations. First off, let's unpack what's
actually being checked in this function:

- `c < 128`: that `c` represents asn ASCII character
- `c > 32`: that `c` is not a control character or space
- all the `!=` comparisons: that `c` isn't any of those characters
- `!(c > 57 && c < 65)`: that `c` is none of `:;<=>?@`
- `!(c > 90 && c < 94)`: that `c` is none of `[\]`


# Eliminate (almost) ALL THE BRANCHES

My basic idea was to replace all the equality checks with XOR, and all the
boolean ands with bitwise ands. Boolean ands (`&&`) and ors (`||`) have short
circuiting semantics: their right hand sides are only evaluated if necessary.
This means that `&&` has to be implemented as a conditional jump to avoid
evaluating the right hand side if the left hand side is false.

To check I was making any sense, I used the fantastic Godbolt interactive
compiler, which [has Rust support][rust-godbolt]. This confirmed that at least
some of the boolean ands were compiling to conditional jumps, which was enough
for me to go down the bitwise operation path.

[rust-godbolt]: http://rust.godbolt.org/

After spending a bunch of time on translating it to be purely bitwise
operations, I ended up with this performance result:

~~~
test bench_http ... bench:     230,030 ns/iter (+/- 6,303)
~~~

About three times slower. This was... disappointing. Especially because along
the way, a broken benchmark setup had me convinced I sped things up by 60%!

# Why so slow?

I talked about my results with Dan Luu, since he's written about this kind of
thing before. He mentioned that on Haswell, the missed branch prediction
penalty is about 14 cycles. Optimally organised bitwise operations can be
about 4 per cycle.

[dan-luu-post]: http://danluu.com/new-cpu-features/#branches

The original version had about 30 instructions, of which about 6 were branches.
Assuming one instruction per cycle, a really bad branch predictor miss rate
like 50% would be somewhere around 75 cycles.  The bitwise version had about
130 instructions, of which one was a branch. Assume the branch predictor is
always right, that still puts me at 40+ cycles if 3 are in parallel.

In summary, unless the branch predictor was abysmally bad at this code, or I
was amazingly excellent at organizing the bitwise operations, there was no
strong reason to expect my my bitwise version to be faster.

This got me to run `perf stat` on the benchmark to get a look at the branch
prediction hit rate. This turned out to be above 99.5%, which is far from
‘abysmal’.

# Another approach: a lookup table

The discussion also suggested implementing this as a lookup table. This would
reduce `is_token` to a single load which can be inlined wherever it's used.
Rust represents boolean values as bytes, so the table will be 256 bytes which
should mean we'll rarely have to go too far down the cache hierarchy.

Indeed, implementing that gave a modest 6-7%  performance improvement over the
version that sent me off on this investigation:

~~~
test bench_http ... bench:      67,890 ns/iter (+/- 1,913)
~~~

So at least there was a somewhat satisfactory result at the end!

# The moral of the story: validate assumptions, and know more numbers!

There were a couple of big takeaways for me. The first is I could have saved
myself a whole bunch of time if I'd investigated the branch prediction
performance before embarking on this adventure. Instead of assuming the
branching was slowing things down, I would know that it probably wasn't.

The other was knowing some performance numbers offhand could have helped a lot
here. I have a rough idea of L1 / L2 latency, and main memory latency. But I
had no idea how many bitwise operations to expect per cycle, or how bad a
branch misprediction was. I've now added these to my collection, but knowing a
few more rough figures like that would certainly be good!

<br>

---

*Thanks to Dan Luu, Anton Dubrau, and David Turner for useful and interesting
discussions, and to Julia Evans for reading drafts of this post.*


<!--

# How I implemented a bitwise version
Here's how I built up my bitwise version. For my own sanity, and ease of
translation, I decided to use `0x0` as false and `0x1` as true. Implement first
and improve afterwards if it seems hopeful. I started off using the fact that
`x ^ y` is all zeros if and only if `x` and `y` are equal. Or, put another way,
if `x != y` at least one bit in `x ^ y` is set. This will be handy since most
of our comparisons are checking inequality with a fixed byte. This lets us
write

~~~
fn eq(x: u8, y: u8) -> u8 {
    not(is_non_zero(x ^ y))
~~~

We need `is_non_zero` to return `0x1` or `0x0`; we'll get to that in a second.
Once it does, the `not` operation is easy: just XOR with `0x1`:

~~~
fn not(x: u8) -> u8 {
    x ^ 1
}
~~~

As for this `is_non_zero` operation, that's a bit trickier. Fortunately, [the
internet][stackoverflow]! Here's the implementation:

~~~
fn is_non_zero(x: u8) -> u8 {
    ((x | ((!x).wrapping_add(1))) >> 7)
}
~~~

What we want to do is return `0x1` if any bit of `x` is set, and otherwise
return `0x0`. We'll work out how this works from the outside in. If we can get the most signicant bit set in The way this works is to force the most significant bit to be set
if any bit is set. Then the `>> 7` will bring the most significant bit down to
be the least significant bit, and we'll have 

Finally, we've got comparisons with `128` and `32`. Both of these are powers of
two, which makes it a bit easier to check! If the most significant bit is set,
then the number is greater than or equal to `128`, so we bitwise and with `128`
and check if that's non-zero:


~~~
pub fn ge_128(x: u8) -> u8 {
    is_non_zero(128 & x)
}
~~~

For checking a number is less than `32`, we need to check that none of the upper 3 bits are set. For this, we can bitwise and with the bitwise compelement of 31, which has all the lower 5 bits set.

~~~
pub fn lt_32(x: u8) -> u8 {
    not(is_non_zero(!0x1f & x))
}
~~~

Finally, we need a way to go from `0x1` and `0x0` to `true` and `false`. Since we only have

This let me put the whole thing together:


[stackoverflow]: http://stackoverflow.com/questions/3912112/check-if-a-number-is-non-zero-using-bitwise-operators-in-c
-->
