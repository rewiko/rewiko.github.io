---
title: "Load Balancing"
date: 2020-11-02
categories:
- kubernetes
tags:
- loadbalancing
hero: /posts/introduction/hero.svg
description: Load Balancing
menu:
  sidebar:
    name: Load Balancing
    identifier: load-balancing
    weight: 15
draft: true
---

NLB idle connection, drain, 
1 million TPS load
Wait for idle connection to be closed by the client
shutdown-grace-period nginx ingress controller and TernminationGracePeriod
catch SIGTERM and send QUIT signal to nginx
keepalive TTL
Akamai idle timeout of 300s
NLB idle timeout of 350s

NLB send only traffic to the node where nginx is running, preserve ip and ensure to gracefully shutdown pod while the instance is terminated
 externalTrafficPolicy set to Local
https://www.asykim.com/blog/deep-dive-into-kubernetes-external-traffic-policies

https://kubernetes.io/docs/tasks/access-application-cluster/create-external-load-balancer/#preserving-the-client-source-ip

https://stackoverflow.com/questions/60276403/kubernetes-pods-graceful-shutdown-with-tcp-connections-spring-boot

```
At this point the cluster nodes will be reconfigured to remove any rules directing new traffic to the Pod. Any existing TCP connections to the Pod/containers will remain in connection tracking until they are closed (by the client, server or network stack).

For HTTP Keep Alive or HTTP/2 services, the client will continue hitting the same Pod Endpoint until it is told to close the connection (or it is forcibly reset)

```


https://medium.com/flant-com/kubernetes-graceful-shutdown-nginx-php-fpm-d5ab266963c2

Remove TLS termination on the NLB - only 9k TPS
Disable croos load balancing 
https://medium.com/swlh/nlb-connection-resets-109720accfc6
https://www.niels-ole.com/cloud/aws/linux/2020/10/18/nlb-resets.html




IPVS maglev
XDP loadbalancing, direct return 
websocket ingress
DDOS protection with CDN

Gatling, golden signal

<!--more-->