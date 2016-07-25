---
title: Some things I learned about libdwarf
date: 2016-07-25T18:04:02-04:00
---

In the last few days, I got drafted / nerd-sniped into helping Julia Evans on [her Ruby stacktrace spying program][julia-post]. Specifically, one of the next big things is to read type information out of the Ruby binary so that it's not hardcoded to work for just one Ruby version.

Going in, I didn't know much about DWARF, its data model, or how to access it programmatically. This post is a minidump of things I learned about libdwarf, including reasons I'm probably not going to use it.

[julia-post]: http://jvns.ca/blog/2016/06/12/a-weird-system-call-process-vm-readv/


## What DWARF is, the very very short and incomplete version

From the [DWARF standard's homepage][dwarfstd]:

> DWARF is a debugging file format used by many compilers and debuggers to support source level debugging.

[dwarfstd]: http://dwarfstd.org/

Here are some of the things stored in DWARF data:

- symbol names, like functions, parameters, variables
- a way to go from source file lines to instruction addresses and vice versa, for setting breakpoints and stepping through functions
- a way to find out where data for a variable is at a given point in the program
- information about the types in a program, including their size and offsets of their fields

I'm sure there's lots of other stuff in there too. For the task at hand, I'm mostly interested in the last of these. The Ruby stacktrace program currently works by using some struct definitions from header files from Ruby itself. This ties it to a specific Ruby version. We want to make it work on any Ruby version by looking up the struct definitions in the DWARF data.


## libdwarf

Linked off the dwarfstd.org site is libdwarf, a C library for reading DWARF.  This is great, because the DWARF format looks pretty complicated, and I definitely don't want to write a reader from scratch! There's an [example program][simplereader] included with libdwarf to show you how to read debug info.

[simplereader]: https://sourceforge.net/p/libdwarf/code/ci/master/tree/dwarfexample/simplereader.c


## Things I found weird about libdwarf

I started putting together a Rust wrapper for libdwarf, but ended up finding it a little too weird.

NB: I don't do much C programming beyond working with Linux system libraries, so these might not actually be weird practices for C libraries. But they were definitely weird for me!


### Memory management tied to a specific libdwarf session

Some libdwarf functions allocate memory for the data they return. The library provides a special `dwarf_dealloc` function to free them. So far so good. But the deallocation also takes a handle to the `Dwarf_Debug` session that the memory was allocated for. This isn't too bad: a Rust wrapped object could just hold a handle to the `Dwarf_Debug` session so that `dwarf_dealloc` can be called in the destructor. There is a small niggle in that the library will deallocate all these objects for you if you call close the `Dwarf_Debug` with `dwarf_finish`. This can all be handled in a reasonable way from the Rust side by tying the lifetimes of things to the `Dwarf_Debug` instance they came from.


### Function names increment with library changes

Here are four function names from libdwarf:

- `dwarf_next_cu_header()`
- `dwarf_next_cu_header_b()`
- `dwarf_next_cu_header_c()`
- `dwarf_next_cu_header_d()`

Each one except for `dwarf_next_cu_header_d()` includes a note like:

> It operates exactly like `dwarf_next_cu_header_d()` but is missing the `header_type` field. This is kept for compatibility. All code using this should be changed to use `dwarf_next_cu_header_d()`.

I don't know how to handle compatibility and API evolution in C, but this is definitely the first time I've seen something like this! It does mean that code compiled against older versions will continue to work, which is good. But it strikes me as odd overall.


### Error handling

Most libdwarf functions take a `Dwarf_Error*` as their last argument. If you pass a non-null pointer, then error data will stored there. If a null pointer is passed, then the library calls `abort()` on your behalf instead. Except if you passed an error handler callback when creating the `Dwarf_Debug` instance: errors will get passed to your callback. It's unclear to me if you are meant the `dwarf_dealloc` the error your callback gets passed in that case. If the callback returns (rather than exiting the program or throwing the error as an exception), then I think the whole call looks like nothing went wrong.

Oh and if you do pass a non-null `Dwarf_Error*`, you are in charge of freeing the error with `dwarf_dealloc`, passing the `Dwarf_Debug` instance, as mentioned above. Except if that error happened in the call to `dwarf_init`, in which case there is no valid `Dwarf_Debug` instance; in that case, you just free it with `free()`.


### Sometimes stateful iteration

There are a bunch of functions for iterating through bits and pieces of DWARF data. For some, there is a `next` function that takes the current item and gives the next one. Iteration can be started from the beginning by passing in `NULL` as the current item. But in one case, the iteration state is stored in the `Dwarf_Debug` instance. You must repeatedly call the same function until it returns NULL: here's no other way to restart the iteration.


## libdw

None of those things were too weird in themselves, but taken together I was wondering about alternatives. The alternative I'm looking at right now is libdw from elfutils. At a short glance, it has a more straightforward API. There's also a potentially useful higher level "frontend library" called libdwfl. I'll be taking a closer look this week and putting together a rough binding to see how that goes!
