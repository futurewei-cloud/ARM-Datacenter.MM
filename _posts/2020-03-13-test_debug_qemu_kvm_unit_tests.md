---
title: "testing QEMU emulation: how to debug kvm-unit-tests"
date: 2020-03-12T16:48:58-04:00
author: Rob
categories:
  - QEMU
tags:
  - debugging
classes: wide
---

[kvm-unit-tests](https://git.kernel.org/pub/scm/virt/kvm/kvm-unit-tests.git/) are a set of low level tests designed to exercise KVM.  These tests are completely separate from QEMU (a different repo), but these tests can also be used to exercise QEMU.  

Testing QEMU with kvm-unit-tests is the use case we will explore below.<br> A comprehensive description of kvm-unit-tests is outside the scope of this blog post, but check out [this description](http://www.linux-kvm.org/page/KVM-unit-tests) for more details.  It is worth noting that these tests launch QEMU using their test as the -kernel argument, and thus debugging the test can be viewed as similar to debugging a kernel under QEMU.

To run the tests with QEMU use a command similar to the below.<br>
The -v option shows the commands it is issuing, which we will need for the next step.<br>
The QEMU= is needed to point it to the QEMU you want to test, and the ACCEL=tcg is needed
since we are testing with QEMU (instead of kvm).<br>
There are more details on running the tests in the [kvm-unit-tests README](http://git.kernel.org/pub/scm/virt/kvm/kvm-unit-tests.git/tree/README.md).

~~~
QEMU=../../qemu/build/aarch64-softmmu/qemu-system-aarch64 ACCEL=tcg ./run_tests.sh -v
~~~
When a test fails, the log directory (./logs) contains the log files for all tests.
A failure might look like this.

~~~
$ TIMEOUT=0 QEMU=../../qemu/build/aarch64-softmmu/qemu-system-aarch64 ACCEL=tcg ./run_tests.sh -v
TESTNAME=gicv2-mmio TIMEOUT=0 ACCEL=tcg ./arm/run arm/gic.flat -smp $((($MAX_SMP < 8)?$MAX_SMP:8)) -machine gic-version=2 -append 'mmio'
PASS gicv2-mmio (17 tests, 1 skipped)
TESTNAME=gicv2-mmio-up TIMEOUT=0 ACCEL=tcg ./arm/run arm/gic.flat -smp 1 -machine gic-version=2 -append 'mmio'
FAIL gicv2-mmio-up (17 tests, 2 unexpected failures)
TESTNAME=gicv2-mmio-3p TIMEOUT=0 ACCEL=tcg ./arm/run arm/gic.flat -smp $((($MAX_SMP < 3)?$MAX_SMP:3)) -machine gic-version=2 -append 'mmio'
FAIL gicv2-mmio-3p (17 tests, 3 unexpected failures)
~~~
The first line of the log file has the command needed to run this test standalone.

~~~
../../qemu/build/aarch64-softmmu/qemu-system-aarch64 -nodefaults -machine virt,accel=tcg -cpu cortex-a57 -device virtio-serial-device -device virtconsole,chardev=ctd -chardev testdev,id=ctd -device pci-testdev -display none -serial stdio -kernel arm/gic.flat -smp 1 -machine gic-version=2 -append mmio # -initrd /tmp/tmp.RrDNvP8sPT
~~~

In order to attach the debugger we want to add two arguments to the above command line.<BR>
<B>-s</B> tells QEMU to use the TCP port :1234<BR>
<b>-S</b> will pause at startup, waiting for the debugger to attach.

~~~
../../qemu/build/aarch64-softmmu/qemu-system-aarch64 -s -S -nodefaults -machine virt,accel=tcg -cpu cortex-a57 -device virtio-serial-device -device virtconsole,chardev=ctd -chardev testdev,id=ctd -device pci-testdev -display none -serial stdio -kernel arm/gic.flat -smp 1 -machine gic-version=2 -append mmio # -initrd /tmp/tmp.RrDNvP8sPT
~~~

Then in another console, we will run gdb.

It is worth noting that in this case we are debugging an aarch64 QEMU on an x86_64 host.  So we will need to use the gdb-multiarch since it can do cross debugging.

~~~
$ gdb-multiarch
(gdb) set arch aarch64
The target architecture is assumed to be aarch64
(gdb) file ./arm/gic.elf
Reading symbols from ./arm/gic.elf...(no debugging symbols found)...done.
(gdb) 
~~~
Note how it says <I>no debugging symbols found</I> above.  That indicates that the .elf file is missing symbols.  Note that the .elf file is essentially the built "kernel" that we are going to give to QEMU.  There is also a similar .flat file.  This is the same as the elf, but just stripped of any unnecessary sections.

In order to get the symbols built into the .elf file, we need to modify the Makefile.

The issue is that when we link the .elf file, it excludes the symbols.  Because of the special way we are constructing the test, as essentially a kernel, kvm-unit-test uses a custom linker script.  The script is in /arm/flat.lds.

There is a section of the flat.lds where we tell the linker which sections of the objects to set aside in the resulting binary.

~~~
    /DISCARD/ : {
        *(.note*)
        *(.interp)
        *(.debug*)
        *(.comment)
        *(.dynamic)
    }
~~~

We will need to remove the "*(.debug)" line so that the symbol table is not excluded (!excluded == included). 

This section will then look like the below.

~~~
    /DISCARD/ : {
        *(.note*)
        *(.interp)
        *(.comment)
        *(.dynamic)
    }
~~~

Once we re-build the binaries, we can resume our debugging process where we left off launching qemu in one console and then launching gdb-multiarch in a different console.

~~~
$ gdb-multiarch
(gdb) set arch aarch64
The target architecture is assumed to be aarch64
(gdb) file ./arm/gic.elf
Reading symbols from ./arm/gic.elf...done.
(gdb) 
~~~
Note how the symbols were read successfully !

But we're not out of the woods yet.  To get the symbols loaded properly at the correct address, we need to first find the location where our code is loaded.

We will first attach to the target.  Note that we use :1234 this is the port that was specified for us by the -s command to QEMU.

~~~
(gdb) target remote :1234
Remote debugging using :1234
warning: No executable has been specified and target does not support
determining executable automatically.  Try using the "file" command.
0x0000000040000000 in ?? ()
~~~
We then disassemble the first snippet of code.  We disassemble using the address which is displayed after we attach to the target: 0x0000000040000000

~~~
(gdb) disassemble 0x0000000040000000,+100
Dump of assembler code from 0x40000000 to 0x40000064:
=> 0x0000000040000000:  ldr x0, 0x40000018
   0x0000000040000004:  mov x1, xzr
   0x0000000040000008:  mov x2, xzr
   0x000000004000000c:  mov x3, xzr
   0x0000000040000010:  ldr x4, 0x40000020
   0x0000000040000014:  br  x4
   0x0000000040000018:  .inst   0x44000000 ; undefined
   0x000000004000001c:  .inst   0x00000000 ; undefined
   0x0000000040000020:  .inst   0x40080000 ; undefined
~~~
If you follow the assembly above, you will see that we load in an address to jump to: 0x40080000

This happens to be the address where we loaded our code.

In order to load the symbols we need to determine the current location of our .text and .data segments.

"info files" will list the current offsets for our gic.elf binary.

~~~
(gdb) info files
Symbols from "/home/rob/qemu/alex-kvm-unit-tests/build/arm/gic.elf".
Local exec file:
    `/home/rob/qemu/alex-kvm-unit-tests/build/arm/gic.elf', file type elf64-littleaarch64.
    Entry point: 0x0
    0x0000000000000000 - 0x000000000000ef84 is .text
    0x0000000000010720 - 0x0000000000010768 is .dynsym
    0x0000000000010768 - 0x0000000000010769 is .dynstr
    0x0000000000010770 - 0x0000000000010788 is .hash
    0x0000000000010788 - 0x00000000000107a4 is .gnu.hash
    0x00000000000107a8 - 0x00000000000107c8 is .got
    0x00000000000107c8 - 0x0000000000012308 is .rodata
    0x0000000000012308 - 0x0000000000013670 is .data
    0x0000000000013670 - 0x0000000000022798 is .bss
    0x0000000000010000 - 0x0000000000010720 is .rela.dyn
~~~
Unload the current symbols with the "file" command (no argument).
Note that we need to answer 'y' below.

~~~
(gdb) file
No executable file now.
Discard symbol table from `/home/rob/qemu/alex-kvm-unit-tests/build/arm/gic.elf'? (y or n) y
No symbol file now.
~~~
Find the current .text and .data addresses.  Since we know our code was loaded at the address we found above (0x40080000), we merely need to add the offsets from the table to the load address.

.text is at offset 0 in the table, so the address is 0x40080000
.data is at offset 0x12308, so 0x40080000 + 0x12308 = 0x40092308

The add-symbol-file command will map the symbols properly.
Note again we need to answer 'y' below.

~~~
(gdb) add-symbol-file ./arm/gic.elf 0x40080000 -s .data 0x40092308
add symbol table from file "./arm/gic.elf" at
    .text_addr = 0x40080000
    .data_addr = 0x40092308
(y or n) y
~~~
That's it!  The symbols are loaded.

We can tests this by setting a breakpoint or listing code.

~~~
Reading symbols from ./arm/gic.elf...done.
(gdb) b main
Breakpoint 1 at 0x40081748: file /home/rob/qemu/alex-kvm-unit-tests/arm/gic.c, line 517.
(gdb) l main
511     if (gic_version() == 2)
512         test_targets(nr_irqs);
513 }
514 
515 int main(int argc, char **argv)
516 {
517     if (!gic_init()) {
~~~
 
