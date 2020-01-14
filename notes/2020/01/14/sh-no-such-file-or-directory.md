## -sh: no such file or directory

```
root@bmc:~# /usr/local/bin/phosphor-psu-monitor -c /usr/local/share/phosphor-psu-monitor/bmc_psu_config.json
-sh: /usr/local/bin/phosphor-psu-monitor: No such file or directory
root@bmc:~# ls -l /usr/local/bin
-rwxr-x--x    1 root     root       6004032 Jan 10 23:18 phosphor-psu-monitor
```

OpenBMC supports several generations of BMC SoCs. In the case of ASPEED BMC
SoCs, each generation has moved forward with the supported ISA and hardware
features, and the ARMv7 AST2600 now sports hard-float support in the form of
vfpv4d16.

Switching from the soft-float strategy of previous generations to the hard-float
strategy supported by hardware in the AST2600 requires a change of kernel
configuration and binary ABI options passed to the compiler for an application
[if the base system software has been configured for
hard-float](https://github.com/openbmc/meta-aspeed/commit/53e322518fe6f471663746934e38bf5d8143a501).
If we use the wrong ABI we end up with the cryptic output provided above, which
is telling us that the kernel cannot find the right handler for the binary.

In the case of ELF binaries, we can use `readelf --dynamic` as a
system-independent way to determine the libraries that need to be loaded (as
opposed to `ldd`, which you might want to be [cautious of using
anyway](https://catonmat.net/ldd-arbitrary-code-execution)):

```sh
$ readelf --dynamic phosphor-psu-monitor | grep NEEDED
 0x00000001 (NEEDED)                     Shared library: [libsdbusplus.so.1]
 0x00000001 (NEEDED)                     Shared library: [libsystemd.so.0]
 0x00000001 (NEEDED)                     Shared library: [libsdeventplus.so.0]
 0x00000001 (NEEDED)                     Shared library: [libstdc++.so.6]
 0x00000001 (NEEDED)                     Shared library: [libgcc_s.so.1]
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x00000001 (NEEDED)                     Shared library: [ld-linux-armhf.so.3]
```

The final entry here is `ld-linux-armhf.so.3` - this binary will work on our
hard-float-enabled systems. This is in contrast to:

```sh
$ readelf -d phosphor-psu-monitor | grep NEEDED
 0x00000001 (NEEDED)                     Shared library: [libsdbusplus.so.1]
 0x00000001 (NEEDED)                     Shared library: [libsystemd.so.0]
 0x00000001 (NEEDED)                     Shared library: [libsdeventplus.so.0]
 0x00000001 (NEEDED)                     Shared library: [libstdc++.so.6]
 0x00000001 (NEEDED)                     Shared library: [libgcc_s.so.1]
 0x00000001 (NEEDED)                     Shared library: [libc.so.6]
 0x00000001 (NEEDED)                     Shared library: [ld-linux.so.3]
```

This binary is looking for `ld-linux.so.3`, which is not present on the
hard-float-enabled images, and its absence is the reason we get `no such file
or directory`.

If you encounter this problem with OpenBMC make sure you are using the right
SDK for the system you're building binaries for and that you are not moving
back-and-forth between images that switch between soft- and hard-float
strategies.
