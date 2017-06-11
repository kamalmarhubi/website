---
title: What's in a coredump?
---

As I mentioned in my [last post][dwarf-post], I've recently been working
on [ruby-stacktrace], a program to do profiling on running Ruby processes
without interrupting them.

Right now I want to be able to benchmark the code that peeks inside the
running Ruby process' memory. We want that to be pretty quick, since it'll
be happening many times a second. I guessed yesterday that this should be
possible with a coredump of the running Ruby process, and today that's
what I've been trying to do.

This post is a quick dump of a few things I've learned in the process.


## A coredump is an ELF file!

This was my first bit of surprise. I'd never thought about what a core
file was. I figured it was a big fat binary blob of "stuff". But I also
knew it wasn't the entire address space of the process because, well that
would be really really really really really big on a 64-bit machine. This
means that it has to contain blobs for segments of memory that are dumped,
and the addresses they belong at.

This is exactly what ELF files store! They have a bunch of "sections",
each with a size and an address it should be placed at in the loaded
program.

ELF files consist of a bunch of sections. Some possibly familiar examples
 are `.text` (program code), `.data` (read-only data), `.bss`
(read-write, zero-initialized static data), and `.debug_info` (DWARF debug
info).

At a high level, a section consists of some metadata that describes where
in memory it should be placed, what permissions the memory should have and
so on. A section can have an offset that says where their data is located
in the ELF file, and some don't. For example, the `.bss` section in
a binary 




## What's *not* in a coredump?

A few things are 


## Some things I don't know yet

I mentioned that the non-writeable sections of the binary aren't included
in the coredump. This makes sense, since they can be found in the binary
itself. But I'm confused about how gdb knows where to put them in the
address space. I could only find the base address of the binary in the
coredump in one location: the memory associated with lib



[dwarf-post]: TODO
