---
title: "How to launch ARM aarch64 VM with QEMU from scratch."
date: 2020-03-04T14:16:30-05:00
author: Rob
categories:
  - QEMU
tags:
  - config
classes: wide
---
The below instructions will
To launch an aarch64 VM we first need to install a few dependencies, including QEMU and the qemu-efi-aarch64 package, which includes the efi firmware.

~~~
apt-get install qemu-system-arm
apt-get install qemu-efi-aarch64
~~~
Create the flash images with the correct sizes.

~~~
dd if=/dev/zero of=flash1.img bs=1M count=64
dd if=/dev/zero of=flash0.img bs=1M count=64
dd if=/usr/share/qemu-efi-aarch64/QEMU_EFI.fd of=flash0.img conv=notrunc
~~~

Download the image you want to boot.

For our example we use an Ubuntu installer.

~~~
wget http://ports.ubuntu.com/ubuntu-ports/dists/bionic-updates/main/installer-arm64/current/images/netboot/mini.iso
~~~

Create the empty Ubuntu image file we will install Ubuntu into.

We will use 20 gigabytes for this file.

~~~
qemu-img create ubuntu-image.img 20G
~~~

Start QEMU with the installer.

~~~
qemu-system-aarch64 -nographic -machine virt,gic-version=max -m 512M -cpu max -smp 4 \
-netdev user,id=vnet,hostfwd=:127.0.0.1:0-:22 -device virtio-net-pci,netdev=vnet \
-drive file=ubuntu-image.img,if=none,id=drive0,cache=writeback -device virtio-blk,drive=drive0,bootindex=0 \
-drive file=mini.iso,if=none,id=drive1,cache=writeback -device virtio-blk,drive=drive1,bootindex=1 \
-drive file=flash0.img,format=raw,if=pflash -drive file=flash1.img,format=raw,if=pflash 
~~~

Follow the instructions to install Ubuntu to the ubuntu-image.img file.

Once the install is finished you can exit QEMU with <ctrl>-a x.

Then restart QEMU without the installer image with the following command.

~~~
qemu-system-aarch64 -nographic -machine virt,gic-version=max -m 512M -cpu max -smp 4 \
-netdev user,id=vnet,hostfwd=:127.0.0.1:0-:22 -device virtio-net-pci,netdev=vnet \
-drive file=ubuntu-image.img,if=none,id=drive0,cache=writeback -device virtio-blk,drive=drive0,bootindex=0 \
-drive file=flash0.img,format=raw,if=pflash -drive file=flash1.img,format=raw,if=pflash 
~~~