---
title: "Metric"
date: 2020-11-02
categories:
- kubernetes
tags:
- cni
- bpf
hero: /posts/introduction/hero.svg
description: Metric
menu:
  sidebar:
    name: Metric
    identifier: metric
    weight: 15
draft: true
---

Cardinality is key - https://www.robustperception.io/cardinality-is-key

<!--more-->

https://chrismarchbanks.com/assets/talks/containing-your-cardinality.pdf

## Tip #3: Use multiple metrics
All the labels on one metric

### http_request_duration_seconds_bucket
100 instances *
10 buckets *
10 response codes *
10 routes = 100,000 series

### Split the metric

http_request_duration_seconds_bucket
100 instances * 10 buckets * 10 routes = 10,000
series
http_requests_total
100 instances * 10 response codes * 10 routes =
10,000 series
Total: 20,000 series

### Prometheus v2.14.0 releases cardinality
stats in the UI!
Useful queries:
1. sum(scrape_series_added) by (job)
2. sum(scrape_samples_scraped) by (job)
3. prometheus_tsdb_symbol_table_size_bytes

Sharding prometheus is common way to scale prometheus, but you won't be able to query metrics located in 2 differents proemtheus you will need to federate metric to get those informations.

Thanos allow to shard your prometheus while storing your metrics into the same databases, allow deduplication and long term storage with downsampling. 
Cortex 
