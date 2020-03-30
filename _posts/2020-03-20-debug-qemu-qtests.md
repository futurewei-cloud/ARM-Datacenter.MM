---
title: "Testing QEMU emulation: how to debug QTest with gdb"
date: 2020-03-20T16:28:58-04:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Debugging
classes: wide
---

[QEMU](https://www.qemu.org/) has several different kinds of tests for exercising different aspects of the code. 
A page [here](https://wiki.qemu.org/Testing) has good details on these tests.  Another very useful doument is [testing.rst](https://github.com/qemu/qemu/blob/master/docs/devel/testing.rst). Within this document it states that "QTest is a device emulation testing framework. It can be very useful to test device models; it could also control certain aspects of QEMU (such as virtual clock stepping), with a special purpose "qtest" protocol."

We can run the QTests as part of either make check or make check-qtest.

If we run the QTests with the below command, it shows us more information about the commands it is executing.

~~~
make check-qtest V=1
~~~

For example, you might see something like this displayed.

~~~
$ QTEST_QEMU_BINARY=aarch64-softmmu/qemu-system-aarch64 QTEST_QEMU_IMG=qemu-img tests/qtest/tpm-tis-device-test -m=quick -k --tap < /dev/null | ./scripts/tap-driver.pl --test-name="tpm-tis-device-test"
PASS 1 tpm-tis-device-test /aarch64/tpm-tis/test_check_localities
PASS 2 tpm-tis-device-test /aarch64/tpm-tis/test_check_access_reg
PASS 3 tpm-tis-device-test /aarch64/tpm-tis/test_check_access_reg_seize
PASS 4 tpm-tis-device-test /aarch64/tpm-tis/test_check_access_reg_release
PASS 5 tpm-tis-device-test /aarch64/tpm-tis/test_check_transmit
~~~

Let's break this down a bit.<BR>
<b>QTEST_QEMU_BINARY</b> - This is the command that gets issued when starting QEMU.  This is useful since we can add onto it if we would like to run QEMU within another command (like a debugger).<BR>
<b>tests/qtest/tpm-tis-device-test -m=quick -k --tap</b> - This is the actual QTest which will get executed.  Inside this test it will launch QEMU.<BR>

Another useful option is: 

~~~
--verbose
~~~

Once you add that to any qtest command, it would look something like this with more information displayed.

~~~
$ QTEST_QEMU_BINARY=aarch64-softmmu/qemu-system-aarch64 QTEST_QEMU_IMG=qemu-img tests/qtest/bios-tables-test -m=quick -k --tap < /dev/null | ./scripts/tap-driver.pl --test-name="bios-tables-test"  --show-failures-only --verbose
   random seed: R02S0d429b0279b778325d7c631f360b375b
   Start of aarch64 tests
   Start of acpi tests
   starting QEMU: exec aarch64-softmmu/qemu-system-aarch64 -qtest unix:/tmp/qtest-29244.sock -qtest-log /dev/null -chardev socket,path=/tmp/qtest-29244.qmp,id=char0 -mon chardev=char0,mode=control -display none -machine virt  -accel tcg -nodefaults -nographic -drive if=pflash,format=raw,file=pc-bios/edk2-aarch64-code.fd,readonly -drive if=pflash,format=raw,file=pc-bios/edk2-arm-vars.fd,snapshot=on -cdrom tests/data/uefi-boot-images/bios-tables-test.aarch64.iso.qcow2 -cpu cortex-a57 -accel qtest
   Start of virt tests
   starting QEMU: exec aarch64-softmmu/qemu-system-aarch64 -qtest unix:/tmp/qtest-29244.sock -qtest-log /dev/null -chardev socket,path=/tmp/qtest-29244.qmp,id=char0 -mon chardev=char0,mode=control -display none -machine virt  -accel tcg -nodefaults -nographic -drive if=pflash,format=raw,file=pc-bios/edk2-aarch64-code.fd,readonly -drive if=pflash,format=raw,file=pc-bios/edk2-arm-vars.fd,snapshot=on -cdrom tests/data/uefi-boot-images/bios-tables-test.aarch64.iso.qcow2  -cpu cortex-a57 -object memory-backend-ram,id=ram0,size=128M -numa node,memdev=ram0 -accel qtest
   starting QEMU: exec aarch64-softmmu/qemu-system-aarch64 -qtest unix:/tmp/qtest-29244.sock -qtest-log /dev/null -chardev socket,path=/tmp/qtest-29244.qmp,id=char0 -mon chardev=char0,mode=control -display none -machine virt  -accel tcg -nodefaults -nographic -drive if=pflash,format=raw,file=pc-bios/edk2-aarch64-code.fd,readonly -drive if=pflash,format=raw,file=pc-bios/edk2-arm-vars.fd,snapshot=on -cdrom tests/data/uefi-boot-images/bios-tables-test.aarch64.iso.qcow2  -cpu cortex-a57 -m 256M,slots=3,maxmem=1G -object memory-backend-ram,id=ram0,size=128M -object memory-backend-ram,id=ram1,size=128M -numa node,memdev=ram0 -numa node,memdev=ram1 -numa dist,src=0,dst=1,val=21 -accel qtest
   End of virt tests
   End of acpi tests
   End of aarch64 tests
~~~

We can launch the QTest from the debugger with something like this.

~~~
QTEST_QEMU_BINARY=aarch64-softmmu/qemu-system-aarch64 QTEST_QEMU_IMG=qemu-img gdb --args tests/qtest/bios-tables-test -m=quick -k --tap
~~~

But what if we want to debug QEMU itself with gdb?

To achieve this we would change the QTEST_QEMU_BINARY to something like this:

~~~
QTEST_QEMU_BINARY="sudo xterm -e gdb --tty $(tty) --args aarch64-softmmu/qemu-system-aarch64"
~~~

At least on our system we found that this only works by using sudo in front of xterm.  Otherwise we found that the debugger actually seems to crash and exit with error. :(

When you launch the test, an xterm will pop up with a window.  Inside that window, just hit r to run the test.  Also keep in mind that you need to have the DISPLAY environment variable set for xterm to pop up.

~~~
export DISPLAY=12.345.67.89:0
~~~

When you use the verbose option you get to see the actual QEMU command used.<BR>

It might look something like this:

~~~
 starting QEMU: exec i386-softmmu/qemu-system-i386 -qtest unix:/tmp/qtest-33869.sock -qtest-log /dev/null -chardev socket,path=/tmp/qtest-33869.qmp,id=char0 -mon chardev=char0,mode=control -display none -machine q35,kernel-irqchip=off -accel kvm -accel tcg -net none -display none -device pci-bridge,chassis_nr=1 -drive id=hd0,if=none,file=tests/acpi-test-disk-X74eKE,format=raw -device ide-hd,drive=hd0  -accel qtest
~~~

We also posted a follow-up article on [how to change QEMU Qtest accelerators]({{ site.url }}/qemu/debug-qemu-qtest-accelerators).