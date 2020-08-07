---
title: "How to connect two aarch64 QEMU guests with a bridge"
date: 2020-08-06T15:52:34-04:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Debugging
  - ARM
classes: wide
---

In this post we will show how to share a network between two QEMU guests using a bridge.

There are many possible uses of this kind of a setup.  One of them is for integration
testing of for example, target and initiator code using one guest for the initiator and another
for the target.

This post creates a bridge on the host, which the guests both share.

Create Bridge for Shared Network
--------------------------------
We will first create the bridge and give it a "local" address, since for now
we are not planning on exporting this network off the host.  You will also notice
we add an IP address for the host on this network 192.168.0.1

~~~
sudo ip link add br0 type bridge
sudo ip addr add 192.168.0.1/24 dev br0
sudo ip link set br0 up
~~~
Now we can check that the bridge exists and is ready (state is UP).

~~~
$ ip addr

6: br0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether fe:06:bb:4c:37:a1 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.1/24 scope global br0
       valid_lft forever preferred_lft forever
    inet6 fe80::8c39:6fff:fe23:ca06/64 scope link 
       valid_lft forever preferred_lft forever
~~~

Next we will tell QEMU about this bridge by adding it to the QEMU bridge configuration file.

~~~
$ echo 'allow br0' | sudo tee -a /etc/qemu/bridge.conf
~~~

Starting the guests
------------------

When we bring up the QEMU guests, we will provide the -netdev option to specify a bridge that
our guests will use for their network.  Below is an example of these network options.

~~~
-netdev bridge,id=hn1 -device virtio-net,netdev=hn1,mac=e6:c8:ff:09:76:99
~~~

Here are the full set of options to bring up our aarch64 guests.

Note that we specify a different MAC address for each guest.

~~~
guest A:

$ sudo qemu-system-aarch64 -nographic -machine virt,gic-version=max -m 8G -cpu max \
       -drive file=./ubuntu20-a.img,if=none,id=drive0,cache=writeback              \
       -device virtio-blk,drive=drive0,bootindex=0                                 \
       -drive file=./flash0-a.img,format=raw,if=pflash                             \
       -drive file=./flash1-a.img,format=raw,if=pflash                             \
       -smp 4 -accel kvm -netdev bridge,id=hn1                                     \
       -device virtio-net,netdev=hn1,mac=e6:c8:ff:09:76:99

guest B:

$ sudo qemu-system-aarch64 -nographic -machine virt,gic-version=max -m 8G -cpu max \
       -drive file=./ubuntu20-b.img,if=none,id=drive0,cache=writeback              \ 
       -device virtio-blk,drive=drive0,bootindex=0                                 \ 
       -drive file=./flash0-b.img,format=raw,if=pflash                             \
       -drive file=./flash1-b.img,format=raw,if=pflash                             \
       -smp 4 -accel kvm -netdev bridge,id=hn1                                     \ 
       -device virtio-net,netdev=hn1,mac=e6:c8:ff:09:76:9c
~~~

Once the guests are up, you can configure the IP addresses quickly for both
guests via the below commands.  

Note that we chose IP addresses of 192.168.0.8 for guest a and .16 for guest b.

~~~
$ ip addr

2: enp0s3: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN group default qlen 1000
    link/ether e6:c8:ff:09:76:99 brd ff:ff:ff:ff:ff:ff

$ sudo ip addr add 192.168.0.8/24 dev enp0s3
$ sudo ip link set enp0s3 up
~~~

Testing the Shared Network
--------------------------

To test that the guest's network is up, check ip addr again.  It should show "state UP".

~~~
$ ip addr

2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether e6:c8:ff:09:76:99 brd ff:ff:ff:ff:ff:ff
    inet 192.168.0.8/24 scope global enp0s3
       valid_lft forever preferred_lft forever
    inet6 fe80::e4c8:ffff:fe09:7699/64 scope link 
       valid_lft forever preferred_lft forever
~~~

Then we can try pinging the host.

~~~
$ ping 192.168.0.1
PING 192.168.0.1 (192.168.0.1) 56(84) bytes of data.
64 bytes from 192.168.0.1: icmp_seq=1 ttl=64 time=0.193 ms
~~~

It works !  Now let's try pinging the other guest.

~~~
$ ping 192.168.0.16
PING 192.168.0.16 (192.168.0.16) 56(84) bytes of data.
64 bytes from 192.168.0.16: icmp_seq=1 ttl=64 time=0.358 ms
~~~

Also worked !  The guests can now see each other and are sharing the same network.

Access to Network Beyond Host
-----------------------------

Suppose that we wanted to get access to the external network also.
This can be added by simply adding a device to the bridge.
In our case the device is: enahisic2i0, and we add it to bridge br0

~~~
sudo ip link set enahisic2i0 master br0
~~~

After that, we just add public IP addresses on our bridge.
You might need to remove it from the device before adding it to the bridge.

~~~
sudo ip addr del 1.234.55.67/24 dev enahisic2i0
sudo ip addr add 1.234.55.67/24 dev br0
~~~

Finally, inside the guests, give them public IP addreses as well:

~~~
sudo ip addr add 1.234.55.65/24 dev enp0s3
~~~

Note that to access beyond your local subnet, you might need to add a default route:

~~~
sudo ip route add default via 1.234.55.1 dev enops
~~~

References:
- [Bridging two QEmu guests](http://www.kaizou.org/2018/06/qemu-bridge.html)
- [QEMU Networking documentation](https://wiki.qemu.org/Documentation/Networking)