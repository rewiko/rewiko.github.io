---
title: "Cross Cluster Communication"
date: 2020-11-02
categories:
- kubernetes
tags:
- cni
- bpf
hero: /posts/introduction/hero.svg
description: Introduction to Sample Post
menu:
  sidebar:
    name: Cross Cluster Communication
    identifier: cross-cluster-communication
    weight: 16
draft: true
---

## Test

Depending on your requirements and needs, you might not need cross cluster communcation at the pod level. 
## Test 2

Depending on your requirements and needs, you might not need cross cluster communcation at the pod level. 

Depending on your requirements and needs, you might not need cross cluster communcation at the pod level. 
### Test 3

Depending on your requirements and needs, you might not need cross cluster communcation at the pod level. 
Depending on your requirements and needs, you might not need cross cluster communcation at the pod level. 

Depending on your requirements and needs, you might not need cross cluster communcation at the pod level. 
<!--more-->
You can have a kubernetes cluster with different CNI like flannel, calico, weave, cilium, native routing wih AWS, Azure, AliCloud CNI, and have routing working inside your cluster and communication will be natted when talking to a different network.  
In most cases you can have stateless components running on your cluster and talking to some stateful components like databases which are running outside of the cluster. This is a common approach right now because it is still very difficult to run stateful components on a kubernetes cluster even if some solutions exists like statefulsets which add ordering and stable dns for your pods. However managing stateful components is a bit more involve than just running a statefulset, you will need to:
- handle a dynamic configuration of your container to define how to join a existing cluster 
- having automated backup 
- having the ability to execute some maintenance tasks like the cassandra repair process, rtcd defragmentation
- having a replication in place accross two clusters to handle load, failover due to network issue, or dirusptive change which hopefully should not happen often

While most of those issues can be solve by the development of a kubernetes operators. You might have seen a lot's projects of github or the web, dedicated to a specific type of workload, databases.. 
These operators extend kubernetes by creating some Custom Resources Definition (defining the schema of a kubernetes object) and Custom Resources (an actual kubernetes similar to the core kubernetes resources: deployments, statefulsets..). 

Your pods will be able to talk inside the same cluster, however you might be interested in a stateful database spreaded accross 2 clusters. 
For example a cassandra cluster with some pods in cluster A and some others in cluster B, the will need to talk between each others which include direct routing between 2 pods in differents clusters. 

You can use an overlay network for example calico and connect the 2 clusters wih calico reflectors.
However it can be difficult to run calico reflector on GKE or any many managed cluster, because the underlying network can change (explain why).

Native VPC routing is my prefered way, that will allow native network performance and using the best CNI for each cloud provider. 

Service discovery DNS or ohers like consul


Use consul as service discovery but it seems amother component to manage and to integrate with kubernetes
Service discovery with coreDNS and kubernetai

Routing
Direct routing 
Calico based cross cluster communication and weave 
For Kubernetes self-managed cluster like GKE, can be quite hard to deploy on CNI which will modify network on the underlying node.

Ip space issue using 100.x.x.x ips
