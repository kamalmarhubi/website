---
layout: post
title: What even is a kubelet?
date: 2015-08-27T22:24:28-04:00
---

[Kubernetes] is Google's open source, container-focused cluster management
thing.  I see it as their attempt to tell everyone how they think containers
and clusters fit together. The Kubernetes documentation is quite good, but it's
divided up in a way that makes it great as a reference. I want to understand
both the concepts Kubernetes introduces, and the components that make up a
Kubernetes cluster, and I want to learn by doing. I'm planning to build up a
cluster from scratch, documenting the moving parts and concepts as I go.

[kubernetes]: http://kubernetes.io/
[getting-started]: http://kubernetes.io/v1.0/docs/getting-started-guides/README.html

I'll start of with a look at the *[kubelet]*, which is the lowest level component
in Kubernetes. It's responsible for what's running on an individual machine.
You can think of it as a process watcher like [supervisord], but focused on
running containers. It has one job: given a set of containers to run, make sure
they are all running.

[kubelet]: http://kubernetes.io/v1.0/docs/admin/kubelet.html
[supervisord]: http://supervisord.org/

# Kubelets run pods

The unit of execution that Kubernetes works with is the *[pod]*. A pod is a
collection of containers that share some resources: they have a single IP, and
can share volumes. For example, a web server pod could have a container for the
server itself, and a container that tails the logs and ships them off to your
logging or metrics infrastructure.

Pods are defined by a JSON or YAML file called a pod manifest. A simple one
with one container looks like this:

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
pod's IP. By default, the [entrypoint] defined in the image is what will run; in
the nginx image, that's the nginx server.

[entrypoint]: http://docs.docker.com/reference/builder/#entrypoint

Let's add a log truncator container to this pod. This will take care of the
nginx access log, truncating it every 10 seconds—who needs those anyway? To do
this, we'll need nginx to write its logs to a volume that can be shared to the
log truncator. We'll set this volume up as an `emptyDir` volume: it will start
off as an empty directory when the pod starts, and be cleaned up when the pod
exits, but will persist across restarts of the component containers.

Here's the updated pod manifest:

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
    volumeMounts:
    - mountPath: /var/log/nginx
      name: nginx-logs
  - name: log-truncator
    image: busybox
    command:
    - /bin/sh
    args: [-c, 'while true; do cat /dev/null > /logdir/access.log; sleep 10; done']
    volumeMounts:
    - mountPath: /logdir
      name: nginx-logs
  volumes:
  - name: nginx-logs
    emptyDir: {}
~~~

We've added an `emptyDir` volume named `nginx-logs`. nginx writes its logs at
/var/log/nginx, so we mount that volume at that location in the `nginx`
container. For the `log-truncator` container, we're using the [busybox] image.
It's a tiny Linux command line environment, which provides everything we need
for a robust log truncator. Inside that container, we've mounted the
`nginx-logs` volume at `/logdir`. We set its `command` and `args` up to run a
shell loop that truncates the log file every 10 seconds.

[busybox]: https://hub.docker.com/_/busybox/

Now we've got this paragon of production infrastructure configured, it's time
to run it!

# Running a pod

There are a few ways the kubelet finds pods to run:

- a directory it polls for new pod manifests to run
- a URL it polls and downloads pod manifests from
- from the Kubernetes API server

The first of these is definitely the simplest: to run a pod, we just put it in
the watched directory. Every 20 seconds, the kubelet checks for changes in the
directory, and adjusts what it's running based on what it finds. This means
both launching pods that are added, as well as killing ones that are removed.

The kubelet is such a low level component with such limited responsibilities
that we can actually use it independently of Kubernetes—all we have to do is
not tell it about a Kubernetes API server. The kubelet supports [Docker] and
[rkt] as continer runtimes. The default is Docker, and that's what we'll use in
the examples here. You'll need a machine with Docker installed and running to
try this out.

[docker]: https://github.com/docker/docker
[rkt]: https://github.com/coreos/rkt

First off, let's get the kubelet binary from Google.

~~~
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.0.3/bin/linux/amd64/kubelet
$ chmod +x kubelet
~~~

If you run `./kubelet --help`, you'll get an overwhelming list of options. For
what we're about to do, we only need one of them though: the `--config` option.
This is the directory that the kubelet will watch for pod manifests to run.
We'll create a directory for this, and then start the kubelet. You might need
to run it under `sudo` so that it can talk to the docker daemon.

~~~
$ mkdir manifests
$ ./kubelet --config=$PWD/manifests
~~~

Now let's stick the example nginx pod manifest from above in an `nginx.yaml`
file, and then drop it in the `manifests` directory. After a short wait, the
kubelet will notice the file and fire up the pod.

We can check the list of running containers with `docker ps`:

~~~
$ docker ps
CONTAINER ID        IMAGE                                  COMMAND                CREATED             STATUS              PORTS               NAMES
f1a27680e401        busybox:latest                         "/bin/sh -c 'while t   6 seconds ago       Up 5 seconds                            k8s_log-truncator.72cfff7a_nginx-kx_default_419bc51e985b6bb5e53ea305e2c1e737_401a4c94   
c5e357fc981a        nginx:latest                           "nginx -g 'daemon of   6 seconds ago       Up 6 seconds                            k8s_nginx.515d0778_nginx-kx_default_419bc51e985b6bb5e53ea305e2c1e737_cd02602b           
b2692643c372        gcr.io/google_containers/pause:0.8.0   "/pause"               6 seconds ago       Up 6 seconds                            k8s_POD.ef28e851_nginx-kx_default_419bc51e985b6bb5e53ea305e2c1e737_836cadc7             
~~~

There are three containers running: the `nginx` and `log-truncator` containers
we defined, as well as the pod infrastructure container.[^pause] The
infrastructure container is where the kubelet puts all the resources that are
shared across containers in the pod. This includes the IP, as well as any
volumes we've defined. We can poke around with `docker inspect` to see how
they're configured and hooked up to each other:

[^pause]:
    The `pause` command that the infrastructure container runs is a 129 byte
    ELF binary that just calls the [`pause` system call][man-2-pause], and
    exits when a signal is received. This keeps the infrastructure container
    around until the kubelet brings it down. It's pretty cool, check the
    [source][pause-source]!

[man-2-pause]: http://man7.org/linux/man-pages/man2/pause.2.html
[pause-source]: https://github.com/kubernetes/kubernetes/blob/88317efb42db763b9fb97cd1d9ac1465e62009d0/third_party/pause/pause.asm

~~~
$ docker inspect --format '{{ .NetworkSettings.IPAddress }}' f1a27680e401

$ docker inspect --format '{{ .NetworkSettings.IPAddress }}' c5e357fc981a

$ docker inspect --format '{{ .NetworkSettings.IPAddress }}' b2692643c372
172.17.0.2
~~~

The nginx and log trunctator containers have no IP, but the infrastructure container does. Taking a
closer look at at the containers we defined, we can see their `NetworkMode` is set to
use the infrastructure container's network:

~~~
$ docker inspect --format '{{ .HostConfig.NetworkMode }}' c5e357fc981a
container:b2692643c37216c3f1650b4a5b96254270e0489b96c022c9873ad63c4809ce93
$ docker inspect --format '{{ .HostConfig.NetworkMode }}' f1a27680e401
container:b2692643c37216c3f1650b4a5b96254270e0489b96c022c9873ad63c4809ce93
~~~

Since we exposed port 80 from the nginx container with `containerPort`,
we can connect to the nginx server at port 80 at the pod's IP:

~~~
$ curl --stderr /dev/null http://172.17.0.2 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
~~~

It really is running! And just to check the log truncator is doing what we
expect, let's watch the log file and make some requests with

~~~
$ docker exec -tty f1a27680e401 watch cat /logdir/access.log
~~~

while we make a few requests. The log lines accumulate for a bit, but then they
all disappear: the truncator doing its job!


# Kubelet introspection

The kubelet also has an internal HTTP server. We won't go into it in detail,
except to say that it serves a read-only view at port
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

# Tearing things down

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
