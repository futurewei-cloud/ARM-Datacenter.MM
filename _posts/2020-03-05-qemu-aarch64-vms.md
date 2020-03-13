---
title: "QEMU aarch64 ARM VMs"
date: 2020-03-05T07:15:10-05:00
author: Rob
categories:
  - QEMU
tags:
  - config
classes: wide
---
This is a follow-up to a prior post on [how to launch ARM aarch64 VMs from scratch](../how-to-launch-aarch64-vm).

We are working on QEMU enhancements to support aarch64 ARM VMs inside QEMU's vm-build infrastructure.  This is a bit of test infrastructure which allows for building and testing QEMU source code within various flavors of VMs.

The aarch64 VMs are supported for Ubuntu and CentOS VMs.

Although we are working to this upstreamed into mainline QEMU, the current WIP for this project is here: [https://github.com/rf972/qemu/tree/aarch64vm_2.7](https://github.com/rf972/qemu/tree/aarch64vm_2.7)

Note that this support is also being used by the lisa-qemu integration [https://github.com/rf972/lisa-qemu](https://github.com/rf972/lisa-qemu) in order to enable easier testing and debug of varied hardware architectures.

To try this out you can see the available VMs via:

~~~
$ make vm-help

vm-build-ubuntu.aarch64 - Build QEMU in ubuntu aarch64 VM 
vm-build-centos.aarch64 - Build QEMU in CentOS aarch64 VM 
QEMU_CONFIG=/path/conf.yml - Change path to VM configuration .yml file. See config_example.yml for file format details.
~~~
To create the VM and build qemu inside it:

~~~
$ make vm-build-ubuntu.aarch64
~~~

or to make the VM and ssh into it:

~~~
$ make vm-boot-ssh-ubuntu.aarch64
~~~

### Configuration yaml
We also support providing a configuration yaml file. This allows passing specific arguments to qemu to configure the hardware. For example, in our config_example.yaml, we configure a 4 NUMA node topology. To use a specific configuration, just provide the configuration YAML like so:

~~~
$ QEMU_CONFIG=../tests/vm/config_default.yml make vm-boot-ssh-ubuntu.aarch64
~~~