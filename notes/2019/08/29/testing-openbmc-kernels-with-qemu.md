## Testing OpenBMC kernels with QEMU

There are two intuitive approaches to testing OpenBMC kernels with QEMU:

1. Boot the kernel of interest with `-kernel`, `-dtb` and `-initrd` options to
   QEMU
2. Boot a full firmware image with `-drive file=...,if=mtd,format=raw`

Using the first approach is easy if we want to test the kernel out of
context, as we can just use a small, pre-built initrd as the userspace
environment. However it can be harder to deal with if we want to run a
platform-specific userspace to exercise particular paths in the kernel as we
would on hardware (we'd need to repack the rootfs in an initrd form, which is
possible, but takes some poking at bitbake configuration).

Using the second approach is more time-consuming than the first from a kernel
development perspective, as we need to either generate the image using our test
kernel source tree via `bitbake` or reimplement `bitbake`'s image preparation
in a little script of our own, which would mostly boil down to fiddly
invocations of `dd`.

As it turns out we can take a hybrid approach, using the full firmware image as
a base. In this case we can make use of existing builds of the firmware image,
as the image itself doesn't need to contain any custom code.

The main insight is to use QEMU's `-net nic -net user,tftp=...` options to
expose a directory on the host via TFTP in the guest. Assuming our working
directory is our kernel tree, build a FIT binary to boot and then launch QEMU:

```
$ mkimage -f witherspoon.its witherspoon.itb
$ qemu-system-arm \
	-M witherspoon-bmc \
	-nographic \
	-drive file=obmc-phosphor-image-witherspoon.ubi.mtd,if=mtd,format=raw \
	-no-reboot \
	-net nic -net user,tftp=$(realpath .) # absolute path to kernel tree
qemu-system-arm: Aspeed iBT has no chardev backend


U-Boot 2016.07 (Aug 28 2019 - 05:21:39 +0000)

       Watchdog enabled
DRAM:  496 MiB
Flash: 32 MiB
In:    serial
Out:   serial
Err:   serial
Net:   aspeednic#0
Error: aspeednic#0 address not set.

Hit any key to stop autoboot:  0
```

Stop autoboot, and configure a dummy MAC address:

```
ast# setenv ethaddr C0:FF:EE:00:00:02
ast# saveenv
Saving Environment to Flash...
Un-Protected 1 sectors
Un-Protected 1 sectors
Erasing Flash...
. done
Erased 1 sectors
Writing to Flash... done
Protected 1 sectors
Protected 1 sectors
```

Now, use `dhcp` to fetch a specific file (our `witherspoon.itb`) from the TFTP
"server".  The path is relative to the root directory specified on the QEMU
commandline.

```
ast# dhcp witherspoon.itb
aspeednic#0: PHY at 0x00
set_mac_control_register 1453
Found NCSI Network Controller at (0, 0)
Found NCSI Network Controller at (0, 1)
BOOTP broadcast 1
DHCP client bound to address 10.0.2.15 (2 ms)
Using  device
TFTP from server 10.0.2.2; our IP address is 10.0.2.15
Filename 'witherspoon.itb'.
Load address: 0x80800000
Loading: #################################################################
         #################################################################
         #################################################################
         #####
         36.3 MiB/s
done
Bytes transferred = 2851744 (2b83a0 hex)
```

Finally, in this Witherspoon-based example we do some platform-specific setup
of the u-boot environment, then boot our test kernel:

```
ast# run set_bootargs
ast# bootm
## Loading kernel from FIT Image at 80800000 ...
   Using 'conf@aspeed-bmc-opp-witherspoon.dtb' configuration
   Trying 'kernel@1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  uncompressed
     Data Start:   0x8080012c
     Data Size:    2814312 Bytes = 2.7 MiB
     Architecture: ARM
     OS:           Linux
     Load Address: 0x80001000
     Entry Point:  0x80001000
     Hash algo:    sha256
     Hash value:   02420fac85907143e68bb26fed72212324403e259883def6c308e0e6631be3b4
   Verifying Hash Integrity ... sha256+ OK
...
```

And now we're up and running with our test kernel and a platform-specific
userspace without having to rebuild the full firmware image.
