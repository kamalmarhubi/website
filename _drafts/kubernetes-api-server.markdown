---
title: "Kubernetes from the ground up: the API server"
layout: post
---

*This is the second post in a series on [Kubernetes], the open source cluster
manager. The first post was [about the kubelet][kubelet-post].*

[kubernetes]: http://kubernetes.io/

[Last time][kubelet-post] we took a look at the kubelet, Kubernetes'
container-focused process watcher. The kubelet runs pods, which are collections
of containers that share an IP and some volumes. In the post, we gave it pods
to run by putting pod manifest files in directory it watched. This was a great
way to understand the core purpose of the kubelet.  In a Kubernetes cluster,
however, a kubelet will get most its pods to run from the Kubernetes API
server.

[kubelet-post]: http://kamalmarhubi.com/blog/2015/08/27/what-even-is-a-kubelet/

Kubernetes stores all its cluster state in [etcd], a distributed data store with
a strong consistency model. This state includes what nodes exist in the cluster,
what pods should be running, which nodes they are running on, and a whole lot
more. The API server is the only Kubernetes component that connects to etcd; all
the other components must go through the API server to work with cluster state.
In this post we'll look at the API server, and its interaction with the kubelet.

[etcd]: https://github.com/coreos/etcd

# Starting the API server

*You may need to use `sudo` on some commands, depending on your setup.*

First off, we're going to need etcd running. Luckily this is as easy as creating
a directory for it to store its state and starting it with Docker. We'll also
save the Docker container ID so we can stop the container later.

~~~
$ mkdir etcd-data
$ docker run --volume=$PWD/etcd-data:/default.etcd \
--detach --net=host quay.io/coreos/etcd > etcd-container-id
~~~

We use host networking so that the API server can talk to it at `127.0.0.1`.

Next we'll want the API server binary:

~~~
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.0.3/bin/linux/amd64/kube-apiserver
$ chmod +x kube-apiserver
~~~

Now we can start it up. It needs to know where the etcd server is, as well as
the service cluster IP range. We'll save talking about what the IP range is for
a later post that will go into Kubernetes' services and networking. For now
we'll just provide `10.0.0.0/16` so that the API server starts up without
shouting at us!

~~~
$ ./kube-apiserver \
--etcd-servers=http://127.0.0.1:2379 \
--service-cluster-ip-range=10.0.0.0/16 
~~~

We can now `curl` around and check a few things out. First off, we can get a
list of nodes in our cluster:

~~~
$ curl http://localhost:8080/api/v1/nodes
{
  "kind": "NodeList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/nodes",
    "resourceVersion": "150"
  },
  "items": []
~~~

Not surprisingly, there aren't any yet.

As a quick note on other fields in the response: the `kind` and `apiVersion`
are giving information about the API version and type of response we got. The
`selfLink` field is a canonical link for the resource in the response. The
`resourceVersion` is used for concurrency control. Clients send it back when
they are changing a resource, and the server can determine if there was a
conflicting write to the same resource in the meantime.

All that is to say: right now we only care about the `items` field. We can use
the incredibly handy [`jq`][jq] utility to just get at the items. We'll use `jq`
to cut out noisy bits of responses throughout this post. For example, we can
look at what pods our cluster is running:

~~~
$ curl --stderr /dev/null http://localhost:8080/api/v1/pods | jq '.items'
[]
~~~

No surprises there, either!

[jq]: https://stedolan.github.io/jq/

# Adding a node

In the last post, we had the kubelet watching for pod manifest files in a
directory we gave it via the `--config` flag. This time we'll have it get pod
manifests from the API server.

~~~
$ ./kubelet --api-servers=127.0.0.1:8080
~~~

When a kubelet starts up, it registers itself as a node with the API server and
starts watching for pods to run. This is really great, because it means that
when we get to running a multinode cluster, we can add nodes without having to
reconfigure the API server.

We can check that the API server knows about our node:

~~~
$ curl --stderr /dev/null http://localhost:8080/api/v1/nodes/ \
| jq '.items' | head
[
  {
    "metadata": {
      "name": "awesome-node",
      "selfLink": "/api/v1/nodes/awesome-node",
      "uid": "6811f7b0-5181-11e5-b364-68f7288bdc45",
      "resourceVersion": "246",
      "creationTimestamp": "2015-09-02T14:46:34Z",
      "labels": {
        "kubernetes.io/hostname": "awesome-node"
~~~

We now have a one-node cluster!

# Running a pod via the API server

Let's run our nginx example from the last post:

~~~
$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/01-the-kubelet/nginx.yaml
~~~

In a complete Kubernetes cluster, the scheduler will decide which node to run a
pod on. For now, we've only got the API server and a kubelet, so we'll have to
specify it ourselves. To do this, we need to add a `nodeName` to the spec with
the node's `name` from above:

~~~
$ sed --in-place '/spec:/a\ \ nodeName: awesome-node' nginx.yaml
$ head nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  nodeName: awesome-node
  containers:
  - name: nginx
    image: nginx
    ports:
~~~

With the `nodeName` configured, we're almost ready to send the pod manifest to
the API server. Unfortunately, it only speaks JSON so we have to convert our YAML to
JSON:

~~~
$ ruby -ryaml -rjson \
-e 'puts JSON.pretty_generate(YAML.load(ARGF))' < nginx.yaml > nginx.json
~~~

Alternatively, just download the [JSON file][json-file] and [YAML
file][yaml-file]

~~~
$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/02-the-api-server/nginx.json
$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/02-the-api-server/nginx.yaml
~~~

Then edit the files so that the `nodeName` matches your hostname.

[json-file]: https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/02-the-api-server/nginx.json
[yaml-file]: https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/02-the-api-server/nginx.yaml

Now we can post the JSON pod manifest to the API server:

~~~
$ curl \
--stderr /dev/null \
--request POST http://localhost:8080/api/v1/namespaces/default/pods \
--data @nginx.json | jq 'del(.spec.containers, .spec.volumes)'
{
  "kind": "Pod",
  "apiVersion": "v1",
  "metadata": {
    "name": "nginx",
    "namespace": "default",
    "selfLink": "/api/v1/namespaces/default/pods/nginx",
    "uid": "28aa5a55-5194-11e5-b364-68f7288bdc45",
    "resourceVersion": "1365",
    "creationTimestamp": "2015-09-02T17:00:48Z"
  },
  "spec": {
    "restartPolicy": "Always",
    "dnsPolicy": "ClusterFirst",
    "nodeName": "awesome-node"
  },
  "status": {
    "phase": "Pending"
  }
~~~

After a short wait, the kubelet should have started the pod. We can check this
by making a GET request:

~~~
$ curl --stderr /dev/null http://localhost:8080/api/v1/namespaces/default/pods \
| jq '.items[] | { name: .metadata.name, status: .status} | del(.status.containerStatuses)'
{
  "name": "nginx",
  "status": {
    "phase": "Running",
    "conditions": [
      {
        "type": "Ready",
        "status": "True"
      }
    ],
    "hostIP": "127.0.1.1",
    "podIP": "172.17.0.37",
    "startTime": "2015-09-02T18:00:00Z"
  }
}
~~~

The pod is up, and it's been assigned the IP `172.17.0.37` by Docker. Docker
networking is really quite interesting, and well worth reading about. A good
place to start is [the network configuration article][docker-networking] in the Docker
documentation.

[docker-networking]: https://docs.docker.com/articles/networking/

Let's check that nginx is reachable at that IP:

~~~
$ curl --stderr /dev/null http://172.17.0.37 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
~~~

Excellent!


# The Kubernetes command line client: kubectl

While it's great to know that the API server speaks a fairly intelligible REST
dialect, talking to it directly with `curl` and using `jq` to filter the
responses isn't the best user experience. This is a great point to pause and
introduce the command line client[ `kubectl`][kubectl], which we'll use
throughout the rest of this series. It will make things __much__ nicer!

[kubectl]: http://kubernetes.io/v1.0/docs/user-guide/kubectl/kubectl.html

First off, let's download the client:

~~~
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.0.3/bin/linux/amd64/kubectl
$ chmod +x kubectl
~~~

Now we can get the list of nodes and see what pods are running:

~~~
$ ./kubectl get nodes
NAME      LABELS                      STATUS
awesome-node        kubernetes.io/hostname=awesome-node   Ready
$ ./kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
nginx     2/2       Running   0          28m
~~~

Much easier and prettier! Creating pods is also easier with `kubectl`. Let's
create a copy of the nginx pod manifest with a different name.

~~~
$ sed 's/^  name:.*/  name: nginx-the-second/' nginx.yaml > nginx2.yaml
~~~

Now we can use `kubectl create` to start another copy.

~~~
$ ./kubectl create --filename nginx2.yaml
pods/nginx-the-second
$ ./kubectl get pods
NAME               READY     STATUS    RESTARTS   AGE
nginx              2/2       Running   0          1h
nginx-the-second   0/2       Running   0          6s
~~~

Now we've got our second nginx pod running, but it reports `0/2` containers
running. Let's give it a bit and try again:

~~~
$ ./kubectl get pods
NAME               READY     STATUS    RESTARTS   AGE
nginx              2/2       Running   0          1h
nginx-the-second   2/2       Running   0          1m
~~~

We can also use `kubectl describe` to get at more detailed information on the
pod:

~~~
$ ./kubectl describe pods/nginx-the-second | head
Name:                           nginx-the-second
Namespace:                      default
Image(s):                       nginx,busybox
Node:                           awesome-node/127.0.1.1
Labels:                         <none>
Status:                         Running
Reason:
Message:
IP:                             172.17.0.38
Replication Controllers:        <none>
~~~

And just to be sure, we can check that this pod's nginx is also up and serving
requests at the pod's IP:

~~~
$ curl --stderr /dev/null http://172.17.0.38 | head -4
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
~~~

Great! So now we've seen what it's like to start a server in Kubernetes using
the command line client. We've still got a little way to go before this is a
full-blown Kubernetes cluster, but we are inching closer. Next time we'll bring
in the scheduler and add a couple more nodes into the mix.

For now, let's just tear everything down:

~~~
$ ./kubectl delete pods/nginx pods/nginx-the-second
pods/nginx
pods/nginx-the-second
$ ./kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
$ docker stop $(cat etcd-container-id)
$ sleep 20  # wait for the Kubelet to stop all the containers
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
~~~
