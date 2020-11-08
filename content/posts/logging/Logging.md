---
title: "Logging"
date: 2020-11-02
categories:
- kubernetes
tags:
- cni
- bpf
hero: /posts/introduction/hero.svg
description: Logging
menu:
  sidebar:
    name: Logging
    identifier: logging
    weight: 15
draft: true
---
 
Do not log every single request, hard to scale, create metric for it 
elasticsearch mapping conflicts
fluentd rate throttling, do not saturate network bandwidth on the node 
an index per team to avoid affecting another and have different log retention
not too many index that can decrease search performance and memory consumption 

<!--more-->