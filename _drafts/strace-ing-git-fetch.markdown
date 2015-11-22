---
layout: post
title: How git pull over SSH works
---

Yesterday I was curious about how `git push` works over SSH. I'm getting more
used to using `strace` to figure this kind of thing out, so I gave it a shot.
If I `strace` pushing to this site's repository, this shows up:

~~~
[pid 15943] execve("/usr/bin/ssh", ["ssh", "git@github.com", "git-receive-pack 'kamalmarhubi/w"...], [/* 51 vars */]) = 0
~~~

So `git push` eventually calls `ssh git@github.com git-receive-pack
<repo-path>`. Trying this out at my terminal gives me this:

~~~
$ ssh git@github.com git-receive-pack kamalmarhubi/website
00bb29793c39c8e4bfec627d60938c4ed2086cc60bb1 refs/heads/gh-pagesreport-status delete-refs side-band-64k quiet atomic ofs-delta agent=git/2:2.4.8~upload-pack-wrapper-script-1211-gc27b061
003f04bfcb3e238e5660ae9e71a6ce99f472211fe85f refs/heads/master
0000
~~~

with the terminal waiting for my input. SSH is used to handle authentication
and remote connection, and then it runs a command at the other end to handle
the data exchange. These lines are the start of that exchange.

A tiny bit of looking around the internet told me that the protocol is made up
of lines prefixed by their length as 4 hex digits. Then it looks like a commit
SHA-1 and a ref. The sender terminates with `0000`.

There are a couple of lines here, one for each branch in the repository. The
first line additionally has a bunch of stuff at the end that looks like a
description of what the sending program is and some features it supports.

While I was looking into this, I used `xsel` to copy the output to paste into
an editor. This was really confusing, because all that got pasted was the first
line without all the metadata!

~~~
00bb29793c39c8e4bfec627d60938c4ed2086cc60bb1 refs/heads/gh-pages
~~~

Looking at the entire output through `hexdump -C`, it turns out that there's a
null byte after `refs/heads/gh-pages`, and then a newline at the end:

~~~
00000000  30 30 62 62 32 39 37 39  33 63 33 39 63 38 65 34  |00bb29793c39c8e4|
00000010  62 66 65 63 36 32 37 64  36 30 39 33 38 63 34 65  |bfec627d60938c4e|
00000020  64 32 30 38 36 63 63 36  30 62 62 31 20 72 65 66  |d2086cc60bb1 ref|
00000030  73 2f 68 65 61 64 73 2f  67 68 2d 70 61 67 65 73  |s/heads/gh-pages|
00000040 *00*72 65 70 6f 72 74 2d  73 74 61 74 75 73 20 64  |.report-status d|
00000050  65 6c 65 74 65 2d 72 65  66 73 20 73 69 64 65 2d  |elete-refs side-|
00000060  62 61 6e 64 2d 36 34 6b  20 71 75 69 65 74 20 61  |band-64k quiet a|
00000070  74 6f 6d 69 63 20 6f 66  73 2d 64 65 6c 74 61 20  |tomic ofs-delta |
00000080  61 67 65 6e 74 3d 67 69  74 2f 32 3a 32 2e 34 2e  |agent=git/2:2.4.|
00000090  38 7e 75 70 6c 6f 61 64  2d 70 61 63 6b 2d 77 72  |8~upload-pack-wr|
000000a0  61 70 70 65 72 2d 73 63  72 69 70 74 2d 31 32 31  |apper-script-121|
000000b0  31 2d 67 63 32 37 62 30  36 31*0a*30 30 33 66 37  |1-gc27b061.003f7|
000000c0  39 32 66 34 39 36 65 37  35 33 64 62 39 33 33 30  |92f496e753db9330|
000000d0  66 30 61 34 65 38 32 39  30 62 38 61 36 63 62 61  |f0a4e8290b8a6cba|
000000e0  38 61 62 36 64 61 62 20  72 65 66 73 2f 68 65 61  |8ab6dab refs/hea|
000000f0  64 73 2f 6d 61 73 74 65  72 0a 30 30 30 30        |ds/master.0000|
000000fe
~~~

Without doing any research, here's what I think happened. The git folks defined
the fairly simple length-prefixed, newline-separated protocol. Then at some
point they wanted to add some metadata to the protocol without breaking
compatibility with older versions of git. They came up with a nifty hack that
exploits C's null-terminated strings: add the metadata after a null byte but
before the newline. This way, reading up to a newline will get all the
metadata. The metadata-processing code knows to look past the null byte, but
the existing protocol code would see only the part up before it, presumably
letting it worked unchanged!

And when I copied it using `xsel`, the stuff past the null byte got skipped.
Cute hack, and mystery solved!
