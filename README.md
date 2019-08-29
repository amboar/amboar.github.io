## Notes

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
