---
title: "Kubernetes Networking"
date: 2020-11-02
categories:
- kubernetes
tags:
- cni
- bpf
hero: /posts/introduction/hero.svg
description: Kubernetes Networking
menu:
  sidebar:
    name: Kubernetes Networking
    identifier: kubernetes-networking
    weight: 15
draft: true
---

Network CNI migration from flannel to aws CNI

Weave 
Calico 
Cilium:
  - CRDs gotcha update status of each CRD was DDOSing etcd
  - restrict the number of identities by defining the labels parameters is key to scale 
  - cluster entities can be quite heavy it will udpate bpf map
  - cilium architecture BPF -> BPF template/program and BPF maps  
Native Cloud routing: AWS CNI
Ip space issue using 100.x.x.x ips, ip space with nodes, pods 

<!--more-->