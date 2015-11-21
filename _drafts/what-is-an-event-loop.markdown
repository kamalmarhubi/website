---
title: Asynchronous IO: what is an event loop, anyway?
---

asyncrhonous programming
event driven
asyncio

what is an event loop, anyway?

I knew lots of WORDS:
- event
- libevent
- libuv
- asyncio, twisted
- epoll, kqueue

"readiness"
c10k

Looked at asyncio from python 3.4+

Pure Python
Easy to read
Really simple!

Socket programming ordinarily involves blocking reads and writes. If you call
read on a socket, the OS puts you to sleep until there's something there to read. Same for writes.

A single thread handles at most one connection at a time.

Ways to deal with this: multiple threads, so that each connection gets its own thread.

Alternatively, event driven stuffs.

One thread multiple connections.

Did someone tell us to do something?
Is it time to do something someone told us do earlier?
Does the OS have any new things for us?



