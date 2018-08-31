## General Architecture of Hostboot

2018-08-19

[Hostboot](https://github.com/open-power/hostboot) is split into several parts,
in terms of the artefacts generated and their roles.

Before we dive into hostboot, it's worth a recap of the boot process of an
OpenPOWER machine:

1. The Baseboard Management Controller (BMC, specifically
   [OpenBMC](https://github.com/openbmc/openbmc)) applies power to the chip
   bringing up components in the standby domain. Critically, this includes the
   Self Boot Engine (SBE).
2. The BMC uses the Flexible Service Inteface (FSI) to send the magic boot
   sequence to the SBE. This kicks off the boot process inside the POWER chip.
3. The SBE loads and executes code from its internal SEEPROM. The SEEPROM code
   loads hostboot from the system firmware PNOR into the processor cache using
   [firmware cycles](https://en.wikipedia.org/wiki/Low_Pin_Count#Supported_peripherals)
   on the [LPC bus](https://en.wikipedia.org/wiki/Low_Pin_Count). It then kicks
   off execution on the first core.
4. Hostboot initialises the internal buses and remaining cores, trains memory,
   and chain-loads the subsequent firmware payload (on OpenPOWER systems,
   [skiboot](https://github.com/open-power/skiboot)).

Step 3. is what we'll be exploring here - the pieces and execution sequence of
hostboot.

At a high level, hostboot is its own cache-contained operating system. It has
the notion of a kernel and userspace, task management, virtual memory and
a virtual filesystem layer. Virtual memory and the virtual filesystem layer in
particular are necessary as hostboot is too big to fit into the 10MiB of cache
available to it (HBI, a partition we'll explore below, weighs in at over
14MiB). As such it uses demand-paging to bring code and data in and out of the
cache (from the PNOR) as necessary.

Lets look at the pieces in terms of their split into the on-flash partitions:

### HBBL - Hostboot Bootloader

The hostboot bootloader ultimately lives in the SBE's SEEPROM, however it is
provided as part of the PNOR image. HBBL's job is to simply find the Hostboot
Base (HBB) partition from the PNOR, cryptographically verify it, load the
binary into the processor's cache and kick off execution on the first core.

Hostboot doesn't just work with BMC-based platforms: It also works with IBM's
enterprise FSP systems. The bootloader needs to work across both types of
systems, and so its means to access the PNOR must also generalise over both.
This requirement is met in a fairly straight-forward way as the bootloader does
not require any capability to write to the flash; reading HBB off the PNOR is a
matter of locating it and issuing the necessary LPC firmware read cycles.

In terms of the hostboot source tree, HBBL [lives in its own isolated directory:
`src/bootloader`](https://github.com/open-power/hostboot/tree/e07f0c96e66b4dc7986076db0a9f3ebe02902361/src/bootloader).

### HBB - Hostboot Base

The hostboot base partition contains the hostboot kernel and the services
necessary to read _and_ write to the PNOR. Depending on the design of the
service processor (BMC or FSP), writes may need additional support in order to
make their way onto the flash. The necessary logic for writes is captured in
the base's PNOR driver.

The services provided by HBB and initialised in its execution [are determined
by the top-level
`makefile`](https://github.com/open-power/hostboot/blob/e07f0c96e66b4dc7986076db0a9f3ebe02902361/src/makefile#L140)
and the [base initialisation service task
list](https://github.com/open-power/hostboot/blob/e07f0c96e66b4dc7986076db0a9f3ebe02902361/src/usr/initservice/baseinitsvc/initsvctasks.H#L42).

As mentioned above hostboot implements demand-paging - the services in the base
image are the exception, these are not pageable due to their nature.

### HBI - Hostboot Extended Image

Like HBB, the content of HBI [is defined by the top-level
`makefile`](https://github.com/open-power/hostboot/blob/master/src/makefile#L152)
and the initialisation sequence in the [extended initialisation service task list](https://github.com/open-power/hostboot/blob/e07f0c96e66b4dc7986076db0a9f3ebe02902361/src/usr/initservice/extinitsvc/extinitsvctasks.H#L43). Unlike HBB the content of HBI is pageable.

Execution out of HBI is where the real work begins, in the form of "ISTEPs".
Each ISTEP performs a unique set of actions to advance initialisation of the
chip, to the point where memory is available and the chip is thermally
protecting itself via the On-Chip Controller (OCC). The content of these steps
is _not_ unique to hostboot: The ISTEP actions are shared between hostboot and
other chip development tools that allow the bringup team to boot a chip in a
non-self-hosted way. Put another way, hostboot is the framework that enables
self-hosting chip initialisation for POWER platforms.

On the subject of ISTEPs we can return to the discussion above of HBBL. HBBL is
shipped as part of the PNOR image, but ultimately the SBE executes HBBL from
its SEEPROM. During the initialisation process, an ISTEP checks the PNOR's HBBL
against the content of the SBE's SEEPROM. If they are different, hostboot
updates the SEEPROM with the newer HBBL. Obviously this is a dangerous
operation - if the new HBBL is broken then applying the update will brick the
chip. To mitigate this risk the SBE has two "sides" to its SEEPROM, side 0 and
side 1; hostboot writes the new HBBL to side 0 and requests a reboot. The BMC
then reboots the processor. If the boot process fails, the BMC will reconfigure
the SBE to boot from side 1 and try again, otherwise if the reboot succeeds
side 1 is updated to match the content of side 0. From this point forward the
SBE will boot with the new HBBL.

Finally, a number of other partitions are necessary for correct operation of
hostboot (HBEL, HBD, HBRT, HB\_VOLATILE) but are ancillary to HBBL, HBB and HBI
covered, so for the moment we'll leave exploration of hostboot's architecture
here.

Hostboot series:

* [Hacking Hostboot](notes/2018/08/17/hacking-hostboot.md)
* [Debugging Hostboot the Hard Way](notes/2018/09/03/debugging-hostboot.md)
