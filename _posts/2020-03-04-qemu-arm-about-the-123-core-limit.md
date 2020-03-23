---
title: "QEMU ARM limit of 123 cores"
date: 2020-03-04T14:16:30-05:00
author: Rob
categories:
  - QEMU
tags:
  - config
  - ARM
classes: wide
---

<b>Why does QEMU ARM emulation limit the number of cores to 123?</b>

When running a [QEMU](https://www.qemu.org/) ARM emulation of a system, you might start your QEMU session with something like this[^1].

~~~
$ qemu-system-aarch64 -machine virt,gic-version=max -m 4G -cpu cortex-a57 -smp 16
~~~
See below for a more detailed breakdown of these arguments.

Above we chose -smp 16 to choose 16 cores.  Theoretically we can choose any number up to 123 cores.

<b>Where does the 123 number derive from?</b>  

Under the virt machine QEMU creates a memory map that reserves only a certain amount of space for the GIC redistributors, which also limits the number of CPUs to 123.  The reason 123 was chosen seems to be to simply align the next region on a nice multiple 0x09000000.

If we breakdown the above command line argument by argument:

<b>-machine virt,gic-version=max</b><br>
This particular machine is a virtual representation of an ARM system.
We are using ARM GIC (Generic Interrupt Controller) set to the max supported by this configuration.  The reason we need this is because virt without GIC=max only supports up to 8 cpus.

<b>-cpu cortex-a57</b><br>
This means to emulate the cortex-a57

<b>-smp 16</b><br>
The number of cores is argument to -smp, which we set to 16 above.

[^1]: It is worth mentioning we omitted the disks from the command line for clarity.  <br>There is another post on [starting aarch64 VMs](../how-to-launch-aarch64-vm), which describes these additional details.
