---
title: "Kubernetes Network Policies - Cilium & EBPF"
date: 2021-08-15
categories:
- kubernetes
tags:
- cni
- bpf
- network
- cilium
hero: /posts/introduction/Rainbow-Vortex.svg
description: Kubernetes Network Policies - Cilium & EBPF
menu:
  sidebar:
    name: Kubernetes Network Policies - Cilium
    identifier: kubernetes-network-policies-cilium-ebpf
    weight: 2
draft: false
---

## Secure your network with network policies

Kubernetes comes with a **flat network** by default where every pod can talk to each other. 

**Kubernetes Network Policy** is a concept which allows you to segregate the network within your cluster. 

Multiple CNI are available to implement network policies. **Cilium** and **Calico** are the main CNI available to secure your network. 

Initially Calico was relying on iptables rules to block/allow ingress/egress traffic related to your pod. Nowadays Calico also provides an EBPF implementation. 

## What's wrong with iptables?

Iptables is the default way to load balance kubernetes services and block network flows within a kubernetes cluster.

Iptables is chaining the rules defined, on large cluster this will introduce:
 - service load balancing extra latency 
 - delay on network policies propagation, latency to add/remove rules

When a TCP connection is created, it will need to go through every single iptables rule until it matches with one of them.
Iptables introduces a complexity O(n), which means it will not scale very well and it will introduce a latency increase when reaching a large number of kubernetes services and pods, because every new connection will need to traverse all these rules. 
Also adding and removing iptables is a sequential process with a locking mechanism, it will slow down the network policies propagation to the cluster. 

For those reasons, iptables is not the most scalable solution to segregate the network and to load balance your kubernetes services.

More infos: 

[Slideshare - Scale Kubernetes to Support 50,000 Services [I] - Haibin Xie & Quinton Hoole](https://www.slideshare.net/LCChina/scale-kubernetes-to-support-50000-services)

[Youtube video - Scale Kubernetes to Support 50,000 Services [I] - Haibin Xie & Quinton Hoole](https://www.youtube.com/watch?v=4-pawkiazEg)

## How Cilium works?

Cilium has always been relying on EBPF to improve the security of your cluster and provides also a lot's of feature like: 
- a more efficient load balancing using EBPF (replacing kube-proxy)
- deny and host policies allowing to block specific ips on the host level or at the pods layer
- cross-cluster routing
- observability through hubble, cilium monitor
  

Cilium will allow you to (non-exhaustive list):
- Restrict network communication within the cluster
- Allow and block access to external endpoint (databases) to a specific namespaces/applications 
- Block "malicious" ips at the cluster level, the owner of the kubernetes namespaces won't be able to overwrite blocked ips defined at the cluster level (Deny Cluster-wide Cilium Network Policies)
- Least privilege access (block any traffic by default)

Cilium offers a lot of functionalities, we are going to see in this article "How cilium works" and the importance of a good testing strategy. 

## What is EBPF?

### EBPF  

EBPF allows you to load programs within the kernel without the need to recompile it. 

You can attach a program to a specific hook: tc ingress or egress, XDP layer (closest to the NIC).

More infos: [EBF.io](https://ebpf.io/) [Cilium EBPF guide](https://docs.cilium.io/en/v1.10/bpf/)
### Cilium & EBPF and Network Policies 

Cilium compiles and loads a bpf program into the kernel. This program will be attached to the container interface more precisely using the tc hook (ingress or egress).
When a packet arrives :
 1. the bpf program will get the source or destination ip
 2. it will translate the ip to an identity using the **ipcache** (bpf map containing the mapping between an ip and an identity)
 3. after being identity aware, the bpf program will check if the identity is allowed or blocked by checking the bpf policy map, this map is calculated depending on the Cilium Network Policies and standard Network Policies within the cluster
 4. if the identity is not allowed then the packet will be dropped
   
A bpf policy map is created for each cilium endpoint/pod, the list of identities allowed for ingress and egress will be populated by cilium-agent within the bpf policy map. 

[EBPF Code](https://github.com/cilium/cilium/blob/master/bpf/lib/policy.h#L18) blocking egress and ingress based on policy 
### Cilium & EBPF and Service Load balancing

Cilium can also handle kubernetes service load balancing and replace kube-proxy.
Essentially at every connection creation, it will perform a DNAT (destination nating or modifying the destination ip from service ip to pod ip). 
Cilium manages the mapping between service ip and pod ips into the load balancer map and keeps the state of the connection without the connection tracking bpf map, therefore it does not rely on the usual kernel connection tracking table.

The load balancing can be packet or socket based and the ct bpf map will be used to track the connections.
I would recommend using socket based load balancing because the solution per packet seems to have an issue creating connection reset when a pod is deleted. 

This issue is worked on https://github.com/cilium/cilium/issues/14844, to use socket load balancing you can set `kube-proxy-replacement: "partial"`.

## Make seamless change on your infrastructure

Multiples strategy can be used to manage your clusters. 

You can treat them like ephemeral clusters and they can be destroyed at any time. 

In this case, the migration to a new technology can be easier, you do not have to worry about in-place upgrade of components. 
However this strategy can be quite hard to implement:
 - might requires dev teams to make change on their deployments to handle multiple ingresses hostnames
 - stateful components running on an existing cluster make it difficult to migrate to a new cluster
You can also choose to deploy your component on a specific node pool on a subset of your cluster (canary deployment).

Another strategy will be to do in-place upgrade of components in a live cluster. 

Whatever strategy you have chosen, you should establish a proper automated testing strategy. 

Every single change in your infrastructure should come with tests: unit, integration, functional, smoke and NFT (non-functional test) load tests and chaos/resilient tests.

Now, you have a live kubernetes cluster and you would like to upgrade or deploy a component on your cluster.
 
You would basically would like to:
- Ensure the network path is not affected, especially during upgrade.
- Reproduce what applications are doing on the cluster for example DNS/UDP and HTTP/GRPC/TCP flow
- Exercise all network paths: internal, external ingresses, service load balancing, pod to host..
  
## Optimize Cilium to scale efficiently

Cilium uses EBPF and relies on identity allowing a fast network policy propagation at scale.

This is very important to test to scalability of the component, for cilium we got some issues:

### Use dedicated etcd on big cluster - more than 250 nodes

 We have been using CRD mode at this time Cilium was configured to update the status of the Custom resources like Cilium endpoints and Cilium identities quite often. It was generating a lot of updates on a quite large cluster (400 nodes). Every update is sent to the apiserver which receives a significant load especially during rolling restart of cilium-agent. 
 All those modifications were also excercising etcd cluster and the database size grew up quite quickly due to the MVCC (Multiversion concurrency control) mechanism which essentially creates a copy of the data at every update.
 Etcd became fragmented very quickly and reached the etcd database size limited to 2Gb by default.

 We have been tweaking etcd to run the compaction more frequently and increased the maximum database size to 8GB.
 Cilium-agent has now some rate-limiting functionality to avoid this problem and by default update less status information. 

 We also have been moving away from CRD and are now using a dedicated etcd as recommended on the cilium documentation for clusters bigger than 250 nodes.
### Restrict number of identities generated 

 It is also a good practice to restrict the number of identities:
    - the bpf policy map size is by default limited to 16k entries and can be extended up to 64k
    - when using cluster entity, cilium will inject all identities into the bpf maps
    We have been restricting cilium identity creation only based on specific namespace labels which means you can have a single entity per namespace and tenants using the platform cannot generate more identity and we limit the identities churn because the namespace labels are not updated frequently. The pod label can contain the application version which will change at every new deployment.  
   
## Automated Tests through a pipeline

To gain confidence on a new component like cilium, we need to create a continuous integration and delivery pipeline to properly test the expected behavior. It is important to reproduce what you might expect for the next few months/years to prove the scalability of the component. 

### Functional & Integration test

As part of the functional and integration tests, we need to test to right behavior of cilium configured on the cluster:

- run connectivity test suite provided by cilium which will check dns, http connectivity and cilium network policies behavior, essentially run pods and apply network policies
- add tests to validate specific functional requirement: 
  - cannot override clusterwide deny policies with a namespaced Cilium network policy 
  - cilium identities are limited to specific labels 
  - network policies should apply to existing TCP connections 

Those tests are essential to prove the functionality brought by cilium and ensure their behavior within your cluster with additional CNI, specific OS, version of kubernetes... 

### Non-Functionnal test - NFT 

Non-functional are here to prove the scalability of the chosen solution.
The NFT reproduces the worst-case scenario which helps validate the future of the platform.

For Cilium, thousands of namespaces and pods will be created to produce a large number of identities.
Load is generated using a load-injector (k6, gatling..) sending requests to a backend through service ip, internal ingress and external ingress.

Testing a migration from cilium 1.7 to 1.9 through a pipeline: 
- Run existing architecture
- Creation namespaces with pod running to generate a significant number of cilium identities
- Exercise network flow by running load-injector sending request to a backend through service ip, external and internal ingresses
- Exercise DNS/UDP network path (service ip CoreDNS) load-injector and backend resolving kubernetes hostname
- While the load is injected, cilium 1.9 will be deployed 
- Chaos testing in the background (resilient to failures): 
    -  deleting cilium-agent pod
    -  deleting cilium-operator pod
    -  deleting cilium-etcd pod (dedicated node)
    -  deleting backend (pod ip churn, graceful shutdown, recycle TCP connections: the load injector will be forced to create new connection)
- Creation identities churn by deleting and recreating namespace/pods 
- Creation and deletion of cilium network policies (adding and removing items into the policy bpf maps)

For scalability reasons we restrict the label which can be associated to an identity, only specific labels defined at the namespace level can generate an identity.

Therefore tenants cannot generate thousands of identities while adding/updating labels on the pod.
Those identities will need to be injected into the bpf map which has a fixed size from 16k by default up to 64k entries. 

The injector have been hiding some connection reset, to highlight the issue another downstream dependencies has been added:

```
injectors -> backend -> downstream backend through service ip
                     -> downstream backend through internal ingress
                     -> downstream backend through external ingress
```

The backend sends the requests sent by gatling to the downstream via ingress and also service ip to test the full network flow when downstream sends a reset, the backend acting as a middleman will send back a specific status code to the injector.

The test relies on alerts to validate the result: alerts on latency increase, http errors, packet drop.

### Keep improving - Regression test 

It is hard to think of every single use case when defining your tests. Therefore you can keep improving them if your pipeline is already defined and overtime it will get better.

Dev teams might report issue, you should: 
 - investigate
 - reproduce the issue through the test 
 - add regression test 
 - deploy a fix through the pipeline, the deployment to an environment should be automated and it is important to build a promotion mechanism by checking alerts on the lower environment before promoting the new version to the next environment.

## Conclusion

Overall Cilium and EBPF are just amazing technologies which allow you to secure your network within kubernetes.

I would easily recommend it to anyone. Try it! 
 
