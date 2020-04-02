---
title: "LISA-QEMU Demo"
date: 2020-04-02T07:07:05-04:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Configuration
  - ARM
classes: wide
---
<span style="font-size:60%">This article is a follow-up to an earlier article we wrote [Introducing LISA-QEMU]({{ site.url }}/qemu/lisa-qemu).</span>

LISA-QEMU provides an integration which allows [LISA](https://github.com/ARM-software/lisa) to work with QEMU VMs.  LISA's goal is to help Linux kernel developers to measure the impact of modifications in core parts of the kernel.[^1]  Integration with QEMU will allow developers to test wide variety of hardware configurations including ARM architecture and complex NUMA topologies.

This demo will walk through all the steps needed to build and bring up an aarch64 VM on an x86 platform.  Future articles will work through reconfiguring the hardware for these VMs, inserting a new kernel into these VMs and more !

The first step is to get your linux machine ready to run LISA-QEMU.  In this step we will download all the dependencies needed.  We assume Ubuntu in the below steps.

~~~
apt-get build-dep -y qemu
apt-get install -y libfdt-dev flex bison git apt-utils
apt-get install -y python3-yaml wget qemu-efi-aarch64 qemu-utils genisoimage qemu-user-static
~~~
Now that we have the correct dependencies, let's download the LISA-QEMU code.

~~~
git clone https://github.com/rf972/lisa-qemu.git
cd lisa-qemu
git submodule update --init --recursive
~~~

The next step is to build a new VM.  This build command takes all the defaults.  If you want to learn more about the possible options take a look at build_image.py --help.

~~~
$ time python3 scripts/build_image.py  --help
usage: build_image.py [-h] [--debug] [--dry_run] [--ssh]
                      [--image_type IMAGE_TYPE] [--image_path IMAGE_PATH]
                      [--config CONFIG] [--skip_qemu_build]

Build the qemu VM image for use with lisa.

optional arguments:
  -h, --help            show this help message and exit
  --debug, -D           enable debug output
  --dry_run             for debugging.  Just show commands to issue.
  --ssh                 Launch VM and open an ssh shell.
  --image_type IMAGE_TYPE, -i IMAGE_TYPE
                        Type of image to build.
                        From external/qemu/tests/vm.
                        default is ubuntu.aarch64
  --image_path IMAGE_PATH, -p IMAGE_PATH
                        Allows overriding path to image.
  --config CONFIG, -c CONFIG
                        config file.
                        default is conf/conf_default.yml.
  --skip_qemu_build     For debugging script.

examples:
  To select all defaults:
   scripts/build_image.py
  Or select one or more arguments
    scripts/build_image.py -i ubuntu.aarch64 -c conf/conf_default.yml
~~~

But we digress... Below is the command to build the image.

OK let's build that image...

~~~
python3 scripts/build_image.py
~~~
You will see the progress of the build and other steps of the image creation on your screen.  If you would like to see more comprehensive output and progress, use the <b>--debug</b> option.

Depending on your system this might take many minutes.  Below are some example times.

50 minutes - Intel i7 laptop with 2 cores and 16 GB of memory<BR>
6 minutes - Huawei Taishan 2286 V2 with 128 ARM cores and 512 GB of memory.

Once the image creation is complete, you will see a message like the following.

~~~
Image creation successful.
Image path: /home/lisa-qemu/build/VM-ubuntu.aarch64/ubuntu.aarch64.img
~~~
Now that we have an image, we can test it out by bringing up the image and opening an ssh connection to it.
 
~~~
python3 scripts/launch_image.py
~~~

The time to bring up the VM will vary based on your machine, but it should come up in about 2-3 minutes on most machines.

You should expect to see the following as the system boots and we open an ssh connection to bring us to the guest prompt.

~~~
$ python3 scripts/launch_image.py
Conf:        /home/lisa-qemu/build/VM-ubuntu.aarch64/conf.yml
Image type:  ubuntu.aarch64
Image path:  /home/lisa-qemu/build/VM-ubuntu.aarch64/ubuntu.aarch64.img

qemu@ubuntu-aarch64-guest:~$
~~~

Now that the system is up and running, you could for example, use it for a lisa test.

In our case we issue one command to show that we are in fact an aarch64 architecture with 8 cores.

~~~
qemu@ubuntu-guest:~$ lscpu
Architecture:        aarch64
Byte Order:          Little Endian
CPU(s):              8
On-line CPU(s) list: 0-7
Thread(s) per core:  1
Core(s) per socket:  8
Socket(s):           1
NUMA node(s):        1
Vendor ID:           0x00
Model:               0
Stepping:            0x0
BogoMIPS:            125.00
NUMA node0 CPU(s):   0-7
Flags:               fp asimd evtstrm aes pmull sha1 sha2 crc32 atomics fphp asimdhp cpuid asimdrdm jscvt fcma sha3 sm3 sm4 asimddp sha512 sve asimdfhm flagm
~~~
Once you are done with the VM, you can close the VM simply by typing "exit" a the command prompt.

~~~
qemu@ubuntu-guest:~$ exit
exit
Connection to 127.0.0.1 closed by remote host.
~~~

That's it.  The VM was gracefully powered off.

We hope this article was helpful to understand just how easy it can be to build and launch a VM with LISA-QEMU !

[^1]: This definition can be found on the [LISA github page](https://github.com/ARM-software/lisa)

