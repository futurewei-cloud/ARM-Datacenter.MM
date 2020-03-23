---
title: "How to configure NUMA nodes with QEMU"
date: 2020-03-04T15:32:10-05:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Configuration
classes: wide  
---

QEMU does allow very flexible configuration of NUMA node topology.

When starting QEMU we can select a NUMA topology with the -numa argument.

In its most basic form, we can specify the CPUs assigned to each NUMA node.

~~~
-smp cpus=16 -numa node,cpus=0-3,nodeid=0 \
-numa node,cpus=4-7,nodeid=1 \
-numa node,cpus=8-11,nodeid=2 \
-numa node,cpus=12-15,nodeid=3 \
~~~
This gives us a system that looks like the following from lscpu.  

Note below that we specified 4 NUMA nodes with each NUMA node containing 4 CPUs.
~~~
qemu@ubuntu:~$ lscpu
Architecture:        aarch64
Byte Order:          Little Endian
CPU(s):              16
On-line CPU(s) list: 0-15
Thread(s) per core:  1
Core(s) per socket:  8
Socket(s):           2
NUMA node(s):        4
Vendor ID:           ARM
Model:               0
Model name:          Cortex-A57
Stepping:            r1p0
BogoMIPS:            125.00
NUMA node0 CPU(s):   0-3
NUMA node1 CPU(s):   4-7
NUMA node2 CPU(s):   8-11
NUMA node3 CPU(s):   12-15
Flags:               fp asimd evtstrm aes pmull sha1 sha2 crc32 cpuid
~~~

To go one step further we can also specify the NUMA distance between each CPU.

Suppose we have the following NUMA topology.

~~~ 
 ________      ________
|        |    |        |
| Node 0 |    | Node 1 |
|        |-15-|        |
|________|    |________|
     |    \  /     |
    20     20     20
 ____|___ /  \ ____|___
|        |/   |        |
| Node 2 |    | Node 3 |
|        |-15-|        |
|________|    |________|
~~~
We can use the -numa dist option to add on the specific NUMA distances.

~~~
-smp cpus=16 -numa node,cpus=0-3,nodeid=0 \
-numa node,cpus=4-7,nodeid=1 \
-numa node,cpus=8-11,nodeid=2 \
-numa node,cpus=12-15,nodeid=3 \
-numa dist,src=0,dst=1,val=15 \
-numa dist,src=2,dst=3,val=15 \
-numa dist,src=0,dst=2,val=20 \
-numa dist,src=0,dst=3,val=20 \
-numa dist,src=1,dst=2,val=20 \
-numa dist,src=1,dst=3,val=20
~~~

The numactl command will confirm the distances that we configured in QEMU.

~~~
qemu@ubuntu:~$ numactl --hardware
available: 4 nodes (0-3)
node 0 cpus: 0 1 2 3
node 0 size: 983 MB
node 0 free: 852 MB
node 1 cpus: 4 5 6 7
node 1 size: 1007 MB
node 1 free: 923 MB
node 2 cpus: 8 9 10 11
node 2 size: 943 MB
node 2 free: 812 MB
node 3 cpus: 12 13 14 15
node 3 size: 1007 MB
node 3 free: 916 MB
node distances:
node   0   1   2   3 
  0:  10  15  20  20 
  1:  15  10  20  20 
  2:  20  20  10  15 
  3:  20  20  15  10 

~~~
