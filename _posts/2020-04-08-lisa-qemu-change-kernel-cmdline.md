---
title: "How to easily install the kernel in a VM"
date: 2020-04-08T07:50:07-04:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Configuration
  - ARM
classes: wide
---
<span style="font-size:60%">This article is a follow-up to an earlier article we wrote [Introducing LISA-QEMU]({{ site.url }}/qemu/lisa-qemu).</span>

This article will outline the steps to install a kernel into a VM using some scripts we developed.  In our case we have an x86_64 host and a aarch64 VM.

We will assume you have cloned the [LISA-QEMU](https://github.com/rf972/lisa-qemu) repository already.  As part of the LISA-QEMU integration we have added a script to automate the process of installing a kernel into a VM.  The scripts we talk about below can be found in the [LISA-QEMU github](https://github.com/rf972/lisa-qemu)

~~~
git clone https://github.com/rf972/lisa-qemu.git
cd lisa-qemu
git submodule update --init --recursive
~~~

We also assume you have built the kernel .deb install package.  We covered the detailed steps in our [README](https://github.com/rf972/lisa-qemu/blob/master/README.md).  You can also find needed dependencies for this article at that link.

You can use install_kernel.py to generate a new image with the kernel of choice installed.<BR>
Assuming you have the VM image that was created similar to the steps in [this post]({{ site.url }}/qemu/lisa-qemu-demo1), just launch a command like the below to install your kernel.

~~~
$ sudo python3 scripts/install_kernel.py --kernel_pkg ../linux/linux-image-5.5.11_5.5.11-1_arm64.deb 
scripts/install_kernel.py: image: build/VM-ubuntu.aarch64/ubuntu.aarch64.img
scripts/install_kernel.py: kernel_pkg: ../linux/linux-image-5.5.11_5.5.11-1_arm64.deb

Install kernel successful.
Image path: /home/rob/qemu/lisa-qemu/build/VM-ubuntu.aarch64/ubuntu.aarch64.img.kernel-5.5.11-1

To start this image run this command:
python3 /home/rob/qemu/lisa-qemu/scripts/launch_image.py -p /home/rob/qemu/lisa-qemu/build/VM-ubuntu.aarch64/ubuntu.aarch64.img.kernel-5.5.11-1
~~~
We need to use sudo for these commands since sudo is required as part of mounting images.

Note that the argument is:<BR>
<b>-p or --kernel_pkg</b> argument with the .deb kernel package

Also note that the last lines in the output show the command to issue to bring this image up.

~~~
To start this image run this command:
python3 /home/rob/qemu/lisa-qemu/scripts/launch_image.py -p /home/rob/qemu/lisa-qemu/build/VM-ubuntu.aarch64/ubuntu.aarch64.img.kernel-5.5.11-1
~~~

You might wonder where we got the VM image from?<BR>
It was found in a default location after running our build_image.py script.  See [this post]({{ site.url }}/qemu/lisa-qemu-demo1) for more details.<BR>

If you want to supply your own image, we have an argument for that. :)<BR>
<b>--image</b> argument with the VM image to start from.  

When supplying the image, the command line might look like the below.

~~~
sudo python3 scripts/install_kernel.py --kernel_pkg ../linux/linux-image-5.5.11_5.5.11-1_arm64.deb --image build/VM-ubuntu.aarch64/ubuntu.aarch64.img
~~~
There are a few options for installing the kernel.

By default install_kernel.py will attempt to install your kernel using a chroot environment.  This is done for speed more than anything else since in our case is is faster to use the chroot than to bring up the aarch64 emulated VM and install the kernel. 

We also support the <b>--vm</b> option which will bring up the VM with QEMU and then install the kernel into it.  If you run into issues with the chroot environment install this would be a good alternative.

An example of the VM install method.

~~~
sudo python3 scripts/install_kernel.py --vm --kernel_pkg ../linux/linux-image-5.5.11_5.5.11-1_arm64.deb
~~~ 
<BR>
Thanks for taking the time to learn more about our work on LISA-QEMU !

