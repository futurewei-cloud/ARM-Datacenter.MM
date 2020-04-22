---
title: "How to debug kernel using QEMU and aarch64 VM."
date: 2020-04-22T06:51:34-04:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Debugging
  - ARM
classes: wide
---

QEMU is a great tool to use when needing to debug the kernel.  
There are many recipes online for this too, I have listed a few helpful ones at the end of the article for reference.  

We would like to share our steps for debug the kernel, but focused on aarch64 systems, as some of the steps might be slightly different for this type of system.

First, create a directory to work in and run these commands to create the flash images:

~~~
dd if=/dev/zero of=flash1.img bs=1M count=64
dd if=/dev/zero of=flash0.img bs=1M count=64
dd if=/usr/share/qemu-efi-aarch64/QEMU_EFI.fd of=flash0.img conv=notrunc
~~~

Next, download a QEMU image.  We will use an ubuntu image that we previously created.

We should mention that our procedure involves building our own kernel from scratch, and feeding this image to QEMU.

Thus the first step is to actually create a QEMU image.  We will assume you already have an image to use.  If not, check out our articles on:
- [how to create a VM using LISA-QEMU]({{ site.url }}/qemu/lisa-qemu-demo1).
- [how to create aarch64 VM using QEMU vm-build]({{ site.url }}/qemu/how-to-launch-aarch64-vm). 
- [how to create an aarch64 VM from scratch]({{ site.url }}/qemu/qemu-aarch64-vms).  

We prefer the first procedure using LISA-QEMU since we also have a helpful script to install your kernel into the VM image automatically.

But don't worry, if you want to take a different route we will show all the steps for that too!

Installing Kernel
===================

You have a few options here.  One is to boot the image and install the image manually or use LISA-QEMU scripts to install it.  The below command will boot the image in case you want to use the later manual approach to boot the image, scp in the kernel (maybe a .deb file) and install it manually with deb -i <kernel>.deb.

~~~
qemu/build/aarch64-softmmu/qemu-system-aarch64 -nographic\
                    -machine virt,gic-version=max -m 2G -cpu max\
                    -netdev user,id=vnet,hostfwd=:127.0.0.1:0-:22\
                    -device virtio-net-pci,netdev=vnet\ 
                    -drive file=./mini_ubuntu.img,if=none,id=drive0,cache=writeback\ 
                    -device virtio-blk,drive=drive0,bootindex=0\ 
                    -drive file=./flash0.img,format=raw,if=pflash \
                    -drive file=./flash1.img,format=raw,if=pflash -smp 4 
~~~

To bring up QEMU with a kernel, typically you will need a kernel image (that you built), an initrd image (built after installing the kernel in your image), and the OS image (created above).

Keep in mind the below steps assume a raw image.  If you have a qcow2, then use qemu-img to convert it to raw first.
For example:

~~~
qemu-img convert -O raw my_image.qcow2 my_image_output.raw
~~~
Below is how to mount an image to copy out files.  You need to copy out the initrd in this case.

~~~
$ mkdir mnt
$ sudo losetup -f -P mini_ubuntu.img
$ sudo losetup -l
NAME       SIZELIMIT OFFSET AUTOCLEAR RO BACK-FILE                                DIO LOG-SEC
/dev/loop0         0      0         0  0 /home/rob/qemu/for_peter/mini_ubuntu.img   0     512
$ sudo mount /dev/loop0 ./mnt
mount: /home/rob/qemu/for_peter/mnt: wrong fs type, bad option, bad superblock on /dev/loop0, missing codepage or helper program, or other error.
rob@ARMv2-1:~/qemu/for_peter$ sudo mount /dev/loop0p2 ./mnt
rob@ARMv2-1:~/qemu/for_peter$ ls ./mnt/boot
config-4.15.0-88-generic  grub                          initrd.img-5.5.11             System.map-5.5.11          vmlinuz-5.5.11
config-5.5.11             initrd.img                    initrd.img.old                vmlinuz                    vmlinuz.old
efi                       initrd.img-4.15.0-88-generic  System.map-4.15.0-88-generic  vmlinuz-4.15.0-88-generic
$ cp ./mnt/initrd.img-5.5.11 .
$ sudo umount ./mnt
$ sudo losetup -d /dev/loop0
~~~

Next, boot the kernel you built with your initrd.  Note the kernel you built can be found at
arch/arm64/boot/Image.

This command line will bring up your kernel image with your initrd and your OS Image.

One item you might need to customize is the "root=/dev/vda1" argument.  This tells the kernel where to find your boot partition. This might vary depending on your VM image.

~~~
qemu/build/aarch64-softmmu/qemu-system-aarch64 -nographic\
                  -machine virt,gic-version=max -m 2G -cpu max\
                  -netdev user,id=vnet,hostfwd=:127.0.0.1:0-:22\
                  -device virtio-net-pci,netdev=vnet\
                  -drive file=./mini_ubuntu.img,if=none,id=drive0,cache=writeback\
                  -device virtio-blk,drive=drive0,bootindex=0\
                  -drive file=./flash0.img,format=raw,if=pflash\
                  -drive file=./flash1.img,format=raw,if=pflash -smp 4\
                  -kernel ./linux/arch/arm64/boot/Image\
                  -append "root=/dev/vda2 nokaslr console=ttyAMA0"\
                  -initrd ./initrd.img-5.5.11 -s -S
~~~

<B>-s</B> tells QEMU to use the TCP port :1234<BR>
<b>-S</b> will pause at startup, waiting for the debugger to attach.

Before we get started debugging, update your ~/.gdbinit with the following:

~~~
add-auto-load-safe-path linux-5.5.11/scripts/gdb/vmlinux-gdb.py
~~~

In another window, start the debugger.
Note, if you are on a x86 host debugging aarch64, then you need to use gdb-multiarch (sudo apt-get gdb-multiarch). In our case below we are on an aarch64 host, so we just use gdb.

It's very important to note that we receive the "done" message below indicating symbols were loaded successfully, otherwise the following steps will not work.

~~~
$ gdb linux-5.5.11/vmlinux
GNU gdb (Ubuntu 8.1-0ubuntu3.2) 8.1.0.20180409-git
Reading symbols from linux-5.5.11/vmlinux...done.
~~~
Attach the debugger to the kernel.  Remember the -s argument above? It told QEMU to use port :1234.  We will connect to it now.

~~~
(gdb) target remote localhost:1234
Remote debugging using localhost:1234
0x0000000000000000 in ?? ()
~~~
That it.  The debugger is connected. 

Now let's test out the setup. <BR>
Add a breakpoint in the kernel as a test.

~~~
(gdb) hbreak start_kernel
Hardware assisted breakpoint 1 at 0xffff800011330cdc: file init/main.c, line 577.
(gdb) c
Continuing.

Thread 1 hit Breakpoint 1, start_kernel () at init/main.c:577
577 {
(gdb) l
572 {
573     rest_init();
574 }
575 
576 asmlinkage __visible void __init start_kernel(void)
577 {
578     char *command_line;
579     char *after_dashes;
580 
581     set_task_stack_end_magic(&init_task);
(gdb) 
~~~

We hit the breakpoint !

Remember above that we used the -S option to QEMU?  This told QEMU to wait to start running the image until we connected the debugger.  Thus once we hit continue, QEMU actually starts booting the kernel.

References:
- [debugging-linux-kernel-with-gdb-and-qemu](https://yulistic.gitlab.io/2018/12/debugging-linux-kernel-with-gdb-and-qemu/)
- [booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb](http://nickdesaulniers.github.io/blog/2018/10/24/booting-a-custom-linux-kernel-in-qemu-and-debugging-it-with-gdb/)