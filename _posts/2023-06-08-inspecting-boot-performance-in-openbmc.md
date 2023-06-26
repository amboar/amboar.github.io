---
author: Andrew
---

# Inspecting Boot Performance in OpenBMC

OpenBMC uses [systemd][] as a system management daemon. While this raises some
eyebrows given the environment, it's what we have. At least it provides
reliability and familiarity as we disregard the perceived complexity. With
systemd comes an opportunity to use [systemd-bootchart][man-1-systemd-bootchart]
for [measuring boot-time behaviour of the system][lwn-bootchart]. While it's
relatively easy to use in general, some of the details of the OpenBMC boot
process can get in the road.

[systemd]: https://systemd.io/
[man-1-systemd-bootchart]: https://manpages.debian.org/jessie/systemd/systemd-bootchart.1.en.html
[lwn-bootchart]: https://lwn.net/Articles/299483/

The first detail is that it's very unlikely that anyone will include the
`systemd-bootchart` binary in their platform's distribution by default. To deal
with that we're going to have to build the image ourselves, and if we're doing
that, we can take the easy road to plonk it in:

```
$ cd openbmc/openbmc
$ . setup p10bmc
$ echo 'IMAGE_INSTALL:append = " systemd-bootchart"' >> conf/local.conf
```

`systemd-bootchart` requires at `CONFIG_SCHEDSTATS=y` be set in the kernel
config. This wasn't the case as it stood, so I've [fixed that
up][meta-phosphor-systemd-bootchart]. Another problem I encountered was that the
`/init` script for eMMC-based systems in OpenBMC didn't honor `init=` on the
kernel commandline, [so I've fixed that too][meta-phosphor-mmc-init].

[meta-phosphor-systemd-bootchart]: https://gerrit.openbmc.org/c/openbmc/openbmc/+/63929
[meta-phosphor-mmc-init]: https://gerrit.openbmc.org/c/openbmc/openbmc/+/63928

With those pieces in place we can now `bitbake obmc-phosphor-image` and boot it
up. However, before doing any analysis, from experience it's worth configuring
bootchart to reduce the sample rate, extend the sample count, and tell it to log
the process command lines rather than just PIDs. For example, on the BMC, I
used:

```
# cat <<EOF >>/etc/systemd/bootchart.conf
Samples=1200
Frequency=10
Cmdline=yes
EOF
#
```

With that done we're set up, we just need to configure `systemd-bootchart` to
run. This is best done by stopping at the u-boot prompt and updating `bootargs`
without writing the environment back to flash[^1]:

```
ast# setenv bootargs console=ttyS4,115200n8 init=/lib/systemd/systemd-bootchart
ast# boot
```

And with that, you should have bootchart `.svg` files appear in `/run/log`!

[^1]: Some quirk that I didn't investigate meant `systemd-bootchart` wasn't
    installed at `/usr/lib/systemd/systemd-bootchart` like the documentation
    suggests it should be
