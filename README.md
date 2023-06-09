## Notes

### [Naming Functions in C](notes/2023/06/09/naming-functions-in-c.md)

To steal from wikipedia, C is an imperative procedural language. Given the
language's lack of formal support for more abstract constructs like
object-oriented classes, it's easy to reach the view that its ecosystem is a
grab-bag of loosely related functions. Developing applications and libraries
with this perspective can lead to choices that feel kind of arbitrary. We need
to be conscious that what we're doing is imposing a structure on the code in
order to organise our own thoughts and those of others.

### [Inspecting Boot Performance in OpenBMC](notes/2023/06/08/inspecting-boot-performance-in-openbmc.md)

OpenBMC uses systemd as a system management daemon. While this raises some
eyebrows given the environment, it's what we have. At least it provides
reliability and familiarity as we disregard the perceived complexity. With
systemd comes an opportunity to use `systemd-bootchart` for measuring boot-time
behaviour of the system. While it's relatively easy to use in general, some of
the details of the OpenBMC boot process can get in the road.

### [Deprecated, Stable and Testing ABIs for `libpldm`](notes/2023/06/06/deprecated-stable-and-testing-abis-for-libpldm.md)

Developing and maintaining libraries is a very different ballgame to
applications. Internal functions of an application tend to have a closed set of
call-sites. Under these conditions refactoring is often straight-forward: Rework
your internal APIs and then clean up the resulting compiler errors. By contrast
libraries rarely have a closed set of call-sites for their APIs. This means
breaking an API impacts a potentially unknowable number of applications, and
makes for a bad experience for the library's users when they try to update.

### [Host Console Access using Aspeed BMC SoCs](notes/2023/05/09/host-console-access-using-aspeed-bmc-socs.md)

Experience suggests that configuring an Aspeed BMC for host console access can
be a confusing task. The UART capabilities provided by the SoCs allow access to
the host console both via physical connectors on the rear of the chassis and
also via Serial-over-LAN (SOL).

### [Happenings in `obmc-console`](notes/2023/05/08/happenings-in-obmc-console.md)

`obmc-console` is quite a slow-paced project relative to others in the OpenBMC
ecosystem, but recently I've merged quite a few changes. The bad news is not
all of them have have kept things in working order in the OpenBMC distro, so
let's look at what's happened, what's broken, and what we need to do to fix it.

### [Fixing Formatting CI Failures in OpenBMC Projects](notes/2023/05/03/fixing-formatting-CI-failures-in-openbmc.md)

Pushing patches for review to `gerrit.openbmc.org` automatically triggers CI
jobs on `jenkins.openbmc.org`. Almost always this triggers builds and a bunch of
linters to run over the change. Many of the linters are also formatters, such as
`prettier`, `black` or clang-format`.

As it stands any differences introduced by the linters causes a build failure.

### [Testing `obmc-console` with `socat`](notes/2023/05/02/testing-obmc-console-with-socat.md)

This is a bit of a gross hack. However, it serves to demonstrate a way to test
the `obmc-console` stack without requiring integration into a BMC and booting
its host (or some equally tedious arrangement).

### [`touch-required` FIDO2 Authentication with `sshd` and a Yubikey on Fedora 38](notes/2023/05/01/sshd-fido2-touch-required.md)

Because I'm lazy the network contains printers and other devices whose
firmware hygiene generally causes infosec side-eye. Leaving `sshd` exposed to
password-based authentication attempts didn't evoke feelings of comfort.

### [Exploiting `obmc-console` service units to expose multiple host consoles to the BMC network](notes/2023/03/31/exploiting-obmc-console-service-units-for-multiple-host-consoles.md)

`obmc-console` provides the plumbing to expose one or more host consoles onto
BMC's network interfaces. It comes in two parts:

1. `obmc-console-server`
2. `obmc-console-client`

### [A Global PLDM Instance ID Allocator in Userspace for `libpldm`](notes/2023/03/29/a-global-pldm-instance-id-allocator-for-libpldm.md)

In Motivating a New Scheme for PLDM Instance ID Management in
OpenBMC I talked about why we need to change how instance IDs are managed in
OpenBMC. Underpinning it is the shift to using `AF_MCTP` sockets provided by
Linux.

### [Motivating a New Scheme for PLDM Instance ID Management in OpenBMC](notes/2023/03/28/motivating-a-new-scheme-for-pldm-instance-id-management-in-openbmc.md)

Recently Rashmica has been doing some work to enable use of Linux's `AF_MCTP`
sockets in OpenBMC. Until now we've relied on a userspace implementation of MCTP
through libmctp, but this rapidly hit limitations at the kernel/userspace
interface boundary. To fix that, Code Construct did the work to move MCTP into
the kernel.

### [OpenBMC development on a Apple M1 Ultra](notes/2023/03/27/openbmc-development-on-an-m1-ultra.md)

I've had an M1 Ultra lying beside me for some time now, waiting for me to
find a good workflow and migrate onto it.

### [An `opkg`-based OpenBMC development workflow](notes/2022/05/13/opkg-based-development-workflow.md)

Previously I talked about the mechanics of how I develop bits and pieces of
userspace for OpenBMC. What I will discuss this time is an alternative
flow that replaces the use of `devtool deploy-target` with `opkg`.

### [My OpenBMC userspace development workflow](notes/2022/01/13/openbmc-development-workflow.md)

I recently pushed a couple of tools (overlay and bbdbg) into the openbmc-tools
repository that help me develop userspace software for OpenBMC. This post
describes how I work with all the different tools involved.

### [History-preserving fork maintenance with git](notes/2021/09/16/history-preserving-fork-maintenance-with-git.md)

Working with long-term forks in git can be painful. It doesn't have to be.

### [`dbus-pcap`: Debugging OpenBMC with `busctl capture`](notes/2020/01/22/dbus-pcap.md)

[@jessfraz](https://twitter.com/jessfraz) recently wrote an [ACM Queue article
on BMCs and the availability of open-source BMC
firmware](https://queue.acm.org/detail.cfm?id=3378404). OpenBMC gets a mention,
though the article also points out that it's modular design interconnected with
D-Bus "makes the BMC software more complex to debug, audit, and put into
production."

### [`dd conv=notrunc`: Working around limitations of busybox](notes/2020/01/21/ddconvnotrunc.md)

Busybox's `dd(1)` as shipped in OpenBMC (as of Jan 21 2020) doesn't support
`conv=notrunc`, which is a mighty handy option if you're trying to patch some
binaries. Which I unfortunately was.

### [-sh: no such file or directory](notes/2020/01/14/sh-no-such-file-or-directory.md)

OpenBMC supports several generations of BMC SoCs. In the case of ASPEED BMC
SoCs, each generation has moved forward with the supported ISA and hardware
features, and the ARMv7 AST2600 now sports hard-float support in the form of
vfpv4d16.

### [Secure and Robust eMMC Flash Layout Design for BMCs](notes/2020/01/08/emmc-flash-layout-design-for-bmcs.md)

Recently a few of us were interested in designing an eMMC flash layout that
allowed for secure boot of a BMC's userspace while also catering to robustness
across updates. This post covers a script I developed to road-test secure
rootfs eMMC images under QEMU. The script appears at the end after the
discussion of how we implement it.

### [KASAN for ARM](notes/2019/12/27/arm-kasan.md)

KASAN (Kernel Address SANitizer, the kernel implementation of an [existing
userspace aid](https://en.wikipedia.org/wiki/AddressSanitizer)) for 32-bit ARM
is not yet in mainline Linux, but [v6 of a
series adding support](https://lore.kernel.org/lkml/20190617221134.9930-1-f.fainelli@gmail.com/)
was posted in mid-2019. As part of tracking down squashfs decompression errors
in OpenBMC I took v6 for a test drive. Ultimately it didn't help my cause, but
I had some fun debugging KASAN along the way.

### [Enabling Linux Dynamic Debug Statements when Scripting QEMU](notes/2019/12/22/enabling-dyndbg-while-scripting-qemu.md)

Trying to debug intermittent data corruption under QEMU can be fairly painful,
especially if you're trying to reproduce with stock boot images. I happened to
be in this dark corner recently and wanted to enable some extra debug output in
the provided kernel. Thankfully Linux supports enabling dynamic debug
statements on the kernel commandline, though depending on which statements you
want and how you're trying to enable them this can be a straightforward or like
trying to run a Rube Goldberg machine in reverse.

### [Debugging UBI Corruption At Boot](notes/2019/12/22/debugging-ubi-corruption-at-boot.md)

Unsorted Block Images is a magic Linux kernel subsystem for handling wear
levelling, data integrity management and dynamic partitioning of raw flash
devices. It's magic in the sense that it does a lot of work under the covers,
frequently shuffling data around to uphold the desirable properties of the
subsystem.

### [Abusing QEMU Device Models for Debugging](notes/2019/12/22/abusing-qemu-device-models.md)

Debugging broken kernel / hypervisor interactions can be painful, and sometimes
requires some creativity to extract the necessary information. This was the
case recently when chasing down a bug whose most obvious symptom was squashfs
decompression failures that appeared intermittently when booting Witherspoon
OpenBMC firmware on QEMU's witherspoon-bmc platform model.

### [Board Bringup, XYZMODEM and Terminal Servers](notes/2019/09/06/board-bringup-xyzmodem-and-terminal-servers.md)

You're doing bringup of a board or SoC or have got yourself in a tight spot;
you have no networking and are unable to write to local storage in the runtime
environment. How do you boot a custom firmware or kernel? If you're local the
obvious approach is to use an external tool to write the boot storage, but lets
add to the challenge and say you're doing this remotely. You have your board
hooked up to a terminal server, and are at the u-boot prompt.

### [Bitbake and Git Submodules Done Right](notes/2019/08/30/bitbake-and-git-submodules.md)

A part of OpenBMC has a use-case where we needed to build a project composed of
multiple repositories arranged as submodules with a submodule tree depth
greater than one. The catch is that we don't need to initialise submodules
below the first layer, and the approach of initialising them anyway means a
large time and bandwidth penalty downloading unnecessary information.

### [Testing OpenBMC kernels with QEMU](notes/2019/08/29/testing-openbmc-kernels-with-qemu.md)

There are two intuitive approaches to testing OpenBMC kernels with QEMU:

1. Boot the kernel of interest with `-kernel`, `-dtb` and `-initrd` options to
   QEMU
2. Boot a full firmware image with `-drive file=...,if=mtd,format=raw`

### [Debugging Hostboot the Hard Way](notes/2018/09/03/debugging-hostboot.md)

Hostboot's console output is pretty terse, and doubly-so when things go wrong.
Debugging Hostboot the Hard Way gives some insight on how to extract more
information from hostboot to root-cause problems and provide some tips on
debugging code under development.

### [General Architecture of Hostboot](notes/2018/08/19/hostboot-architecture.md)

[Hostboot](https://github.com/open-power/hostboot) is split into several parts,
in terms of the artifacts generated and the roles of those parts. At a high
level, hostboot is its own cache-contained operating system. Here we explore
how this firmware OS fits together.

### [Hacking Hostboot](notes/2018/08/17/hacking-hostboot.md)

[Hostboot](https://github.com/open-power/hostboot) is one part of the firmware
stack that initialises an OpenPOWER system - it is the first piece of software
to execute on the "main" cores of the CPU. Developing software that runs in
this environment is always challenging, but the nature of its implementation
also adds to the level of difficulty.

---

<a rel="me" href="https://social.tchncs.de/@arj">Mastodon</a>
<a rel="me" href="https://twitter.com/mramboar">Twitter</a>
