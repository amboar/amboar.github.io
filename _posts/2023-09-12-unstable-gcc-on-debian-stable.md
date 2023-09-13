---
title: Unstable GCC on Debian Stable
author: Andrew
---

When I was experimenting with Fedora earlier in the year I worked around a lack
of cross-compilers for userspace components [by using
buildroot][fedora-userspace-workflow]. However, this
only gets you so far. Complex userspace components tend to require many
dependencies. Their existence tends to be discovered via `pkg-config`, and for
reasons I didn't explore buildroot may deploy the pkg-config files in varying
places in its host build tree. It's possible to specify multiple paths via
`PKG_CONFIG_PATH`, but really we're in kinda sketchy territory as it is. Also,
buildroot [explicitly doesn't support compilers in the target
environment][buildroot-faq-no-compiler-on-target]. If it did life may have been
slightly easier.

[fedora-userspace-workflow]: /notes/2023/06/15/cross-architecture-userspace-workflow-on-fedora-38.html
[buildroot-faq-no-compiler-on-target]: https://buildroot.org/downloads/manual/manual.html#faq-no-compiler-on-target

Anyway, I'm now running Debian, so the cross-build problem is solved more
easily: [Install the cross-compilers][debian-bookworm-gcc] and [register the host
architecture in question with `apt`][apt-multiarch].

[debian-bookworm-gcc]: https://packages.debian.org/search?searchon=names&keywords=gcc
[apt-multiarch]: https://wiki.debian.org/Multiarch/HOWTO

While Debian generally makes cross-compilation life easier, I'm running stable
(currently Bookworm) on the P14s, and that yields a different challenge.
Bookworm ships with GCC-12, which is unfortunate, because OpenBMC userspace
[needs to be compiled with GCC-13][openbmc-statement-gcc-13]. So we're back in
the familiar territory of needing a different compiler. The Debian testing
(trixie) and unstable (sid) suites ship GCC-13, but using the usual method of
[package pinning][apt-package-pin] results in `apt` wanting to upgrade most of
my system. And so again, we need a different solution.

[openbmc-statement-gcc-13]: https://github.com/openbmc/stdplus/issues/3#issuecomment-1636859245
[apt-package-pin]: https://wiki.debian.org/AptConfiguration

Despite that, life continues to be easier on Debian because we have
[debootstrap][debootstrap] at our disposal. Coupled with [schroot][schroot] and
[Meson's cross-files][meson-cross-files] we can get ourselves a relatively
easy-to-use setup that still allows us to dodge the docker container used for
OpenBMC CI.

[debootstrap]: https://wiki.debian.org/Debootstrap
[schroot]: https://wiki.debian.org/Schroot
[meson-cross-files]: https://mesonbuild.com/Cross-compilation.html

So let's set up the `debootstrap` chroot:

```
$ sudo debootstrap testing /home/andrew/Environments/debian/trixie http://ftp.au.debian.org/debian/
```

With that done we can create a small schroot config to make it easily usable:

```
$ cat <<EOF | sudo tee /etc/schroot/chroot.d/trixie.conf
[trixie]
type=directory
description=Debian Trixie (testing)
directory=/home/andrew/Environments/debian/trixie
users=andrew
root-groups=root,andrew
EOF
```

From there we can install GCC:

```
$ schroot -c trixie -u root
# apt install gcc
```

Back outside the chroot we then create a meson cross file to hook the new GCC
from the chroot into `meson setup ...`:

```
$ cat <<EOF > ~/.local/share/meson/cross/gcc-13
[host_machine]
system = 'linux'
cpu_family = 'x86_64'
cpu = 'x86_64'
endian = 'little'

[constants]
trixie = '/home/andrew/Environments/debian/trixie'

[built-in options]
c_args = [ '--sysroot=' + trixie ]
cpp_args = [ '--sysroot=' + trixie ]
pkg_config_path = trixie + '/usr/lib/x86_64-linux-gnu/pkgconfig'

[properties]
ld_args = [ '--sysroot=' + trixie ]
needs_exe_wrapper = true

[binaries]
c = trixie + '/usr/bin/x86_64-linux-gnu-gcc-13'
cpp = trixie + '/usr/bin/x86_64-linux-gnu-g++-13'
ld = 'ld'
strip = 'strip'
pkg-config = 'pkg-config'
exe_wrapper = 'trixie-meson-exe-wrapper'
EOF
```

You'll notice we specify both `exe_wrapper` and `needs_exe_wrapper`. These are
necessary as the test binaries will link against glibc from the Trixie sysroot.
The `trixie-meson-exe-wrapper` script sets up `LD_LIBRARY_PATH` such that the
Trixie sysroot is visible to the dynamic linker:

```
$ cat <<EOF > ~/.local/bin/trixie-meson-exe-wrapper
#!/bin/sh

SYSROOT="/home/andrew/Environments/debian/trixie/usr/lib/x86_64-linux-gnu"
LD_LIBRARY_PATH="${SYSROOT}:$LD_LIBRARY_PATH" "$@"
EOF
$ chmod +x ~/.local/bin/trixie-meson-exe-wrapper
```

We need to force the use of the wrapper with `needs_exe_wrapper` [as the
auto-detection is based on comparing the `system` and `cpu_family`
properties][meson-cross-file-needs-exe-wrapper] of the build and host systems.
That is broadly fine, except for the case of this little stunt we're trying to
pull off with our Trixie sysroot.

[meson-cross-file-needs-exe-wrapper]: https://mesonbuild.com/Cross-compilation.html#properties

With those pieces in place, building a meson-based project with GCC-13 outside
the chroot is a matter of:

```
$ meson setup --cross-file=gcc-13 ...
```
