---
title: Abusing QEMU Device Models for Debugging
author: Andrew
---

Debugging broken kernel / hypervisor interactions can be painful, and sometimes
requires some creativity to extract the necessary information. This was the
case recently when chasing down a bug whose most obvious symptom was squashfs
decompression failures that appeared intermittently when booting Witherspoon
OpenBMC firmware on QEMU's witherspoon-bmc platform model.

To track this down I wanted to boot a stock Witherspoon QEMU image, and record
data and metadata associated with flash accesses to find accesses that were
corrupted. This information was to be extracted from the guest kernel in the
ASPEED Firmware Memory Controller (FMC) SPI-NOR driver. Recording the data
necessitated a custom kernel, but this was injected into the environment via
the [TFTP method I've outlined
previously](/notes/2019/08/29/testing-openbmc-kernels-with-qemu.html).

The issue appeared when getting userspace up and running, and when the problem
occurred it tended to take out the ability to interact with the runtime
environment. The consequence is that any method to record data on the failure
shouldn't depend on using the guest session to extract the data after the fact.
Writing the data to flash is dubious as we don't know how or where in the stack
the corruption is occuring, so we may infact corrupt the debug data in the
process of writing it out. There's a secondary question of where we might store
it in the flash given it's dominated by the shifting sands of the UBI partition
(the only static components of the flash layout are u-boot, its environment and
the UBI partition). Using `printk()` was out of the question due to the volume
of data being written which both killed the runtime performance of the session
and obscured the usual output of the boot process.

I ended up using an approach inspired by Alistair Popple, who's abuse of QEMU
to do some PCIe hardware verification during chip bringup has stuck with me.
QEMU implements many device models to mimic behaviour of the hardware, but it
doesn't have to be that way. In Al's case we had a chip that was progressing
through verification and we hadn't yet got the cores functional, but we needed
to de-risk the PCIe host bridge implementation and the firmware support. What
we did have was a functional QEMU model for the cores so we could boot
firmware and Linux under QEMU. Al's approach was to steal the accesses from the
host bridge MMIO memory region under QEMU and forward them via the chip debug
tools through to the physical chip from the guest, thus driving the hardware
and performing verification without needing to run code on the chip itself.

So with this insight of you-don't-have-to-implement-hardware-semantics in mind,
my approach was to steal an unused address in the FMC QEMU device model and
make it write data written to it to a file associated with the device instance.
The file gets opened for writing by QEMU when the device instance is realized.
In this way I can plumb data out directly from the guest kernel into the host
filesystem, dodging all the complexities and pitfalls of trying to store the
debug data in-band and avoiding the performance penalty and noise of
`printk()`. The register does not reflect any real behaviour of the device, but
the approach gives us a simple escape hatch through which to exfiltrate
arbitrary data.

Using this technique I was able to identify patterns in the data corruption and
determine a workaround that allows us to boot Witherspoon images reliably.
