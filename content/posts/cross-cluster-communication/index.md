---
title: "Kubernetes Cross Cluster Communication"
date: 2021-08-29
categories:
- kubernetes
tags:
- cni
- routing
- dns
hero: /posts/introduction/Rainbow-Vortex.svg
description: Kubernetes Cross Cluster Communication
menu:
  sidebar:
    name: Kubernetes Cross Cluster Communication
    identifier: kubernetes-cross-cluster-communication
    weight: 3
draft: true
---

## Kubernetes Cross Cluster Communication 

Depending on your requirements and needs, you might not need cross cluster communcation at the pod level. 

In most cases, stateless components are running within your cluster and talking to some stateful components like databases which are running outside of the cluster. This is a common approach right now because it is still very difficult to run stateful components on a kubernetes cluster even if some solutions exists like statefulsets which add ordering and stable dns for your pods. 

However managing stateful components is a bit more involve than just running a statefulset, you will need to:
- handle a dynamic configuration of your container to define how to join a existing cluster 
- having automated backup 
- having the ability to execute some maintenance tasks like the cassandra repair process, etcd defragmentation
- having a replication in place accross two clusters to handle load, failover due to network issue, or dirusptive change which hopefully should not happen often
- 
Stateful applications are starting to be deployed on top of kubernetes with the help of kubernetes operator easing the management of those components. 

For example: ElasticSearch, Postgres and Cassandra/Scylladb have their own operator to be able to create a cluster, manage the data backup and restoration, handle upgrade..

Those operators are starting to become production ready, however it is still quite difficult to join 2 stateful components running on different Kubernetes cluster. 

While most of those issues can be solve by the development of a kubernetes operators. You might have seen a lot's projects of github or the web, dedicated to a specific type of workload, databases.. 

These operators extend kubernetes by creating some Custom Resources Definition (defining the schema of a kubernetes object) and Custom Resources (an actual kubernetes similar to the core kubernetes resources: deployments, statefulsets..). 

Your pods will be able to talk inside the same cluster, however you might be interested in a stateful database spreaded across 2 clusters. 

For example a cassandra cluster with some pods in cluster A and some others in cluster B, the will need to talk between each others which include direct routing between 2 pods in differents clusters. 

Ideally all those stateful components should have their data replicated into the other Kubernetes cluster which might be located into another region. 

In that case, I will talk about 2 Kubernetes clusters located in different regions and differents cloud provider.

The first one cluster A will be located in AWS eu-west-1, the latter cluster B on GCP eu-central-1 
TODO: fix gcp region

In order to connect component you will need connectivity to pods between cluster A and B and a way to discover the ips of the cluster located in the other region, also called service discovery.

## Cross cluster connectivity

### Non overlapping ip range

It is essential to do not have overlapping ip range while trying to communicate between different cluster.
In this scenario, ip range allocated into those 2 clusters connect overlap which mean cluster A cannot use some ips on the cluster B otherwise the router will not forward the packet correctly and will stay within the VPC.

### Using an overlay

You can use an overlay network for example calico and connect the 2 clusters wih calico reflectors.
However it can be difficult to run calico reflector on GKE or any many managed cluster, because the underlying network can change based on Google decision.

There are mutliple downsides of those solutions but they can be a good fit based on your requirements:
- On managed kubernetes like GKE, EKS, AKS you might not have control over the kubernetes nodes network.
  For example, it can be complicated to run calico and the route reflector on top on GKE, you will be dependent on GKE network change which might affect the overlay. 
- Network performance can be affected while using an overlay as it will need to encapsulated the network packet within a UDP packet which introduce some overhead.

### Using native routing

A better solution would be to use the native routing solution depending on your provider, eg: GKE native routing, AWS-cni for AWS/EKS, Azure cni.. using the native network performance of the underlying provider. 

By having those native routing capabilities, all pods will interract directly with the network stack of your VPC and route table. Therefore you can connect different cluster using transit gateway or VPC perring if both clusters are located on AWS. In our case we would like to connect and AWS cluster to a GKE cluster, we can rely on the Cloud VPN solution to connect AWS and GCP. When the network is established you will only need to update your route table to forward the ip range of the other cluster into the VPN tunnel.

## Service discovery

When pod to pod connectivity has been acheived, ideally you should rely on a service discovery mechanism to retreive dynamically the list of peers to communicate with them and join the cluster. 

### Using consul

Consul permits to register services and can be connected across mutliple clusters.

This can be a viable solution but you need to take in consideration the overhead of managing another component on top of Kubernetes.

### Using CoreDNS 

CoreDNS is now the default DNS servers deployed within the cluster. By default it will resolve a single cluster watching the apiservers to get the mapping between hostname and ips.

Kubernetai is a plugin which will allow to communicate with mutliples apiservers and get the informations of mutliples clusters.

The default kubernetes domain is `svc.cluster.local`, you can add 2 others domain one per region:
 - `svc.aws-euwest1.local` 
 - `svc.gcp-euwest1.local` 
TODO: update GCP region

Any pod on the cluster will be able to resolve headless service to get the pod ips associated to a statefulset.

## Service mesh routing

### Istio/Linkerd cross cluster


### Cilium cross cluster routing


## Discarded options

### Loadbalancer per pod with external DNS 

Some projects have been creating a loadbalancer per pod via statefulset, headless service and external-dns. 

This seems more a hack than anything else, which might not work for every components and will become very costly on big cluster.


## Conclusion

For now, there is no recommended or standard way to provide a routing and service discovery across Kubernetes clusters. 

It will depends on your requirements, using direct routing and CoreDNS/Kubernetai might be good enough to connect your different stateful component. In this case, the headless service (SRV record) will be provided to kubernetes operator. The latter will get the hostname associated and update the config of the component (eg: cassandra).

Relying on istio or cilium to provide cross-cluster routing and service discovery can also be a good approach. 
This might be explained in another article :)