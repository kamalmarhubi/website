---
title: A list of Linux IPC mechanisms
date: 2015-06-01
---

Here is a list of Linux inter-process communication mechanisms that I know of:

  - pipes
  - FIFOs
  - POSIX IPC (semaphores, message queues, shared memory)
  - `memfd`
  - UNIX domain sockets
  - TCP/UDP sockets on loopback interface
  - `eventfd`
  - `splice` and friends
  - signals
  - the filesystem, including `mmap`'ed files
  - `process_vm_readv` and `process_vm_writev`
  - `ptrace` with `PTRACE_PEEK{TEXT,DATA,USER}` and `PTRACE_POKE{TEXT,DATA,USER}`

I'll link or explain them a bit better soon, but for now I just want to collect them. If you know of any I'm missing, please [tweet][twitter] or [email] me!

[twitter]: https://twitter.com/kamalmarhubi
[email]: mailto:kamal@marhubi.com
