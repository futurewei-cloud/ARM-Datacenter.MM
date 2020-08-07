---
title: "How to setup NVMe/TCP with NVME-oF using KVM and QEMU"
date: 2020-08-07T07:52:34-04:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Debugging
  - ARM
classes: wide
---

In this post we will explain how to connect two QEMU Guests (using KVM) with NVMe over Fabrics using TCP as the transport.

We will show how to connect an NVMe target to an NVMe initiator, using the NVMe/TCP transport.  It is worth mentioning before we get started that we will use the terms of "target" to describe the guest which exports the target, and "initiator" to describe the guest which connects to the target.

The target QEMU guest will export a simulated NVME drive which we will create from an image file.  The initiator guest will connect to the target and will be able to access this NVME drive.

Note that this configuration is largely an example to be used for evaluation and/or development with NVME-of.  This setup described is not intended to be used for a production environment.

First Step: Create a Guest
--------------------------
Before we can get started, we need to bring up our QEMU guests and get them sharing the same network.

Fortunately, we described in an earlier post [how to setup a shared network for two QEMU guests]({{ site.url }}/qemu/network-aarch64-qemu-guests). That's a good place to start.  

We also have other posts for getting an aarch64 VM up and running, including:
- [setting up an aarch64 QEMU guest from scratch]({{ site.url }}/qemu/how-to-launch-aarch64-vm)
- [using QEMU's vm-build automation to create aarch64 guest]({{ site.url }}/qemu/qemu-aarch64-vms)
- [using LISA-QEMU to create and launch an aarch64 guest]({{ site.url }}/qemu/lisa-qemu-demo1)

Kernel Configuration
--------------------------------
Before we get started we will make sure that the guest's kernel has all the right modules built in. 

The guest's kernel config should have these modules.
~~~
$ cat /boot/config-`uname -r` | grep NVME

# NVME Support
CONFIG_NVME_CORE=m
CONFIG_BLK_DEV_NVME=m
# CONFIG_NVME_MULTIPATH is not set
# CONFIG_NVME_HWMON is not set
CONFIG_NVME_FABRICS=m
CONFIG_NVME_FC=m
CONFIG_NVME_TCP=m
CONFIG_NVME_TARGET=m
CONFIG_NVME_TARGET_LOOP=m
CONFIG_NVME_TARGET_FC=m
# CONFIG_NVME_TARGET_FCLOOP is not set
CONFIG_NVME_TARGET_TCP=m
# end of NVME Support
~~~

nvme-cli
-------------

Make sure the nvme-cli is installed on the guests.

~~~
sudo apt-get install nvme-cli
~~~

Initiator Guest Setup
---------------------

This is the QEMU command we use to bring up the initiator QEMU guest.

~~~
sudo qemu-system-aarch64 -nographic -machine virt,gic-version=max -m 8G -cpu max   \
       -drive file=./ubuntu20-a.img,if=none,id=drive0,cache=writeback              \
       -device virtio-blk,drive=drive0,bootindex=0                                 \
       -drive file=./flash0-a.img,format=raw,if=pflash                             \
       -drive file=./flash1-a.img,format=raw,if=pflash                             \
       -smp 4 -accel kvm -netdev bridge,id=hn1                                     \
       -device virtio-net,netdev=hn1,mac=e6:c8:ff:09:76:99
~~~

Target Guest Setup
---------------------

When you bring up the target's QEMU guest, be sure to include an NVME disk.

We can create the disk with the below.

~~~
qemu-img create -f qcow2 nvme.img 10G
~~~

When we bring up QEMU, add this set of options so that the guest sees the NVME disk.

~~~
-drive file=./nvme.img,if=none,id=nvme0 -device nvme,drive=nvme0,serial=1234
~~~

This is the QEMU command we use to bring up the target QEMU guest.

Note how we added in the options for the NVMe device.

~~~
sudo qemu-system-aarch64 -nographic -machine virt,gic-version=max -m 8G -cpu max \
       -drive file=./ubuntu20-b.img,if=none,id=drive0,cache=writeback              \ 
       -device virtio-blk,drive=drive0,bootindex=0                                 \ 
       -drive file=./flash0-b.img,format=raw,if=pflash                             \
       -drive file=./flash1-b.img,format=raw,if=pflash                             \
       -smp 4 -accel kvm -netdev bridge,id=hn1                                     \ 
       -device virtio-net,netdev=hn1,mac=e6:c8:ff:09:76:9c                         \ 
       -drive file=./nvme.img,if=none,id=nvme0 -device nvme,drive=nvme0,serial=1234 
~~~

Configure Target
------------------
Load the following modules on the target:

~~~
sudo modprobe nvmet
sudo modprobe nvmet-tcp
~~~

Next, create and configure an NVMe Target subsystem.
This includes creating a namespace.

~~~
cd /sys/kernel/config/nvmet/subsystems
sudo mkdir nvme-test-target
cd nvme-test-target/
echo 1 | sudo tee -a attr_allow_any_host > /dev/null
sudo mkdir namespaces/1
cd namespaces/1
~~~

Before we can attach our NVMe device to this target, we need to find the name.

~~~
sudo nvme list

Node             SN     Model            Namespace Usage                      Format           FW Rev          
---------------- ------ ---------------- --------- -------------------------- ---------------- --------
/dev/nvme0n1     1234   QEMU NVMe Ctrl   1          10.74  GB /  10.74  GB    512   B +  0 B   1.0      
~~~

The next step attaches our NVMe device /dev/nvme0n1 to this target and enables it.

~~~
echo -n /dev/nvme0n1 |sudo tee -a device_path > /dev/null
echo 1|sudo tee -a enable > /dev/null
~~~

Next we will create an NVMe target port, and configure the IP address and other parameters.

~~~
sudo mkdir /sys/kernel/config/nvmet/ports/1
cd /sys/kernel/config/nvmet/ports/1

echo 192.168.0.16 |sudo tee -a addr_traddr > /dev/null

echo tcp|sudo tee -a addr_trtype > /dev/null
echo 4420|sudo tee -a addr_trsvcid > /dev/null
echo ipv4|sudo tee -a addr_adrfam > /dev/null
~~~

The final step creates a link to the subsystem from the port.
~~~
sudo ln -s /sys/kernel/config/nvmet/subsystems/nvme-test-target/ /sys/kernel/config/nvmet/ports/1/subsystems/nvme-test-target
~~~

At this point we should see a message in the dmesg log

~~~
dmesg |grep "nvmet_tcp"

[81528.143604] nvmet_tcp: enabling port 1 (192.168.0.16:4420)
~~~

Mount Target on Initiator
-------------------------
Load the following modules on the initiator:

~~~
sudo modprobe nvme
sudo modprobe nvme-tcp
~~~

Next, check that we currently do not see any NVMe devices.  The output of the following command should be blank.

~~~
sudo nvme list
~~~

Next, we will attempt to discover the remote target.

When we initially tried the "discover" command we got an error that our hostnqn was needed.  In our example below you will notice that we are providing a hostnqn.

~~~
sudo nvme discover -t tcp -a 192.168.0.16 -s 4420 --hostnqn=nqn.2014-08.org.nvmexpress:uuid:1b4e28ba-2fa1-11d2-883f-0016d3ccabcd

Discovery Log Number of Records 1, Generation counter 2
=====Discovery Log Entry 0======
trtype:  tcp
adrfam:  ipv4
subtype: nvme subsystem
treq:    not specified, sq flow control disable supported
portid:  1
trsvcid: 4420
subnqn:  nvme-test-target
traddr:  192.168.0.16
sectype: none
~~~

Using the subnqn as the -n argument, we will connect to the discovered target.

~~~
sudo nvme connect -t tcp -n nvme-test-target -a 192.168.0.16 -s 4420 --hostnqn=nqn.2014-08.org.nvmexpress:uuid:1b4e28ba-2fa1-11d2-883f-0016d3ccabcd
~~~
Success.  We can immediately check the nvme list for the attached device.

~~~
sudo nvme list
Node             SN               Model  Namespace Usage                      Format           FW Rev  
---------------- ---------------- ------ --------- -------------------------- ---------------- --------
/dev/nvme0n1     84cfc88e9ba4a8f4 Linux  1          10.74  GB /  10.74  GB    512   B +  0 B   5.8.0-rc
~~~

To detach the target, run the following command on the initiator.

~~~
sudo nvme disconnect /dev/nvme0n1 -n nvme-test-target
~~~


References:
- [how to connect two aarch64 QEMU guests using a bridged network]({{ site.url }}/qemu/network-aarch64-qemu-guests)
- [how to launch ARM aarch64 VMs from scratch]({{ site.url }}/qemu/how-to-launch-aarch64-vm)
- [using LISA-QEMU to create and launch an aarch64 guest]({{ site.url }}/qemu/lisa-qemu-demo1)
- [NVMe over Fabrics Using TCP](https://www.linuxjournal.com/content/data-flash-part-iii-nvme-over-fabrics-using-tcp)