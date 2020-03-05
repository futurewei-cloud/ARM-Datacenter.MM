---
layout: single
permalink: /qemu/getting_started/
title: "QEMU getting started"
classes: wide
---

{::options auto_ids="false" /}

## QEMU Information

QEMU stands for Quick EMUlator.  

It is free and open source software.  QEMU is a FAST! processor emulator using dynamic translation to achieve good emulation speed. [^1]

Main QEMU site: [qemu.org](https://www.qemu.org/)

[Introduction to QEMU](https://www.qemu.org/docs/master/qemu-doc.html#Introduction) is a good comprehensive description of QEMU.

## Installing QEMU

~~~
apt-get install qemu-system-arm
apt-get install qemu-utils
apt-get install qemu-efi-aarch64
~~~

### Installing packages to build QEMU on Ubuntu

~~~
apt-get update
apt-get build-dep -y qemu
apt-get install -y libfdt-dev flex bison
~~~

## QEMU Development
The mailing list qemu-devel is a good place to start.
- [Subscribe to qemu-devel](https://lists.nongnu.org/mailman/listinfo/qemu-devel)
- [Mailing list archives](https://lists.gnu.org/archive/html/qemu-devel/)
- [Other QEMU mailing lists](https://savannah.nongnu.org/mail/?group=qemu)
- [Patchew](http://patchew.org/QEMU/) is where the current patches can be monitored 

[^1]: Reference [qemu.org](https://www.qemu.org/docs/master/qemu-doc.html#Introduction)