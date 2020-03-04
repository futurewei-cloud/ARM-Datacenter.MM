---
layout: single
permalink: /qemu/getting_started/
title: "QEMU getting started"
classes: wide
---

{::options auto_ids="false" /}

## Getting started  

Installing QEMU

~~~
apt-get install qemu-system-arm
apt-get install qemu-utils
apt-get install qemu-efi-aarch64
~~~

Installing packages to build QEMU on Ubuntu

~~~
apt-get update
apt-get build-dep -y qemu
apt-get install -y libfdt-dev flex bison
~~~
The mailing list qemu-devel is a good place to start.
- [Subscribe to qemu-devel](https://lists.nongnu.org/mailman/listinfo/qemu-devel)
- [Mailing list archives](https://lists.gnu.org/archive/html/qemu-devel/)
- [Other QEMU mailing lists](https://savannah.nongnu.org/mail/?group=qemu)
- [Patchew](http://patchew.org/QEMU/) is where the current patches can be monitored 