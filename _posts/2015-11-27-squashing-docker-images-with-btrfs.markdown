---
title: Squashing Docker images with Btrfs
date: 2015-11-27T14:44:03-05:00
---

Here's a small hack for squashing a Docker image to a single layer if you
happen to run with `/var/lib` on a Btrfs filesystem. The output will be a tar
file you can use with [`ADD`][docker-add] in a `Dockerfile` to create a flat
base image.

This is an alternative to [`docker create`][docker-create]-ing a container and
running [`docker export`][docker-export] on it to get the filesystem contents.
It saves you having to [`docker rm`][docker-rm] the container afterwards.


[docker-create]: http://docs.docker.com/engine/reference/commandline/create/
[docker-export]: http://docs.docker.com/engine/reference/commandline/export/
[docker-rm]: http://docs.docker.com/engine/reference/commandline/rm/

[docker-add]: http://docs.docker.com/engine/reference/builder/#add

First, get the ID of the image you want to squash:


~~~
$ docker images rethink
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
rethinkdb           latest              ffb24b60063b        6 days ago          181.8 MB
rethinkdb           2.0.4               83d2da4505dc        2 weeks ago         195.8 MB
~~~

I'll go with the latest version, `ffb24b60063b`. This is a prefix of the full
image ID. The image will be a Btrfs snapshot in
`/var/lib/docker/Btrfs/subvolumes`:

~~~
$ sudo -i
# cd /var/lib/docker/btrfs/subvolumes
# ls -d ffb24b60063b*
ffb24b60063ba7e26ebf2f888deb5af0c8966dcbb612353fcd0c5d52a0a1d234
~~~

The [Btrfs storage driver's documentation][docker-btrfs-driver] goes into a
fair bit of detail on how the driver uses Btrfs subvolumes and snapshots for
images and containers.

[docker-btrfs-driver]: http://docs.docker.com/engine/userguide/storagedriver/btrfs-driver/

Here's an exerpt from [`dockviz`][dockviz] output for that image that shows the
various layers:

~~~
├─1565e86129b8 Virtual Size: 125.1 MB
│ └─a604b236bcde Virtual Size: 125.1 MB Tags: debian:latest
│   └─3012d15ee771 Virtual Size: 125.1 MB
│     └─67ab878cc688 Virtual Size: 125.1 MB
│       └─d4d9554f3430 Virtual Size: 125.1 MB
│         └─d4bdd500e4ec Virtual Size: 125.1 MB
│           └─9c56aa19b706 Virtual Size: 181.8 MB
│             └─9102e5038f43 Virtual Size: 181.8 MB
│               └─c5dff5ddf6a8 Virtual Size: 181.8 MB
│                 └─769806cae856 Virtual Size: 181.8 MB
│                   └─ffb24b60063b Virtual Size: 181.8 MB Tags: rethinkdb:latest
~~~

[dockviz]: https://github.com/justone/dockviz

Then it's just a matter of:

~~~
$ sudo -i
# cd /var/lib/docker/btrfs/subvolumes/ffb24b60063ba7e26ebf2f888deb5af0c8966dcbb612353fcd0c5d52a0a1d234
# tar c . > /tmp/rethink-squashed.tar
~~~

Tab completion should pick up the full directory name from the short image ID,
so you don't have to copy the complete ID.

That's pretty much it. You can get a flat base image with a `Dockerfile`
with these contents:

~~~
FROM scratch
ADD rethink-squashed.tar /
~~~
