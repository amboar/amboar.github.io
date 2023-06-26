---
title: Hacking Hostboot
author: Andrew
---

[Hostboot](https://github.com/open-power/hostboot) is one part of the firmware
stack that initialises an OpenPOWER system - it is the first piece of software
to execute on the "main" cores of the CPU. Its job is to initialise the
CPU's configuration and internal buses, stitch together the sockets into a
coherent SMP system, train and initialise main memory, and finally hand off
control of the CPU to the "PAYLOAD" firmware.

Hacking on hostboot is similarly non-trivial, so this post exists to document
what I've come to as a workflow.

Hostboot is most easily built by using the
[buildroot](https://buildroot.org/)-based
[op-build](https://github.com/open-power/op-build) system. However, if you
are hacking hostboot, generally you will want to build your own hostboot tree,
not the current stable release or
[master](https://github.com/open-power/hostboot/tree/master) as op-build's
buildroot configuration allows. op-build also offers a "Custom" option, though
you have to supply a commit ID, and that is irritating as it's forever
changing.

Also irritating is that building hostboot from scratch takes enough
time to go for a reasonable walk, even on a beefy machine. This wouldn't be so
bad if the build system didn't also mostly defeat
[ccache](https://ccache.samba.org/), which means we're left with trying to
build changes from the same working tree to avoid giving up in despair.

This last point eliminates taking advantage of overriding buildroot's
[`<pkg>_VERSION`](https://git.busybox.net/buildroot/tree/docs/manual/adding-packages-generic.txt?h=2018.05#n199),
[`<pkg>_SITE`](https://git.busybox.net/buildroot/tree/docs/manual/adding-packages-generic.txt?h=2018.05#n230)
and
[`<pkg>_SITE_METHOD`](https://git.busybox.net/buildroot/tree/docs/manual/adding-packages-generic.txt?h=2018.05#n270)
variables on the commandline, as similar to the `Custom` configuration option
they create ID-tagged directories in `output/build` (e.g.
`output/build/hostboot-8a8b2e4719156ae124e7bd198b089529b4040081`). Further, even
if we were to disregard the lengthy build times, using
`<pkg>_SITE_METHOD=git` with a local path assigned to `<pkg>_SITE`
breaks the build as hostboot makes use of symlinks with relative paths [and
these get broken by buildroot's archiving of git
repositories](https://patchwork.ozlabs.org/patch/957471/)

The correct way to go about this is to use buildroot's
[`<pkg>_OVERRIDE_SRCDIR`](https://buildroot.org/downloads/manual/manual.html#_using_buildroot_during_development)
variable, which is demonstrated below.

As a final tangent, I like to keep my laptop as the canonical source of the
work that I'm doing, but it's also not beefy enough to build hostboot in a
reasonable amount of time. To this end I've set up a git remote configured with
`git config receive.denyCurrentBranch warn` on a bigger machine, and then I
force push to that from my laptop.  On the remote machine I do a `git reset
--hard` to pick up the new changes, and then issue the following to build them:

```
$ op-build HOSTBOOT_OVERRIDE_SRCDIR=/path/to/hostboot/ hostboot-rebuild machine-xml-rebuild openpower-pnor-rebuild
```

The resulting PNOR image can then be flashed to the target machine for testing.

Hostboot series:

* [General Architecture of Hostboot](/notes/2018/08/19/hostboot-architecture.html)
* [Debugging Hostboot the Hard Way](/notes/2018/09/03/debugging-hostboot.html)
