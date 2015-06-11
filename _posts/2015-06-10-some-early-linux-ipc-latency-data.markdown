---
title: Some early Linux IPC latency data
date: 2015-06-10
---

I've [added][af-unix-commit] [benchmarks][af-inet-commit] for UNIX domain
sockets and TCP sockets over the loopback interface. UNIX domain sockets were
super easy to implement thanks to the handy `[socketpair]` function. It was not
really any different from pipes. The difference is that since sockets are full
duplex, you only need to create one pair.  If the processes were unrelated, or
if I wanted to be able to accept multiple connections, it would be much more
like TCP sockets—ie, a pain!

[af-unix-commit]: https://github.com/kamalmarhubi/linux-ipc-benchmarks/commit/e06c93b54b4d13e1f78c64add9ac8a5cdf19b9ff
[af-inet-commit]: https://github.com/kamalmarhubi/linux-ipc-benchmarks/commit/8f9094522465db54003f08da4d5b797e2944f47e
[socketpair]: http://man7.org/linux/man-pages/man2/socketpair.2.html

I say a pain because, in doing this, I ‘found out’ that, despite having written
a non-zero number of server applications, I've never done socket programming
before. This wasn't exactly a surprise, but it was definitely interesting to
realise how little I knew about how to go about it. Luckily, man pages! (And
[Advanced Programming in the UNIX Environment][adv-unix].)

[adv-unix]: http://www.apuebook.com/index.html

Here's the quick tl;dr for TCP over IPv4::

- to listen for incoming connections:
  1. create a socket with `socket(AF_INET, SOCK_STREAM, 0 /* default protocol */)`.[^default-proto]
  2. bind it to a port with `bind(sockfd, addr, addrlen)` where `addr` is a
     struct that specifies the address to bind to. For `AF_INET`, this means
     the IP and port. In my case, I used `INETADDR_LOOPBACK` and `0` to listen
     on some available port on `127.0.0.1`.[^htonl]
  3. start listening on the socket with `listen(sockfd, 1 /* backlog */)`. I
     used a `backlog` of 1 because I only expect a single incoming connection.
  4. finally, call `accept(sockfd, NULL /* addr */, NULL /* addrlen */)` to
     block until a connection comes in, which returns a new file descriptor to
     talk to the connecting process. I pass in `NULL` for the `addr` because I
     don't care who's talking to me!
- to connect to another process that's listening:
  1. create a socket with `socket(AF_INET, SOCK_STREAM, 0 /* default protocol */)`.
  2. connect to the remote process with `connect(sockfd, addr, addrlen)`. The
     `addr` specifies the address to connect to; again for `AF_INET` this means
     the IP and port.



[socket]: http://man7.org/linux/man-pages/man2/socket.2.html
[bind]: http://man7.org/linux/man-pages/man2/bind.2.html
[listen]: http://man7.org/linux/man-pages/man2/listen.2.html
[accept]: http://man7.org/linux/man-pages/man2/accept.2.html
[connect]: http://man7.org/linux/man-pages/man2/connect.2.html

This brings me up to having programs to test latency for four IPC mechanisms:
- pipes
- eventfd
- UNIX domain sockets
- TCP sockets over the loopback interface

Here is some early latency data from my machine, with emphasis on the tail latencies:

| | 50 | 75 | 90 | 99 | 99.9 | 99.99 | 99.999 |
|:-------+-------:|:-------:|------:|---:|--:|--:|--:|--:|
| pipes |  4255 | 4960 | 5208 | 5352 | 7814 | 16214 | 31290 |
| eventfd  | 4353 | 4443 | 4760 | 5053 | 9445 | 14573 | 68528 |
| af_unix  | 1439 | 1621 | 1655 | 1898 | 2681 | 11512 | 54714 |
| af_inet_loopback  | 7287 | 7412 | 7857 | 8573 | 17412 | 20515 | 37019 |

Units are nanoseconds. Time is measured using `clock_gettime` with
`CLOCK_MONOTONIC`. The quantiles are for a million measurements; in all cases,
the binary was run with flags `--warmup-iters=10000 --iters=1 --repeat=1000000`
(see below).

For me, the biggest surprise was how much faster UNIX domain sockets were than
anything else, and in particular, how much faster they are than eventfd. Or
that they are faster at all. The `read` call in each case blocks until a
corresponding `write`. I would have thought eventfd had the minimal amount of
extra work beyond that, since all it does is read and modify a `uint64_t`. In
fairness, each of the other programs are writing a single byte at present, but
I doubt the difference will be so drastic.

Another fun thing is to see difference in `latency when pinning the two
processes to specific CPUs. My machine has a dual core processor, where each
processor has 2 hardware threads. Here's a quick look at latencies for pipes
with different CPU affinities:

|Percentile | 50 | 75 | 90 | 99 | 99.9 | 99.99 | 99.999 |
|:---:|---:|---:|---:|---:|---:|---:|---:|
| default | 4255 | 4960 | 5208 | 5352 | 7814 | 16214 | 31290 |
| same CPU | 2386 | 2402 | 2564 | 3134 | 12255 | 15126 | 28225|
| same core | 4232 | 4270 | 4395 | 4788 | 14408 | 17101 | 39052 |
| different core | 5043 | 5101 | 5170 | 5772 | 11894 | 38726 | 398796 |
| ---- |

I was expecting a difference between different cores and not, since it requires
a trip through the L3 cache. I have no realy idea of what difference I was
expecting, but a microsecond could make sense if multiple locations needed to
be accessed. This stuff is beyond my ken, so I'm just guessing.

What I was _not_ expecting, was a dramatic difference between ‘same CPU’ and
‘same core’. The CPUs are hardware threads on a single core. I can't think of
any reason there would be such a difference. I do want to check that it's not
due to scheduling weirdness, so I'll probably boot up in single user mode at
some point to give it another go.

If you want to run these on your own system, clone the [repo] and run `make`.
There will be four binaries produced, one for each of the mechanisms.  They all
take the same command line flags:

~~~
  -c, --child-cpu=CPUID      CPU to run the child on; default is to let the
                             scheduler do as it will
  -i, -n, --iters=COUNT      number of iterations to measure; default: 100000
  -p, --parent-cpu=CPUID     CPU to run the parent on; default is to let the
                             scheduler do as it will
  -r, --repeat=COUNT         number of times to repeat measurement; default: 1
  -w, --warmup-iters=COUNT   number of iterations before measurement; default:
                             1000
  -?, --help                 Give this help list
      --usage                Give a short usage message
~~~

[repo]: https://github.com/kamalmarhubi/linux-ipc-benchmarks



<br />

---


[^default-proto]:
    The default protocol for `SOCK_STREAM` for the `AF_INET`
    socket family is TCP.

[^htonl]:
    A fun little thing to be aware of is that the `addr` must contain the IP
    address in network byte order. This necessitates converting the IP address
    and port using `htonl` and `htons`, respectively, to convert the IP from
    _h_ost _to_ _n_etwork byte order (the `l` stands for `long`, which in this
    case means a `uint32_t` because `long`s used to be shorter; the `s` stands
    for `short` which have stayed short at 16 bits long).
