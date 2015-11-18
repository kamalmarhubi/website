---
layout: post
title: "Kubernetes from the ground up: the scheduler"
date: 2015-11-17T17:41:57-05:00
---


*This is the third post in a series on [Kubernetes], the open source cluster
manager. The earlier posts were about [the kubelet][kubelet-post], and [the API
server][api-server-post].*

[kubernetes]: https://kubernetes.io/
[kubelet-post]: http://kamalmarhubi.com/blog/2015/08/27/what-even-is-a-kubelet/
[api-server-post]: http://kamalmarhubi.com/blog/2015/09/06/kubernetes-from-the-ground-up-the-api-server/

It's been a while since the last post, but I'm excited to finally finish this
one off. This is about the scheduler, which is the first part of what makes
Kubernetes Kubernetes. The scheduler's job is to decide where in the cluster to
run our workloads. This lets us stop thinking about which host should run what,
and just declaratively say ‘I want this to be running’.

When we left off last time, we were able to run a collection of containers on a
specific Kubernetes node by posting a JSON manifest to the API server. We also
got a look at the `kubectl`, the command line client for Kubernetes, which
makes it much easier to interact with the cluster.

Oh, except that until now we haven't had a cluster, at least not in the sense
of multiple machines. In this post we're going to change that. To follow along,
you'll need a few machines—virtual, real, cloud, it doesn't matter.
What does matter is

- they are all on the same network
- they all have Docker installed

I've got a few machines in the examples below: the master is `master`, while
the nodes are `node1`, `node2`. I'm assuming they can all be reached
via their hostnames; feel free to substitute in their IPs instead!


# Starting the API server

We're going to breeze through starting the API server, since it's
all [straight out of the last post][api-server-section].

~~~
master$ mkdir etcd-data
master$ docker run --volume=$PWD/etcd-data:/default.etcd \
--detach --net=host quay.io/coreos/etcd > etcd-container-id
master$ wget https://storage.googleapis.com/kubernetes-release/release/v1.1.1/bin/linux/amd64/kube-apiserver
master$ chmod +x kube-apiserver
master$ ./kube-apiserver \
--etcd-servers=http://127.0.0.1:2379 \
--service-cluster-ip-range=10.0.0.0/16 \
--insecure-bind-address=0.0.0.0
~~~

The only difference is we've added `--insecure-bind-address=0.0.0.0`. This
allows the kubelets running on the nodes to connect to the API server remotely
without any authentication. Ordinarily, unauthenticated connections are only
allowed from localhost.

Just to be clear, you *really* don't want to do this in production!

[api-server-section]: http://kamalmarhubi.com/blog/2015/09/06/kubernetes-from-the-ground-up-the-api-server/#starting-the-api-server

While we're here, let's also get `kubectl`, the command line client [we looked
at in the last post][kubectl-section]:

~~~
master$ wget https://storage.googleapis.com/kubernetes-release/release/v1.1.1/bin/linux/amd64/kubectl
master$ chmod +x kubectl
~~~

[kubectl-section]: http://kamalmarhubi.com/blog/2015/09/06/kubernetes-from-the-ground-up-the-api-server/#the-kubernetes-command-line-client-kubectl


# Launching some nodes

This will be quick too, as we've done this a couple of times before. The only
difference here is that the API server isn't running on localhost, so we need
to include its address. I'll just show this once below, but do this on as many
nodes as you want!

~~~
node1$ ./kubelet --api-servers=http://master:8080
~~~

Now back on `master`:

~~~
master$ ./kubectl get nodes
NAME      LABELS                         STATUS    AGE
node1     kubernetes.io/hostname=node1   Ready     2m
node2     kubernetes.io/hostname=node2   Ready     4s
~~~

Excellent.

# Running something on the cluster

Kubernetes runs *pods*, which are collections of caontainers that execute
together.  To start, we'll create a pod specifying which node it should run on.
Get [the pod manifest][nginx-with-nodename], which specifies which containers
to run, and change the `nodeName` to the name of one of your nodes.  I picked
`node2`. Now create the pod:

~~~
master$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/03-the-scheduler/nginx-with-nodename.yaml
master$ $EDITOR nginx-with-nodename.yaml  # edit the nodeName field to match a node
master$ ./kubectl create --filename nginx-with-nodename.yaml
~~~

[nginx-with-nodename]: https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/03-the-scheduler/nginx-with-nodename.yaml

The kubelet on each node is constantly watching for pods to run. We said it
should run on `node2`, and when we check with `kubectl get pods` we see that it
got picked up. If you're quicker than me, you might catch it in the `Pending`
state, before the kubelet starts it, but it should end up `Running` fairly
quickly.

~~~
master$ ./kubectl get pods
NAME                  READY     STATUS    RESTARTS   AGE
nginx-with-nodename   2/2       Running   0          7s
~~~

Just to be sure it's actually on `node2` as we said, we can `kubectl describe` the pod:

~~~
master$ ./kubectl describe pods/nginx-with-nodename | grep ^Node
Node:                           node2/10.240.0.4
~~~

We can also try [a pod manifest][nginx-without-nodename] that doesn't specify a
`nodeName`. In our current setup, this pod will forever sit in the `Pending`
state. because each kubelet is only interested in pods with their name as
`nodeName`. Let's try anyway:


~~~
master$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/03-the-scheduler/nginx-without-nodename.yaml
master$ ./kubectl create --filename nginx-without-nodename.yaml
master$ ./kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-with-nodename      2/2       Running   0          3m
nginx-without-nodename   0/2       Pending   0          20s
~~~

[nginx-without-nodename]: https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/03-the-scheduler/nginx-without-nodename.yaml

Even if you take a break and read the internet for 15 minutes, it'll still be
there, `Pending`:

~~~
master$ ./kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-with-nodename      2/2       Running   0          18m
nginx-without-nodename   0/2       Pending   0          15m
~~~


# The scheduler

This is where the sheduler comes in: its job is to take pods that aren't bound
to a node, and assign them one. Once the pod has a `nodeName`, the normal
behavior of the kubelet kicks in, and the pod gets started. Back on the master:

~~~
master$ wget https://storage.googleapis.com/kubernetes-release/release/v1.1.1/bin/linux/amd64/kube-scheduler
master$ chmod +x kubectl
master$ ./kube-scheduler --master=hp://localhost:8080
~~~

Not long after starting the scheduler, the `nginx-without-nodename` pod should
get assigned a node and start running.

~~~
master$ ./kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-with-nodename      2/2       Running   0          1h
nginx-without-nodename   2/2       Running   0          1h
~~~

If we `describe` it, we can see which node it ended up on:

~~~
master$ ./kubectl describe pods/nginx-without-nodename | head -4
Name:                           nginx-without-nodename
Namespace:                      default
Image(s):                       nginx,busybox
Node:                           node1/10.240.0.3
~~~

We can also get a list of ‘events’ related to the pod. These are state changes
through the pods lifetime:

~~~
master$ ./kubectl describe pods/nginx-without-nodename | grep -A5 ^Events
Events:
  FirstSeen     LastSeen        Count   From            SubobjectPath                           Reason                  Message
  ─────────     ────────        ─────   ────            ─────────────                           ──────                  ───────
  25m           25m             1       {scheduler }                                            Scheduled               Successfully assigned nginx-without-nodename to node1
  23m           23m             1       {kubelet node1} implicitly required container POD       Pulling                 Pulling image "gcr.io/google_containers/pause:0.8.0"
  23m           23m             1       {kubelet node1} implicitly required container POD       Pulled                  Successfully pulled image "gcr.io/google_containers/pause:0.8.0"
~~~

The first one shows it getting scheduled, then others are related to the pod
starting up on the node.


# What's next?

TODO: bad title

describe a pod, try to connect to the pod IP

uh... hrm

do the same over on the node it's running on

do the same from another node

ruh-roh!

kubernetes assumes that the IPs that the pods get from docker are routable across the cluster
the next post will be a little detour into getting that working
next time 
