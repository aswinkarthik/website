---
title: "Build a Minikube From Scratch"
date: 2020-02-23T17:14:13+05:30
draft: true
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

We will go a bottom-up approach of building this cluster with components that we require then and there.

## Prerequisites

1. VirtualBox
2. Vagrant

1. Docker
2. Kubelet

```bash
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl

chmod +x ./kubectl

kubectl version --client
```