## Notes

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
