---
title: Recurse Center lab notes 2015-06-23
---

Having mentioned that, right now could be a good time to—in the words of a
former coworker—take a step back[^step-back]. Why am I even interested in IPC
performance, and latency in particular?

[^step-back]:
    This was usually to make sure

If you're unfamiliar with it, Cap'n Proto is a data serialization protocol,
similar in purpose to Google's [Protocol Buffers][protobuf], or [Apache
Thrift][thrift]. The basic idea is you want to send data from one place to
another, or store it to disk for reading later. You could invent your own file
format every time, but then you'd have to implement it in every language that
might need to read or write the data. You could use something like JSON, but it
would require you to specially encode binary data, and the repeated keys can
take up a lot of space.

[protbuf]:
[thrift]: https://thrift.apache.org/

Protocol Buffers, Thrift and Cap'n Proto work. First you define the structure
of your data—its schema.  Then you use a schema compiler to generate code for
your programming langauge of choice. This generated code knows how to read and
write data with that schema to a common format. Data stored in a file by a
program in Python can be read by a program in JavaScript and so on.

Where Cap'n Proto differs from Pr

[email]: https://groups.google.com/d/msg/capnproto/kLQOsxjkjxM/ngbJYer52OwJ

First of all, what even is a memory mapped file? When we work with files, we
typically `open` them, and do some `read`s and `write`s. The open file in our
program has a cursor which points to where we're at in the file. When we `read`
and `write`, the cursor advances. We can also usually zip the cursor
around—this is called seeking.

If we're just reading through the file once, or just writing out a bunch of
output, this is great.
