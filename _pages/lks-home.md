---
layout: single
permalink: /lks/home/
title: "Linux Kernel Scheduler"
sidebar:
  nav: "site_map"
author_profile: false
---

{::options auto_ids="false" /}

Over the past several years, ARM processors have come to dominate the smartphone industry. 
They offer very competitive performance-per-watt ratio and allow vendors to buy the necessary 
building blocks and design their own processors based on ARMv8 architecture.

ARM server processors typically have a lot more physical cores than x86 server processors.
L1, L2 and LLC cache as well as NUMA topology is also significantly different from x86.
This creates a set of unique challenges for Linux Kernel to maintain optimal performance on 
ARM servers with hundreds of cores and complex NUMA topologies.

In category **Linux Kernel Scheduler** we will cover following topics (**Tags**)
 - NUMA balancer placement decisions (**NUMA**)
 - Scheduler load balancer (**CFS**) placement decisions
 - Excessive task migrations due to large core count 
 - Handling of latency sensitive tasks
 - Detection of system overload with lot of task migrations



