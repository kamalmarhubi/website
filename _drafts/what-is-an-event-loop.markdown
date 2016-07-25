---
title: Event-based IO
---

One of the things I've been doing over the last couple of weeks, is getting my head around event-based IO, and how you actually do it. I've had a high level idea for a while, but there's a bunch of details that were missing.

This post is a brain dump of some things I've learned, focused around epoll on Linux, because that's what my computer runs.


## Background

Assume we are writing some kind of socket server, say a web server that sends a simple hello response back for every request. The basic flow is that we set up a socket to listen for new connections. We then accept new connections which gets us a new socket connected to the client. 

~~~
RESPONSE = "200 OK\r\n\r\nHello, world!"
listener = setup_listener_socket()
while True:
    client_sock = listener.accpet()
    req = client_sock.read(1)  # just wait for *something* and send response
    client_sock.write(RESPONSE)
~~~


## Blocking IO and why it's not the best

By default, sockets are opened in *blocking mode*. This means that if you try to read from one that doesn't have data available, the OS will put your thread to sleep, and give the CPU over to another thread. This is called a *context switch*. When data comes in, the CPU will schedule your thread back onto the CPU and then the `read` call will return with the new data.

For our hello world servers this might happen if the client takes a while to actually send its request after the TCP connection is established. The first attempt to read from the socket will block until the request is actually received.

In this model, each thread can only handle a single request at a time. To handle multiple requests concurrently, we'll need multiple threads. Each takes up memory for its stack, as well as some book-keeping space in the kernel.

Even worse, all those context switches can take up a lot of time. Here are a few things that happen during a context switch:

- the execution context (registers) of the currently running thread gets saved to memory
- the execution context for the newly scheduled thread gets loaded from memory and put into the registers
- the address space switches to the one for the newly scheduled thread, which means that translation-lookaside buffer (TLB) needs to get flushed

And on top of that, the CPU caches are most likely not full of data or instructions useful to the newly-scheduled thread. This means that execution will be slower for a bit as the CPU will have to wait on main memory more often.


## Non-blocking IO

All these threads and context switches are resources being spent on stuff that isn't our core task of greeting as many users as we can as quickly as possible. So instead of having the kernel do all that context switching when, we can set the socket to non-blocking mode by setting the `SO_NONBLOCK` socket option. In this mode, reads and writes return immediately with the `EWOULDBLOCK` error if they couldn't complete right now.

Now if we had a new connection where the request was taking some time to come in, instead of a context switch, we'd get a specific failure from our `read` call. We can even set the listening socket to be non-blocking. That way, `accept` calls won't block until a new connection comes in, they'll just return `EWOULDBLOCK` instead.

This is well and good, but now




# Non-blocking IO and


Over the years, I've heard all kinds of keywords about

~~~
while True:
    events = collect_events()
    for e in events:
        handle_event(e)
~~~



Some time a few months ago, I had a video chat with Dan Callahan where we
read through 

Event driven, evented, non-blocking.












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



