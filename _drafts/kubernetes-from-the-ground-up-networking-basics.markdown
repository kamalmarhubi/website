---
title: Kubernetes from the ground up: networking basics
---

It's been a while. At the end of the last post, we were able to magically
schedule containers onto our cluster by sending them to the API controler
with kubectl. The scheduler assigned a specific node to run that container
and our little service got started. BUT we could only connect to that
service from the same machine it was running on. This is pretty limiting!

In Kubernetes, each pod (container) gets its own IP. This is the same as
what happens when you run a container direclty in Docker: you end up with
a bunch of IPs, one for each container. They can all communicate with all
the other containers running on that machine.


The unit of execution in Kubernetes is a pod, which is a collection of
related containers. Common example is a server in one container, and a log
saver in another. The log saver tails logs from the server and ships them
off to whatever cool distributed logging system you have.

Kubernetes networking has two basic requirements:

- each pod gets its own IP
- pod IPs can be routed across the network

It's not hard to get the first to work. Eg, with Docker, you can set
a container to use another container's network like this:

~~~
TODO: docker --network=container=id thing
~~~

This is what the kubelet does under the hood. Let's make a pod manifest with two nginx instances running on different ports.

~~~
TODO: maniftest
~~~

Now we can get from both of them with curl, so

~~~
TODO: curl
~~~


