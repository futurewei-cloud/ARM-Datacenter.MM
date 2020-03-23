---
title: "Testing QEMU emulation: how to change QTest Accelerator"
date: 2020-03-23T09:25:05-04:00
author: Rob
categories:
  - QEMU
tags:
  - debugging
classes: wide
---
<B>How can we change [QEMU](https://www.qemu.org/)  QTest to use different accelerators?  And why would we do this?</B><BR>
<span style="font-size:60%">This article is a follow-up to a prior article we posted on [how to debug QEMU Qtests](../debug-qemu-qtests).</span>

Each QTest will decide which accelerators it uses.  For example, the test might try to use 'kvm', which causes QEMU to use KVM to execute code. Or the test might try to use 'TCG' support, where QEMU will emulate the instructions itself.  Regardless of which path is chosen, this choice inevitably results in different code paths getting exercised inside QEMU itself.

In some cases when developing QEMU code, we might want to force certain code paths which are specific to different accelerators.  In this case we have a few things to decide.  Take the case for example, where we want to force a specific TCG code path on an aarch64 machine for an aarch64 QTest.  We will use the tests/qtest/arm-cpu-features test as an example.

This test it selects the specific accelerator(s) to use for each test case.  It is possible that we might want to force the use of a specific accelerator to force that code path in QEMU.  We might want to use TCG instead of kvm for instance.   

In this case we would need to edit the test, for instance tests/qtest/arm-cpu-features.c, and replace the use of "kvm" with "tcg", or in cases where both -accel kvm and -accel tcg are used, just remove the kvm.

This will have the effect of forcing the use of a specific code path, which can be very useful when debugging or validating a change.
 
