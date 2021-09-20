---
title: "Kubernetes Ingress Load Balancing"
date: 2021-09-12
categories:
- kubernetes
tags:
- loadbalancing
hero: /posts/introduction/Rainbow-Vortex.svg
description: Kubernetes Ingress Load Balancing
menu:
  sidebar:
    name: Kubernetes Ingress Load Balancing
    identifier: kubernetes-ingress-load-balancing
    weight: 4
draft: true
---

## Kubernetes Ingress Load Balancing 

Kubernetes ingress is the standard specification allowing external traffic to reach your application running within the cluster.

Tenants on the platform create an ingress resource to define the hostname, path and the service exposed.


## Network flow
### BGP/ECMP

BGP is a protocol which permit to advertise ip and route, routers are using BGP to cummunicate.

ECMP allow sharing a path between routers to do not break network communication when a router is down the traffic can be replay by the second router without much disruption.

For NLB AWS does the BGP advertisement for you, allowing defining the network route. The routers will be aware on how to communicate to the 3 elastic ips associated to the NLB.

### Layer 4 

The layer 4 of the OSI define the TCP communication, establish connection from a client to a target/server.

### Layer 7

The layer 7 of the OSI define the data layer, HTTP stack like payload, method, path, also the SSL handshake is hapenning at this layer.

## Direct return, XDP loadbalancing 

### Anycast ip 

Anycast ip is a way to advertise an ip pointing to different location based on where the client lives.
For example for a CDN, this concept can be used to allow the client to communicate to the closest point of presence.

### Direct return 

Most of the loadbalancer will communicate to the downstream application and reply back to the client.

Direct return or DSR allow you to forward the packet to the downstream application and this downstream application will reply directly to the client. DSR will allow to scale better by reducing the network bandwidth and hops.

### IPVS 

IPVS is a loadbalancer in the kernel

Katran 

Cilium external 

Katran, cilium external load balancer 
DDOS mitigation, more efficient 
Cloudflare

## Kubernetes 

### NLB tuning

AWS NLB is a layer 4 load balancer, forwarding packets to targets without terminating a TCP connection. 
It is probably using a consistent hashing which hash source ip and port to send the client to the same target.
NLB is is a great load balancer but we are going to see a few optimizations to use the full potential of the loadbalancer. 

#### Disable TLS termination

I managed to test a NLB by sending 1 million TPS without ramp-up sending request to a nginx ingress controller deploymemt.

Enabling SSL reduced considerably the throughput to 9k TPS. I would advise to disable the ssl termination on the NLB and handle it on the layer 7 loadbalancer targeted. For example in the Kubernetes world the NLB would send traffic to the ingress controller allowing you to manage mutilple certificates (SNI and mutual SSL) and allowing you to have full control over the number of replicas.

#### Preserve client source ip

NLB send only traffic to the node where nginx is running, preserve ip and ensure to gracefully shutdown pod while the instance is terminated
 externalTrafficPolicy set to Local
https://www.asykim.com/blog/deep-dive-into-kubernetes-external-traffic-policies

https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip

https://stackoverflow.com/questions/60276403/kubernetes-pods-graceful-shutdown-with-tcp-connections-spring-boot

```
At this point the cluster nodes will be reconfigured to remove any rules directing new traffic to the Pod. Any existing TCP connections to the Pod/containers will remain in connection tracking until they are closed (by the client, server or network stack).

For HTTP Keep Alive or HTTP/2 services, the client will continue hitting the same Pod Endpoint until it is told to close the connection (or it is forcibly reset)

```
This will also ensure that when a kubernetes node go down one of the ingress controller will be running on this node which will honor the graceful shutdown of this pod and not shutdown sooner.


#### HTTP healthcheck

https://medium.com/flant-com/kubernetes-graceful-shutdown-nginx-php-fpm-d5ab266963c2


#### Disable cross load balancing

Disable cross load balancing 
https://medium.com/swlh/nlb-connection-resets-109720accfc6
https://www.niels-ole.com/cloud/aws/linux/2020/10/18/nlb-resets.html


### Ingress Controller 

An ingress controller will watch those resources and reload their configuration. Mutliple ingress controller are available, to name a few: Traefik, Contour, Ambassador, Istio Gateway, Nginx. Those ingress controllers are acting as a loadbalancer layer 7 which mean it will be able to read the full path and being HTTP aware (read http path..).

One recommend way to handle ingress traffic is to front this ingress controller with a Load balancer layer 4, (TCP aware).
A Load Balancer layer 4 can handle more traffic than a layer 7, it is faster to process a connection without reading the content of the packet.

On AWS, the Network Load Balancer (NLB) is acting at the layer 4 and forward the packet to the target without terminating the connection.

### HTTP Graceful Shutdown 

shutdown-grace-period nginx ingress controller and TernminationGracePeriod
catch SIGTERM and send QUIT signal to nginx
Drain non new connection are accepted 

Close connection by sending a FIN packet 
#### Idle Connection

max keepalive request 
Keepalive timeout 
Keepalive TTL
http://users.cis.fiu.edu/~downeyt/cgs4854/timeout


Wait for idle connection to be closed by the client

Akamai idle timeout of 300s 
DDOS protection with CDN

NLB idle timeout of 350s

#### HTTP Close Header

NLB idle connection, drain, 
1 million TPS load
Connection close header