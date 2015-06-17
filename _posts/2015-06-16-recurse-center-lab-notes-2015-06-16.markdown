---
title: Recurse Center lab notes 2015-06-16
date: 2015-06-16
permalink: /blog/2015/06/16/recurse-center-lab-notes
---

Today I continued working on my `lsaddr` utility for listing IP addresses.
Here's a quick update:

- there is now a [repository on GitHub][lsaddr-repo]
- I have a [man page][man-page] that I'm mostly pleased with; here's the
  synopsis:

  ```
  lsaddr [ -46 ] [--include-loopback] [--include-ipv6-link-local] [ interface ... ]
  ```

- I am checking that the supplied interfaces exist and reporting errors with
  [`error`][error], which I discovered was the done thing by using `ltrace` on
  `ls does-not-exist` :-)
- I tried to use [`lfind`][lfind] to determine if interfaces were listed as
  arguments, but wow what a mess that was[^lfind-mess]; I scrapped that to come
  back to it another day

[lsaddr-repo]: https://github.com/kamalmarhubi/lsaddr
[man-page]: https://github.com/kamalmarhubi/lsaddr/blob/master/lsaddr.adoc
[error]: http://linux.die.net/man/3/error
[lfind]: http://linux.die.net/man/3/lfind

In other news, I read [this exciting post][ebpf-post] by Brendan Gregg on eBPF
and tracing in the kernel. Apparently it will soon be possible to gather
latency data from inside the kernel. I haven't figured out if this could be
interesting for IPC benchmarks.

[ebpf-post]: http://www.brendangregg.com/blog/2015-05-15/ebpf-one-small-step.html

<br />

---

[^lfind-mess]:
    the closest C comes to having generics / parametric polymorphism is
    `void *`. With things like [`malloc`][malloc], this is mostly fine. With algorithms
    like `lfind` and friends... well you're going to have a bad time. Just look
    a this signature:

    ```
    void *lfind(const void *key, const void *base, size_t *nmemb,
                size_t size, int(*compar)(const void *, const void *))
    ```

    I find [`std::find`][std-find] unfriendly, but this is a whole other level.


[std-find]: http://en.cppreference.com/w/cpp/algorithm/find
[malloc]: http://linux.die.net/man/3/malloc
