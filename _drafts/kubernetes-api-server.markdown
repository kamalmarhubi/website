---
title: Kubernetes API server
---

*This is the second post in a series on Kubernetes, the open source cluster
manager. The first post was [What even is a kubelet?][kubelet-post].*

[Last time][kubelet-post] we took a look at the kubelet, Kubernetes' container-focused process
watcher. We also  and pods that lies at the bottom of Kubernetes' stack of abstractions. We saw it running pods that were specified in files in a watched
directory. This was a great way to understand the core purpose of the kubelet.
However, in a Kubernetes cluster, most of the pods the kubelets run will come
from the Kubernetes API server. In this post we'll look at the API server, and
its interaction with the kubelet.

The API server is the central component that all the other Kubernetes components
talk to. TheAll of a Kubernetes cluster's state is kept in etcd, so that it can
survive . The only component that interacts with etcd is the API
server; for any other componenet to read or modify state it must go through the
API server.

When a kubelet first starts up, it registers itself with the API server.

# Starting the API server

*You may need to use `sudo` on some commands, depending on your setup.`*

You're going to want a few terminals open for this! First off, we're going to
need etcd running. Luckily this is as easy as creating a directory for it to
store its state and starting it with Docker:

~~~
$ mkdir etcd-data
$ docker run --detach --net=host --volume=$PWD/etcd-data:/default.etcd quay.io/coreos/etcd
~~~

We use host networking so that the API server can talk to it over the loopback interface.

Next we'll want the API server binary:

~~~
$ wget https://storage.googleapis.com/kubernetes-release/release/v1.0.3/bin/linux/amd64/kube-apiserver
$ chmod +x kube-apiserver
~~~

Now we can start it up. It needs to know where the etcd server is, as well as
the server cluster IP range. We'll save talking about what the IP range is for a
later post that will go into more detail on Kubernetes' networking. For now
we'll just provide `10.0.0.0/16` so that the API server starts up without
shouting at us!

~~~
$ ./kube-apiserver --service-cluster-ip-range=10.0.0.0/16 --etcd-servers=http://127.0.0.1:2379
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

Not surprisingly, we haven't got any yet.

As a quick note on other fields in the response: the `kind` and `apiVersion` are
giving information about the API version and type of response we got. The
`selfLink` field is a canonical link for the resource in the response. The
`resourceVersion` is used for concurrency control: clients send it back when
they are changing a response, and the server can determine if there was a
conflicting write in the meantime.

All that is to say: right now we only care about the `items` field. We can use
the incredibly handy [`jq`][jq] utility to just get at the items. For example,
we can also look at what pods our cluster is running:

~~~
$ curl --stderr /dev/null http://localhost:8080/api/v1/pods | jq '.items'
[]
~~~

Hopefully there are no surprises there, either!

[jq]: https://stedolan.github.io/jq/

# Adding a node

In the last post, we had the kubelet watching for pod manifest files in a
directory we gave it via the `--config` flag. This time we'll have it get pod
manifests from the API server.

~~~
$ ./kubelet --api-servers=127.0.0.1:8080
~~~

When a kubelet starts up, it registers itself as a node with the API server. We
can check this out:

~~~
$ curl --stderr /dev/null http://localhost:8080/api/v1/nodes/ | jq '.items' | head
[
  {
    "metadata": {
      "name": "kx",
      "selfLink": "/api/v1/nodes/kx",
      "uid": "6811f7b0-5181-11e5-b364-68f7288bdc45",
      "resourceVersion": "246",
      "creationTimestamp": "2015-09-02T14:46:34Z",
      "labels": {
        "kubernetes.io/hostname": "kx"
~~~

We now have a one-node cluster! To make things a little more interesting, let's
add another kubelet. To get a second one running on the same host, we'll need to
give it a few flags to prevent them stepping on each others' toes. We'll also
need to set up a hostname in the hosts file for the second kubelet to use:

~~~
$ echo '127.0.1.2 kubelet-the-second  # delete me later!' | sudo tee --append /etc/hosts
127.0.1.2 kubelet-the-second  # delete me later!
$ sudo ./kubelet \
--api-servers=http://127.0.0.1:8080 \
--cadvisor-port=4195 \
--healthz-port=10249 \
--hostname-override=kubelet-the-second \
--port=10251 \
--read-only-port=10256 \
--root-dir=/var/lib/kubelet2
~~~

And now the API server reports two nodes:

~~~
$ curl --stderr /dev/null http://localhost:8080/api/v1/nodes | jq '.items[].metadata.name'
"kubelet-the-second"
"kx"
~~~

# Running a pod on a specific node

Let's run our nginx example from the last post. To save you copying and pasting, you can download a copy:

~~~
$ wget https://raw.githubusercontent.com/kamalmarhubi/kubernetes-from-the-ground-up/master/01-the-kubelet/nginx.yaml
~~~

We can post a pod manifest to the API server, and that will create the pod.
Right now we don't have the Kubernetes scheduler running to decide which node to
run the pod on, so we'll have to specify it ourselves. Open up the yaml file,
and add a `nodeName` at the top of the spec:

~~~
$ head nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: default
spec:
  nodeName: kubelet-the-second
  containers:
  - name: nginx
    image: nginx
~~~


~~~
ruby -ryaml -rjson -e 'puts JSON.pretty_generate(YAML.load(ARGF))' < file.yml > file.json
~~~

