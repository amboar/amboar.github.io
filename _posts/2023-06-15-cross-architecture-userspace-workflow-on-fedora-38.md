---
title: A Cross-Architecture Userspace Workflow on Fedora 38
author: Andrew
---

These days I'm using Fedora on an `aarch64` system but still have a need for
cross-compiling bits and pieces to `x86_64` and other architectures.
Unfortunately, Fedora [doesn't provide a cross toolchain capable of building
userspace binaries][fedoraproject-discussion-hrw-statement]. So, is there a
relatively straight-forward work-around?

[fedoraproject-discussion-hrw-statement]: https://discussion.fedoraproject.org/t/cross-compiling-to-arm/71516/4

It appears so. I've taken to using [buildroot][]. The toolchains built by
`buildroot` require no further mucking around on the command line to specify
sysroots and such, they tend to just work. Further, `buildroot` has all the
magic we need to add bits and pieces to our cross-architecture userspace as we
go. In effect, we use it as a package manager for a source-based Linux
distribution.

[buildroot]: https://buildroot.org/

The usual use of `buildroot` is to generate and package a full root filesystem
image, kernel, and bootloader for flashing onto some embedded platform. For a
cross-architecture build environment we only need a rootfs populated in a
directory somewhere. To that end I've cooked up [this `x86_64`
defconfig][amboar-fedora-cross-buildroot-config] which is just the
[qemu_x86_64_defconfig][buildroot-qemu_x86_64_defconfig] but even more stripped
back.

[amboar-fedora-cross-buildroot-config]: /resources/x86_64_userspace_defconfig
[buildroot-qemu_x86_64_defconfig]: https://git.busybox.net/buildroot/tree/configs/qemu_x86_64_defconfig?h=2023.05

To get a usable toolchain:

```
$ git clone git://git.buildroot.net/buildroot
$ cd buildroot
$ wget https://amboar.github.io/notes/2023/06/15/x86_64_userspace_defconfig -O
configs/x86_64_userspace_defconfig
$ make O=builds/x86_64 x86_64_userspace_defconfig
$ make -j$(nproc) O=builds/x86_64
```

For adding bits and pieces that you need, use `make O=builds/x86_64 menuconfig`
to access [the usual menuconfig-style configuration
environment][buildroot-doc-menuconfig].

[buildroot-doc-menuconfig]: https://buildroot.org/downloads/manual/manual.html#_buildroot_quick_start

## Tying the `buildroot` cross-toolchains into `meson`

Meson supports cross compilation using [cross-files][meson-cross-files]. What's
interesting is that these can be placed in `~/.local/share/meson/cross` and are
then generally accessible through the file's basename.

[meson-cross-files]: https://mesonbuild.com/Cross-compilation.html

For undisclosed reasons I've used `O=/var/tmp/buildroot/x86_64` for my buildroot
build, but with that in mind, I have the following:

```
$ cat ~/.local/share/meson/cross/x86_64-buildroot-linux-gnu.ini
[host_machine]
system = 'linux'
cpu_family = 'x86_64'
cpu = 'x86_64'
endian = 'little'

[binaries]
c = '/var/tmp/buildroot/x86_64/host/bin/x86_64-buildroot-linux-gnu-gcc'
cpp = '/var/tmp/buildroot/x86_64/host/bin/x86_64-buildroot-linux-gnu-g++'
ld = '/var/tmp/buildroot/x86_64/host/bin/x86_64-buildroot-linux-gnu-ld'
strip = '/var/tmp/buildroot/x86_64/host/bin/x86_64-buildroot-linux-gnu-strip'
exe_wrapper = [ 'qemu-x86_64', '-L', '/var/tmp/buildroot/x86_64/target' ]
```

And with that in place I can now easily cross-build projects such as `libpldm`
with:

```
$ meson setup --cross-file=x86_64-buildroot-linux-gnu.ini builds/x86_64
$ meson compile -C builds/x86_64
```
