---
layout: post
title: What even is a kubelet?
---

*This post is the first in a series exploring the concepts and components that
underly [Kubernetes]. It talks about the kubelet component, and the pod
concept.*

[Kubernetes] is Google's open source, container-focused cluster management
thing.  I see it as their attempt to tell everyone how they think containers
and clusters fit together. The Kubernetes documentation is quite good, but it's
divided up in a way that makes it great as a reference. I want to understand
both the concepts Kubernetes introduces, and the components that make up a
Kubernetes cluster, and I want to learn by doing. I'm planning to build up a
cluster from scratch, documenting the moving parts and concepts as I go.

[kubernetes]: http://kubernetes.io/
[getting-started]: http://kubernetes.io/v1.0/docs/getting-started-guides/README.html

I'll start of with a look at the [kubelet], which is the lowest level component
in Kubernetes. It's responsible for what's running on an individual machine.
You can think of it as a process watcher like [supervisord], but focused on
running containers. It has one job: given a set of containers to run, make sure
they are all running.

[kubelet]: http://kubernetes.io/v1.0/docs/admin/kubelet.html
[supervisord]: http://supervisord.org/

The unit of execution that Kubernetes works with is the [*pod*]. A pod is a
collection of containers that share some resources: they have a single IP, and
can share volumes. For example, a web server pod could have a container for the
server itself, and a container that tails the logs and ships them off to your
logging or metrics infrastructure.

Pods are defined by a JSON or YAML file called a pod manifest. A simple one
looks like this:

[pod]: http://kubernetes.io/v1.0/docs/user-guide/pods.html

~~~ yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
~~~

The container's `image` is a Docker image name. The `containerPort` exposes
that port from the nginx container so we can connect to the nginx server at the
pod's IP.

There are a few ways to tell the kubelet what to run. The simplest is to put a
pod manifest in a directory it watches. Every 20 seconds, it checks for changes
in the directory, and adjusts what it's running based on what it finds. This
means both launching pods that are added, as well as killing ones that are
removed.

The kubelet is such a low level component with such limited responsibilities
that we can actually use it independently of Kubernetes. The kubelet supports
[Docker] and [rkt] as continer runtimes. The default is Docker, so that's what
we'll use in the examples here. You'll need a machine with Docker installed and
running to try this out.

[docker]: https://github.com/docker/docker
[rkt]: https://github.com/coreos/rkt

First off, let's get the kubelet binary from Google.

~~~
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.0.3/bin/linux/amd64/kubelet
$ chmod +x kubelet
~~~

If you run `./kubelet --help`, you'll get an overwhelming list of options. For
what we're about to do, we only need one of them though: the `--config` option.
This is the directory that the kubelet will watch for pod manifests to run. We'll
create a directory for this, and then start the kubelet.

~~~
$ mkdir manifests
$ sudo ./kubelet --config=$PWD/manifests
~~~

Now let's stick the example nginx pod manifest from above in an `nginx.yaml`
file, and then drop it in the `manifests` directory. After a short wait, the
kubelet will notice the file and fire up nginx.

We can check the list of running containers with `docker ps`:

~~~
$ docker ps
CONTAINER ID        IMAGE                                  COMMAND                CREATED             STATUS              PORTS               NAMES
64c55f0d9994        nginx:latest                           "nginx -g 'daemon of   5 minutes ago       Up 5 minutes                            k8s_nginx.d7d3eb2f_nginx-kx_default_86ad95a1282b0607619533d662696c04_e7b1f43d   
697f93073509        gcr.io/google_containers/pause:0.8.0   "/pause"               5 minutes ago       Up 5 minutes                            k8s_POD.ef28e851_nginx-kx_default_86ad95a1282b0607619533d662696c04_d0666e90     
~~~

There are two containers running. One is nginx, the other is the pod
infrastructure container. The infrastructure container is a place to put all
the pod resources that are shared across containers in the pod shared resources
are placed, eg, the IP and any volumes.[^pause] We can poke around with `docker
inspect` to see how they're configured and hooked up to each other:

[^pause]:
    The `pause` command that the infrastructure container runs is a 129 byte
    ELF binary that just calls the [`pause` system call][man-2-pause], and
    exits when a signal is received. This keeps the infrastructure container
    around until the kubelet brings it down. It's pretty cool, check the
    [source][pause-source]!

[man-2-pause]: http://man7.org/linux/man-pages/man2/pause.2.html
[pause-source]: https://github.com/kubernetes/kubernetes/blob/88317efb42db763b9fb97cd1d9ac1465e62009d0/third_party/pause/pause.asm

~~~
$ docker inspect --format '{{ .NetworkSettings.IPAddress }}' 64c55f0d9994

$ docker inspect --format '{{ .NetworkSettings.IPAddress }}' 697f93073509
172.17.0.8
~~~

The nginx container has no IP, but the infrastructure container does. Taking a
closer look at at the nginx container, we can see its `NetworkMode` is set to
use the infrastructure container's network:

~~~
$ docker inspect --format '{{ .HostConfig.NetworkMode }}' 64c55f0d9994
container:697f93073509513d47f80b9ce76f8f4217c3503182c1b212c4f8856d75600bcd
~~~

Since we exposed port 80 from the nginx container with `containerPort`,
we can connect to the nginx server at port 80 at the pod's IP:

~~~
$ curl --stderr /dev/null http://172.17.0.8 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
~~~

It really is running!

The kubelet also has an internal HTTP server. We won't go into it in detail
in this post, except to say that it serves a read-only view at port
10255.  There's a health check endpoint at `/healthz`:

~~~
$ curl http://localhost:10255/healthz
ok
~~~

There are also a few status endpoints. For example, you can get a list
of running pods at `/pods`:

~~~
$ curl --stderr /dev/null http://localhost:10255/pods | jq . | head
{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {},
  "items": [
    {
      "metadata": {
        "name": "nginx-kx",
        "namespace": "default",
        "selfLink": "/api/v1/pods/namespaces/nginx-kx/default",
~~~

You can also get specs of the machine the kubelet is running on at
`/spec/`:

~~~
$ curl --stderr /dev/null  http://localhost:10255/spec/ | jq . | head
{
  "num_cores": 4,
  "cpu_frequency_khz": 2700000,
  "memory_capacity": 4051689472,
  "machine_id": "9eacc5220f4b41e0a22972d8a47ccbe1",
  "system_uuid": "818B908B-D053-CB11-BC8B-EEA826EBA090",
  "boot_id": "a95a337d-6b54-4359-9a02-d50fb7377dd1",
  "filesystems": [
    {
      "device": "/dev/mapper/kx--vg-root",
~~~

Finally, we can clean up after ourselves. Just deleting the nginx pod
manifest will result in the kubelet stopping the containers.

~~~
$ rm $PWD/manifests/nginx.yaml
$ sleep 20  # wait for the kublet to spot the removed manifest
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
$ curl --stderr /dev/null http://localhost:10255/pods
{"kind":"PodList","apiVersion":"v1","metadata":{},"items":null}
~~~

All gone!

We've seen that while the kubelet is a part of Kubernetes, at heart it's a
container-oriented process watcher. You can use it in isolation to manage
containers running on a single host. In fact, the Kubernetes [getting started
guides for Docker][k8s-getting-started-docker] run the kubelet under Docker and
use the kubelet to manage the Kubernetes master components. In a later post,
we'll do something similar!

[k8s-getting-started-docker]: http://kubernetes.io/v1.0/docs/getting-started-guides/docker.html#step-two-run-the-master

<br>

---

