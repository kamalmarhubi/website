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
to include its address. I've got two nodes, but I'll just show this once below.
If you're following along, do this on as many nodes as you want!

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

Kubernetes runs *pods*, which are collections of containers that execute
together.  To start, we'll create a pod and specify which node it should run
on.

We'll continue running our nginx example pod from the earlier posts. Get [the
pod manifest][nginx-with-nodename], which specifies which containers to run. We
specify the node to run on by setting the `nodeName` field. Edit the file and
set it to run on one of your nodes.  I picked `node2`.

~~~
master$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/03-the-scheduler/nginx-with-nodename.yaml
master$ $EDITOR nginx-with-nodename.yaml  # edit the nodeName field to match a node
~~~

Now create the pod:

~~~
master$ ./kubectl create --filename nginx-with-nodename.yaml
~~~

[nginx-with-nodename]: https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/03-the-scheduler/nginx-with-nodename.yaml

We can check with `kubectl get pods` we see that it got picked up. If you're
quicker than me, you might catch it in the `Pending` state, before the kubelet
starts it, but it should end up `Running` fairly quickly.

~~~
master$ ./kubectl get pods
NAME                  READY     STATUS    RESTARTS   AGE
nginx-with-nodename   2/2       Running   0          7s
~~~

Just to be sure it's actually on `node2` as we said, we can `kubectl describe`
the pod:

~~~
master$ ./kubectl describe pods/nginx-with-nodename | grep ^Node
Node:                           node2/10.240.0.4
~~~

We can break down what happened here:

- initially, the kubelets on each node are watching the API server for pods
  they are meant to be running
- `kubectl` created a pod on the API server that's meant to run on `node2`
- the kubelet on `node2` noticed the new pod, and so started running it.

We can also try [a pod manifest][nginx-without-nodename] that doesn't specify a
node to run on. In our current setup, this pod will forever sit in the
`Pending` state. Let's try anyway:

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

Breaking it down in the same way:

- initially, the kubelets on each node are watching the API server for pods
  they are meant to be running
- `kubectl` created a pod on the API server without specifying which node to
  run on
- ...
- ...
- ... yeah, nothing's going to happen.


# The scheduler

This is where the scheduler comes in: its job is to take pods that aren't bound
to a node, and assign them one. Once the pod has a node assigned, the normal
behavior of the kubelet kicks in, and the pod gets started.

Let's get the scheduler binary and start it running on `master`:

~~~
master$ wget https://storage.googleapis.com/kubernetes-release/release/v1.1.1/bin/linux/amd64/kube-scheduler
master$ chmod +x kubectl
master$ ./kube-scheduler --master=http://localhost:8080
~~~

Not long after starting the scheduler, the `nginx-without-nodename` pod should
get assigned a node and start running.

~~~
master$ ./kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-with-nodename      2/2       Running   0          1h
nginx-without-nodename   2/2       Running   0          1h
~~~

If we `describe` it, we can see which node it got scheduled on:

~~~
master$ ./kubectl describe pods/nginx-without-nodename | grep ^Node
Node:                           node1/10.240.0.3
~~~

It ended up on `node1`! The scheduler tries to spread out pods evenly across
the nodes we have available, so that makes sense. If you're interested in more
about how the scheduler places pods, there's a really good [Stack
Overflow](http://stackoverflow.com/a/28874577) answer with some details.

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

At this point, if you create another pod without specifying a node for it to
run on, the scheduler will place it right away. Try it out!


# Wrapping up

So now we are able to declaratively specify workloads, and get them scheduled
across our cluster, which is great! But if we actually try connecting to the
nginx servers we have running, we'll see we have a little problem:

~~~
master$ ./kubectl describe pods/nginx-with-nodename | grep ^IP
IP:                             172.17.0.2
master$ curl http://172.17.0.2
curl: (7) Failed to connect to 172.17.0.2 port 80: No route to host
~~~

This pod is running on `node2`. If we go over to that machine,
we get through:

~~~
node2$ curl --stderr /dev/null http://172.17.0.2 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
~~~

But our other node can't reach it:

~~~
node1$ curl http://172.17.0.2
curl: (7) Failed to connect to 172.17.0.2 port 80: No route to host
~~~

In the next post, we'll take a little detour into Kubernetes networking, and
make it possible for containers to talk to each other over the network.
