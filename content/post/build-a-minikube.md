---
title: "Build a Minikube From Scratch"
date: 2020-02-23T17:14:13+05:30
---

Inspired from this [What-is-even-kubelet series](http://kamalmarhubi.com/blog/2015/08/27/what-even-is-a-kubelet/) of blogs.

In this blog, we will build a minikube from scratch. This is targetted at people who are already familiar with Kubernetes and are interested to learn about the internal key components of the Kubernetes master by building our own Kubernetes cluster by hand.

## Components - The Recap

To refresh you about the various components of master from the official [blog](https://kubernetes.io/docs/concepts/overview/components/), the components are

1. Kube API Server
2. Scheduler
3. Controller Manager
4. ETCD
5. Cloud Controller Manager
6. Kubelet
7. Kube Proxy

However, let us not go through what those components do, for now. Instead, we will take a bottom-up approach of building a cluster with components that we require then and there from the ground up.

## Prerequisites

1. VirtualBox
2. Vagrant

> I have not tested these steps on a Linux machine.

## Steps

#### Step 1: A VM with necessary tools installed.

```bash
$ git clone https://github.com/aswinkarthik/build-a-minikube
$ cd build-a-minikube
$ vagrant up
```

As part of provisioning the following are installed,

1. Docker
2. Kubelet
3. Kubectl

*Check your setup by*

```
$ vagrant ssh

## Switch to root
vagrant@k8s-master:~$ sudo su -

root@k8s-master:~$ kubelet --version
Kubernetes v1.17.0

root@k8s-master:~$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.0", GitCommit:"70132b0f130acc0bed193d9ba59dd186f0e634cf", GitTreeState:"clean", BuildDate:"2019-12-07T21:20:10Z", GoVersion:"go1.13.4", Compiler:"gc", Platform:"linux/amd64"}
```

#### Step 2: Kubelet

Kubelet is the basic building block of a Kubernetes cluster. Kubelet runs pods. To run pods, let us use the `--pod-manifest-path` flag.

```
root@k8s-master:~$ mkdir manifests

root@k8s-master:~$ kubelet --pod-manifest-path ./manifests
I0309 16:59:13.825187    4553 server.go:416] Version: v1.17.0
I0309 16:59:13.833665    4553 plugins.go:100] No cloud provider specified.
W0309 16:59:13.833703    4553 server.go:555] standalone mode, no API client
...
```

Kubelet is now running, watching for pod specifications in the directory `manifests`. Let us drop a simple pod YAML inside the directory.

```YAML
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
  hostName: k8s-master
```

*Start another session*

```
$ vagrant ssh
vagrant@k8s-master:~$ sudo su -

root@k8s-master:~$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
## No containers

## Drop a file
root@k8s-master:~$ cp /src/01-simple-nginx-pod.yml ./manifests/
root@k8s-master:~$ watch docker ps
```

The `Nginx` container should be running. Kubelet converted our pod specification to appropriate `docker run` commands i.e container runtime commands. Let us kill the container.

```
root@k8s-master:~$ docker stop <nginx-container-id>
root@k8s-master:~$ watch docker ps
```

The `nginx` container comes back up. We just saw 2 responsibilities of Kubelet.

**Kubelet Responsibilities**

1. Convert pod specifications to container runtime commands.
2. Make sure the container is running by constantly monitoring the container runtime.

**But there was a pause,**

We noticed that there were 2 containers running (even if we did not specify any sidecars)

1. nginx
2. pause

Before we look into it,

**Access the nginx**

Let us try to access the nginx container. To get the IP address of the container.

```
root@k8s-master:~$ docker ps | grep " nginx "
ae19ebe965b8        nginx                  "nginx -g 'daemon of…"   4 minutes ago
root@k8s-master:~$ docker inspect ae19ebe965b8 | jq '.[0].NetworkSettings'
{
  ...
  "SecondaryIPAddresses": null,
  "Gateway": "",
  "IPAddress": "",
  "MacAddress": "",
  ...
}
```

We notice that the nginx container does not have any IP address associated with it. However, doing the same for the `pause` container.

```
root@k8s-master:~$ docker inspect 9c47f90031a3 | jq '.[0].NetworkSettings.IPAddress'
"172.17.0.2"
root@k8s-master:~$ curl 172.17.0.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

If we access the pause container, we get the nginx welcome page.

This is because, the nginx docker container's network is configured to be the pause container i.e by doing a `docker run --network=container:<pause-container-id>`. This would make both the container share the *network namespace* i.e both containers can access each other through `localhost`.

The primary reason why the `pause` container is added is to retain the linux namespace from getting removed in case the nginx container crashes/halts. (if all process under a given namespace is stopped, the namespace is also lost). More info can be found in the [Almighty Pause Container](https://www.ianlewis.org/en/almighty-pause-container) blog.

![Pod Zoomed In](/build-a-minikube/pod-zoomed-in.png)

Let us try using the CLI

```
root@k8s-master:~$ kubectl get pods
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

The `kubectl` client still needs an API server to talk to. Since we now no how to run workloads on docker through kubelet, let us run the API server.

#### Step 3: API Server

API Server is the single point of contact for users/components of a cluster. Using verbose mode in the above command reveals that the client `kubectl` does a RESTful GET request to the API.

```
root@k8s-master:~$ kubectl get pods -v=8
```

The API Server needs ETCD as storage. Let us deploy both these components to our cluster.

```
root@k8s-master:~$ cp /src/02-api-server.yml ./manifests/
root@k8s-master:~$ cp /src/03-etcd.yml ./manifests/
root@k8s-master:~$ docker ps
## API Servers and ETCD with their respective pause container starts
```

Now rerunning the same command,

```
root@k8s-master:~$ kubectl get nodes
No resources found in default namespace.
```

As you can see now, a successful response is returned.

![Kubelet with API Server](/build-a-minikube/kubelet-with-api-server.png)

No nodes were returned in the `kubectl` command because no `kubelet` is connected to the API Server. We do have a `kubelet` running. Let us connect it to the API Server. Stop the kubelet and re-run it by

```
root@k8s-master:~$ kubelet --pod-manifest-path ./manifests --kubeconfig /src/kubeconfig.yml
```

Now, if we query the API server for nodes

```
root@k8s-master:~$ kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-master   Ready    <none>   22s   v1.17.0
```

We have a node up and running. It is time to "deploy" an Nginx instead of running it as a pod.


#### Step 4: Deploy Nginx

As we now have nodes, it is time to run our nginx workload as a deployment so that we can push updates and make sure it has desired number of replicas up all the time.

```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

```
root@k8s-master:~$ kubectl apply -f /src/04-nginx-deployment.yml
deployment.apps/nginx-deployment created
root@k8s-master:~$ kubectl get pods
## No pods from nginx-deployment
```

> Ignore the "nginx-k8s-master" pod so as to not cause confusion. It was the test pod we created initially.

*What could be the reason?*

Let us see if atleast the *ReplicaSets* were created.

```
root@k8s-master:~$ kubectl get replicaset
No resources found in default namespace.
```

Even replicasets have not been created.

Let us add another component to our cluster: *ControllerManager*.

```
root@k8s-master:~$ cp /src/05-controller-mngr.yml ./manifests/
````

Wait for sometime for the container to startup.

```
root@k8s-master:~$ kubectl get replicaset
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-54f57cf6bf   2         2         0       3m6s
```

Our *Replicaset* is up. Let us check the pods.

```
root@k8s-master:~$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54f57cf6bf-5gbz7   0/1     Pending   0          3m32s
nginx-deployment-54f57cf6bf-d5ftb   0/1     Pending   0          3m32s
```

Our pods are still not in running state. They are in *Pending* state.

*What could be the reason?*

Let us add another component to our cluster: *Scheduler*.

```
root@k8s-master:~$ cp /src/06-scheduler.yml ./manifests/
## Wait for it to run
root@k8s-master:~$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
nginx-deployment-6fcf476c4-7td5l   1/1     Running   0          20s
nginx-deployment-6fcf476c4-q664p   1/1     Running   0          22s
## Pods are running
````

Pods are finally in running state in our cluster.

#### Step 5: Components Responsibilities

**Controller Manager Responsibilities**

- Created ReplicaSet watching the Deployment, Created Pods watching the replica set.
- Responsibile for the reconciliation loop i.e checks what is the *Desired* state and checks the *Current* state, performs actions to converge *current* to *desired*.
- Communicates via the API server to fetch the different resources it monitors.

**Scheduler Responsibilities**

- Watched for pods with *Pending* status.
- Assigned node to pods by setting `.spec.nodeName` of the pod spec.

**ETCD Responsibilities**

- Distributed Store.
- Stores all state (E.g Pod spec, Deployment spec).
- Stateful component of the cluster.

**API Server Responsibilities**

- A RESTful service to access Etcd.
- Point of contact/interaction for all the components.
- Provides a way to "watch" for changes.


## In the end

Our cluster has the capability to accept a `Deployment` resource and create `Pods` to run our workloads.

![Final state](/build-a-minikube/final-state.png)

> Output of [kubectl tree](https://github.com/ahmetb/kubectl-tree)

**What is not covered (yet)?**

- DNS Resolution
- Networking across containers
- Kube Proxy

**Revisiting the components image**

![Revisiting official blog](/build-a-minikube/components-of-kubernetes.png)