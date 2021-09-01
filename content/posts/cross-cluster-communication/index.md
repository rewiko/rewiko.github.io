---
title: "Kubernetes Cross Cluster Communication"
date: 2021-09-01
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
draft: false
---

## Kubernetes Cross Cluster Communication 

Depending on your requirements and needs, you might not need cross cluster communication at the pod level. 

In most cases, stateless components are running within your cluster and talking to some stateful components like databases which are running outside of the cluster. This is a common approach right now because it is still very difficult to run stateful components on a kubernetes cluster even if some solutions exist like statefulsets which add ordering and stable dns for your pods. 

However managing stateful components is a bit more involved than just running a statefulset, you will need:
- a dynamic configuration of your container to define how to join a existing cluster 
- automated backup 
- the ability to execute some maintenance tasks like the cassandra repair process, etcd defragmentation
- a replication in place across two clusters to handle load, failover due to network issue, or disruptive change which hopefully should not happen often

Stateful applications are starting to be deployed on top of kubernetes with the help of kubernetes operators easing the management of those components. 

For example: ElasticSearch, Postgres and Cassandra/Scylladb have their own operator to be able to create a cluster, manage the data backup and restoration, handle upgrades..

These operators extend kubernetes by creating some Custom Resources Definition (defining the schema of a kubernetes object) and Custom Resources (an actual kubernetes similar to the core kubernetes resources: deployments, statefulsets..). 

By default your pods will be able to talk inside the same cluster, however you might be interested in a stateful database spreaded across 2 clusters. 

For example a cassandra cluster with some pods in cluster A and some others in cluster B, they will need to talk between each other which include routing and service discovery between 2 pods in different clusters. 

Ideally all those stateful components should have their data replicated into the other Kubernetes cluster which might be located into another region. 

In that case, I will talk about 2 Kubernetes clusters located in different regions and different cloud providers.

The first one cluster A will be located in AWS eu-west-1, the latter cluster B on GCP europe-west4-c.

In order to connect components you will need connectivity to pods between cluster A and B and a way to discover the ips of the cluster located in the other region, also called service discovery.

## Cross cluster connectivity

### Non overlapping ip range

It is essential to not have overlapping ip range while trying to communicate between different clusters.
In this scenario, ip range allocated into those 2 clusters connect overlap which means cluster A cannot use some ips on cluster B otherwise the router will not forward the packet correctly and will stay within the VPC.

### Using an overlay

You can use an overlay network for example calico and connect the 2 clusters with calico reflectors.
However it can be difficult to run calico reflector on GKE or any managed cluster, because the underlying network can change based on the provider's decision.

There are multiple downsides of those solutions but they can be a good fit based on your requirements:
- On managed kubernetes like GKE, EKS, AKS you might not have control over the kubernetes nodes network.
  For example, it can be complicated to run calico and the route reflector on top of GKE, you will be dependent on GKE network change which might affect the overlay. 
- Network performance can be affected while using an overlay as it will need to encapsulate the network packet within a UDP packet which introduces some overhead.

### Using native routing

A better solution would be to use the native routing solution depending on your provider, eg: GKE native routing, AWS-cni for AWS/EKS, Azure cni.. using the native network performance of the underlying provider. 

By having those native routing capabilities, all pods will interact directly with the network stack of your VPC and route table. Therefore you can connect different clusters using transit gateway or VPC peering if both clusters are located on AWS. In our case we would like to connect an AWS cluster to a GKE cluster, we can rely on the Cloud VPN solution to connect AWS and GCP. When the network is established you will only need to update your route table to forward the ip range of the other cluster into the VPN tunnel.

## Service discovery

When pod to pod connectivity has been achieved, ideally you should rely on a service discovery mechanism to retrieve dynamically the list of peers to communicate with them and join the cluster. 

### Using consul

Consul permits to register services and can be connected across multiple clusters.

This can be a viable solution but you need to take in consideration the overhead of managing another component on top of Kubernetes.

### Using CoreDNS 

#### Kubernetai plugin

CoreDNS is now the default DNS server deployed within the cluster. By default it will resolve a single cluster watching the apiservers to get the mapping between hostname and ips.

[Kubernetai](https://github.com/coredns/kubernetai) is a plugin which will allow to communicate with multiple apiservers and get the informations of multiples clusters.

The default kubernetes domain is `svc.cluster.local`, you can add 2 others domain one per region:
 - `svc.aws-euwest1.local` 
 - `svc.gcp-europe-west4-c.local` 

Any pod on the cluster will be able to resolve headless service to get the pod ips associated with a statefulset.

#### Network Load Balancer + DNS forwarding

Another way would be to create a network load balancer pointing to CoreDNS service foreach region and update CoreDNS to forward dns query to the other cluster for a specific domain.

Useful link:

- https://www.cockroachlabs.com/docs/stable/orchestrate-cockroachdb-with-kubernetes-multi-cluster.html?filters=eks

## Service mesh routing

### Istio/Linkerd cross cluster

Service meshes like Istio or Linkerd allow cross cluster routing via their gateway which has the full network knowledge of both clusters. Service mesh brings a lot of extra feature like mutualTLS, improve load balancing feature (circuit breaker, A/B testing) and observability (distributed tracing, sidecar metrics).
Service mesh will need a complete blog post to explain all those features.

Useful links:

- https://istio.io/latest/docs/setup/install/multicluster/
- https://linkerd.io/2.10/features/multicluster/
  
### Cilium cross cluster routing

Cilium mesh will allow you to send traffic to pods behind a service to different clusters. The cluster ip will be different on each cluster but the list of pod ips will be the all pods available on the cluster.

Useful links: 

- https://cilium.io/blog/2019/03/12/clustermesh
- https://docs.cilium.io/en/v1.9/gettingstarted/clustermesh/

## Conclusion

For now, there is no recommended or standard way to provide routing and service discovery across Kubernetes clusters. 

It will depend on your requirements, using direct routing and CoreDNS/Kubernetai might be good enough to connect your different stateful components. 

In this case, the headless service (SRV record) will be provided to the kubernetes operator. The latter will get the hostname associated and update the config of the component (eg: cassandra).

Relying on Istio/Linkerd or Cilium to provide cross-cluster routing and service discovery can also be a good approach for stateless services. 

