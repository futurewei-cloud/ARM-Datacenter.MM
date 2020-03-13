---
layout: single
permalink: /qemu/
title: "QEMU"
sidebar:
  nav: "site_map"
classes: wide
---

{::options auto_ids="false" /}

[QEMU](getting_started) emulation of ARM systems is a valuable piece of software infrastructure, which enables the rapid and seamless adoption of ARM based hardware into a data center ecosystem.

Today there are limits on the configuration and scaling of ARM systems within QEMU system emulation.  For example, the number of ARM cores is limited to [123](qemu-arm-about-the-123-core-limit) under a typical QEMU configuration.

Moreover the scaling of QEMU with large numbers of cores limits the practical usable number of cores to about half that 123 core limit in many cases.

Our goal is to enable QEMU emulation of ARM systems.

The areas we will focus on include:
- Optimize scaling of QEMU to support large numbers of cores.
- Enable easier use of aarch64 VMs:
  - Add support for building and using aarch64 VMs for testing QEMU itself.
  - Enable easier change of hardware toplology including core count when building VMs.
  - Allow developers to easily test the impact of modifications on VMs for aarch64 architectures with variable topologies.  
  
We plan to contribute to open source by working with maintainers to upstream all of the above.
  
For more information on our current work, please see our [blog posts on QEMU](../categories/#qemu).

  