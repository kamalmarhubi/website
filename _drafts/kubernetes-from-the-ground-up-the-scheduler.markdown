---
layout: post
title: Kubernetes from the ground up: the scheduler
---


*This is the third post in a series on [Kubernetes], the open source cluster
manager. The earlier posts were about [the kubelet][kubelet-post], and [the API
server][api-server-post].*

[kubernetes]: http://kubernetes.io/
[kubelet-post]: TODO
[api-server-post]: TODO

When we left off last time, we were able to run a collection of containers on a
specific Kubernetes node by posting a JSON manifest to the API server. We also
got a look at the `kubectl`, the command line client for Kubernetes, which
makes it much easier to interact with the cluster.

Oh, except that until now we haven't had a cluster, at least not in the sense
of multiple machines. In this post we're going to change that. To follow along,
you'll need a few machines—virtual, real, cloud, containers, it doesn't matter.
What does matter is

- they are all on the same network
- they all have Docker installed

I've got a few machines in the examples below: the master is `master`, while
the nodes are `node1`, `node2` and so on. I'm assuming they can all be reached
via their hostnames; feel free to substitute in their IPs instead!

# Starting the API server

We're going to breeze through starting the API server, since it's
all [straight out of the last post][api-server-section].

master$ mkdir etcd-data
master$ docker run --volume=$PWD/etcd-data:/default.etcd \
--detach --net=host quay.io/coreos/etcd > etcd-container-id
master$ wget https://storage.googleapis.com/kubernetes-release/release/v1.0.3/bin/linux/amd64/kube-apiserver
master$ chmod +x kube-apiserver
master$ ./kube-apiserver \
--etcd-servers=http://127.0.0.1:2379 \
--service-cluster-ip-range=10.0.0.0/16 \
--insecure-bind-address=0.0.0.0
~~~

The only difference is we've added `--insecure-bind-address=0.0.0.0`. TODO explain.
TODO test this actually will work across network

[api-server-section]: TODO


# Launching some nodes

This will be quick too, as we've done this a couple of times before. The only
difference here is that the API server isn't running on localhost, so we need
to include its address. I'll just show this once below, but do this on as many
nodes as you want!

node1$ ./kubelet --api-servers=localhost:8080

Yup, that's it.




# A little bit of security stuffs??

on other machines, run kubelet

check if we can avoid this with insecure-bind-address?
tokens

# Creating some pods

We're going to use `kubectl`, the command line client [we looked at in the last post][kubectl-section].Over on the master, make sure you've got `kubectl`:

master$ wget https://storage.googleapis.com/kubernetes-release/release/v1.0.3/bin/linux/amd64/kubectl
master$ chmod +x kubectl

Just to make sure everything's working, let's check out our nodes:

master$ ./kubectl get nodes
NAME      LABELS                      STATUS
awesome-node        kubernetes.io/hostname=awesome-node   Ready
TODO fix output

Excellent. To start, we'll create a pod and specify which node it should run
on.  Get the pod manifest, change the `nodeName` to one of the names from
the output of `kubectl get nodes`, and then create the pod. I picked `node2`.

master$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/03-the-scheduler/nginx-with-nodeName.yaml
master$ $EDITOR nginx-with-nodeName.yaml  # edit the nodeName field
master$ ./kubectl create --filename nginx-with-nodeName.yaml

# TODO add link to yaml source

If you check with `kubectl get pods`, you should see it running. If you're
quick, you might catch it in the `PENDING` state, before the kubelet starts it,
but it should end up `RUNNING` fairly quickly.

master$ TODO some output here

L

TODO motivate scheduler MOTIV



create a pod without a specified node

observe the second one is pending


# The scheduler

this is where the sheduler comes in

back on master:

launch scheduler

now check the status

should be running

launch another!

this is all pretty cool!

an issue:

describe a pod, try to connect to the pod IP

uh... hrm

do the same over on the node it's running on

do the same from another node

ruh-roh!

kubernetes assumes that the IPs that the pods get from docker are routable across the cluster
the next post will be a little detour into getting that working
next time 
