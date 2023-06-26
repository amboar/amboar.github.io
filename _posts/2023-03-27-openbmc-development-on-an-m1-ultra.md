---
title: OpenBMC Development on an Apple M1 Ultra
author: Andrew
---

I've had an M1 Ultra lying beside me for some time now, waiting for me to
find a good workflow and migrate onto it.

As it turns out, sandbagging on migrating for 6 months or so turned out to be a
bit of a blessing. The software ecosystem evolved quite a bit from when I first
tried to figure out how I wanted to work with it. Until now I've been running
Linux on various Lenovo Thinkpads and so I wanted a workflow with similar
properties. I want to use Linux, but am constrained to continuing to run macOS,
so that rules out e.g. [Asahi][asahi-linux].

[asahi-linux]: https://asahilinux.org/

However, we can run Linux under a hypervisor, and with that I had the following
goals:

1. Use Linux to maintain familiarity
2. Run Linux as a native guest (aarch64) for minimal overhead
3. Bridged networking for unhampered guest network access
4. Keep critical data on the host and share it into the guest
5. Secondary activities like web browsing and chat continue to happen on the
   host

Critical data in this case is essentially my `$HOME` from my Linux laptop. I
want to keep the point of coherency on the host so I don't accidentally wipe the
critical data out by mindlessly blowing away a VM disk image. Essentially, the
VM itself shouldn't be special. Further, I'd like that `$HOME` data copied from
my Thinkpad(s) to serve as my `$HOME` in the Linux guest.

To continue performing secondary activities under macOS, I plan to use
`terminal.app` and just `ssh` into the guest. This keeps all the copy/paste and
click semantics working without too much hassle.

Previously I'd cooked up some horrific combination of QEMU,
[vde_vmnet][] for network bridging, and 9pfs for directory sharing
betwen the host and the guest. There were many things that were wrong with this.
Hooking QEMU up to `vde_vmnet` avoids the need to run QEMU as root for bridge
networking, but externalises the network from the QEMU process. This appeared to
cause some throughput issues. The throughput issues were compounded by [fs-cache
bugs for 9pfs in the guest Ubuntu kernel][linux-lore-9p-duplicate-cookie], which
meant I had to rebuild the guest kernel to disable fs-cache. Further, it turned
out that [QEMU had 9pfs issues all of its own][schreibt-qemu-9p-performance].

[vde_vmnet]: https://github.com/lima-vm/vde_vmnet
[linux-lore-9p-duplicate-cookie]: https://lore.kernel.org/lkml/3791738.ukkqOL8KQD@silver/
[schreibt-qemu-9p-performance]: https://linus.schreibt.jetzt/posts/qemu-9p-performance.html

## Software Bits

To get there this time around I've used:

1. [UTM][utm] 4.1.6 (75)
   1. Experimental support for [Hypervisor Virtualization Framework][apple-macos-hvf]
      (HVF)
   2. [VirtioFS][utm-docs-macos-virtiofs] support
   3. Boot from ISO support for HVF guests
2. [Fedora 37][getfedora]

[utm]: https://mac.getutm.app/
[apple-macos-hvf]: https://developer.apple.com/videos/play/wwdc2022/10002/
[utm-docs-macos-virtiofs]: https://docs.getutm.app/guest-support/linux/#macos-virtiofs
[getfedora]: https://getfedora.org/

## Configuration

I've created a guest with the following configuration and resources:

1. HVF Guest
2. 8 cores
3. 16GiB RAM
4. 128GiB Storage

With that, I did a stock install of Fedora 37.

## Tricks

With Fedora 37 installed in the guest I needed a few tricks to get things
working as I desired.

### On the host

1. [Create a volume with case-sensitive APFS][apple-support-apfs] for my home
   data
2. Sync Linux `$HOME` onto the case-sensitive volume
3. Create a [VirtioFS share of my home data in UTM][utm-docs-basics]

[apple-support-apfs]: https://support.apple.com/en-au/guide/disk-utility/dsku19ed921c/22.0/mac/13.0
[utm-docs-basics]: https://docs.getutm.app/basics/basics/

### In the guest

#### Set up the directory share as my guest home directory

1. `mkdir /mnt/host`
2. `echo 'share /mnt/host virtiofs rw,nofail 0 0' >> /etc/fstab`
3. `rm -rf /home/$USER`
4. `ln -s /mnt/host/$USER /home/$USER`

#### Cater to the VirtioFS share mount mangling permissions (?)

Fixing this is a double win as we can keep the build artifacts inside the guest
and save on some overhead out to the host.

1. `mkdir -p /var/tmp/bitbake/build`
2. `ln -s /var/tmp/bitbake/build ~/src/openbmc/openbmc/build`

#### [Fix qemu-user][qemu-issue-447] so `meson` actually works under `bitbake`

[qemu-issue-447]: https://gitlab.com/qemu-project/qemu/-/issues/447

Attempting to build OpenBMC in the guest eventually lead to `meson` complaining
that the executables created by cross-compilers weren't runnable. On the surface
this seems fair, but due to ✨implementation details✨ of the `meson` support
in [Poky][yocto-poky], running cross-built binaries on the build machine is
necessary.

[yocto-poky]: https://www.yoctoproject.org/software-overview/reference-distribution/

Once I eventually dug through all the `bitbake` and `meson` logs to find out
what was actually being invoked, it turned out `qemu-arm` was producing the
following error:

```
qemu-arm: Unable to reserve 0xffff0000 bytes of virtual
address space at 0x8000 (Success) for use as guest address
space (check your virtual memory ulimit setting,
min_mmap_addr or reserve less using -R option)
```

In the process of tracking this down I ended up [sending a couple of minor
patches to qemu][qemu-linux-user-patches]. Anyway, the issue was largely
solved by:

[qemu-linux-user-patches]: https://lore.kernel.org/qemu-devel/20230327115524.1981482-1-andrew@aj.id.au/

1. `sudo dnf install qemu-user`
2. `echo 'vm.mmap_min_addr = 65536' > /etc/sysctl.d/qemu-arm.conf`
3. `systemctl reboot`

### Other random things that went wrong

Various things went wrong in silly ways. As a bit of a record:

#### Fix up qemu submodules

I hadn't done any upstream development in recent times and submodules had come,
gone and shifted around:

1. `git submodule sync --recursive`
2. `git submodule update --recursive`

#### Deal with qemu build dependencies

1. `dnf builddep qemu`

#### Fix up Github's SSH host key

Github managed to [publish their private SSH RSA host key][github-key-rotation]
right around the time I started this migration, so that needed fixing in my
`~/.ssh/known_hosts`.

[github-key-rotation]: https://github.blog/2023-03-23-we-updated-our-rsa-ssh-host-key/

#### Fix up time being in the past

Not sure what happened here, but I managed to freeze the VM several times and
this may have contributed to the issue. The freezing seemed to begin after
allocating 32GiB of memory to the VM, which is the entirety of the host RAM.
It's been stable since I reduced that to 16GiB.

1. `timedatectl set-ntp false`
2. `timedatectl set-time 22:09`
3. `timedatectl set-ntp true`

## How It Went

With all this configured I found I could build a full OpenBMC image in the guest
in a bit over an hour, from cold caches. Taking that straight to the bank!
