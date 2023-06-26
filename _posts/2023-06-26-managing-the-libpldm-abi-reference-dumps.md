---
title: Managing the libpldm ABI Reference Dumps
author: Andrew
---

A recent change to [libpldm][] [added three visibility
classes][amboar-libpldm-abi-control] to control its [Application Binary
Interface (ABI)][mike-hearn-writing-shared-libraries]. The approach is fairly
crude but also fairly effective. As outlined, the primary motivation is to allow
us to fix (break) new Application Programmer Interfaces (APIs) without getting
hung up on existing users. The `testing` ABI visibility class allows functions
to exist in the library but remain out of reach for general use until we're
confident that their API is reasonable.

[libpldm]: https://github.com/openbmc/libpldm
[amboar-libpldm-abi-control]: _posts/2023-06-06-deprecated-stable-and-testing-abis-for-libpldm.md
[mike-hearn-writing-shared-libraries]: https://plan99.net/~mike/writing-shared-libraries.html#backwards-compat

The existence of the `stable` visibility class implies that we shouldn't break
the existence or behaviour any of its functions. This promise is only as good as
our ability to measure it. The ideal measurement is that any stable API and ABI
breaks are detected by Continuous Integration (CI). To that end I integrated
support for [abi-compliance-checker][] [into the build
system][libpldm-abi-compliance].

[abi-compliance-checker]: https://lvc.github.io/abi-compliance-checker/
[libpldm-abi-compliance]: https://github.com/openbmc/libpldm/commit/953bc8c1a4e9f21981897d06b71159e403dfff1b

However, this integration requires the existence of reference dumps against
which CI will measure proposed patches. The existence of reference dumps implies
we also need to maintain them as functions come and go across library releases.
This post aims to outline exactly how and when we should go about updating these
reference dumps.

We find the reference dumps under `libpldm`'s `abi/` directory, namespaced by
the host architecture[^1] and compiler:

[^1]: In the sense of [GCC's configure terms][gcc-configure-terms]
[gcc-configure-terms]: https://gcc.gnu.org/onlinedocs/gccint/Configure-Terms.html

```
$ git ls-files abi/
abi/aarch64/gcc.dump
abi/x86_64/gcc.dump
$
```

As we can see the dumps currently cover the library's ABI as produced by GCC on
`aarch64` and `x86_64`. Support for `aarch64` is required because that's the
architecture of my current laptop. The `x86_64` reference dump is captured
because the architecture is reasonably popular, but crucially, the dominant
architecture used by the project's CI.

Setting the reference dumps aside, each build also generates its own ABI dump.
These per-build dumps are currently [written out as `current.dump` in the root
of the build directory][libpldm-build-abi-dump-location]. The role of
`abi-compliance-checker` is to test `current.dump` against the appropriate
reference dump for the build.

[libpldm-build-abi-dump-location]: https://github.com/openbmc/libpldm/blob/291da1952b7459b890b2de742774e9dfc1b28cec/meson.build#L106-L116

A corrolary of this is that the reference dumps can be updated by copying a
build's `current.dump` into the appropriate location under the `abi/` directory.
Needless to say, the build's host architecture (again, with respect to [^1]) and
compiler suite need to be taken into account. If you are unsure, these details
are summarised in the output of `meson setup ...`:

```
$ meson setup build
...
C compiler for the host machine: ccache cc (gcc 13.1.1 "cc (GCC) 13.1.1 20230614 (Red Hat 13.1.1-4)")
C linker for the host machine: cc ld.mold 1.11.0
C++ compiler for the host machine: ccache c++ (gcc 13.1.1 "c++ (GCC) 13.1.1 20230614 (Red Hat 13.1.1-4)")
C++ linker for the host machine: c++ ld.mold 1.11.0
Host machine cpu family: aarch64
Host machine cpu: aarch64
...
```

A critical issue is that any need to update a reference dump requires that all
reference dumps be updated. Further, it may be the case that you do not have
access to machines to cover all of the relevant host architectures. For
Debian-derived distributions you can set yourself up with [the multiarch
instructions][debian-wiki-multiarch]. Fedora is a bit more of a challenge,
however [I've covered one possible approach][amboar-fedora-multiarch] if it is
your distro of choice. From there it's a matter of choosing a build directory
naming scheme such as `builds/${arch}/${compiler}` to keep your build
configurations separate[^2]:

[debian-wiki-multiarch]: https://wiki.debian.org/Multiarch/HOWTO
[amboar-fedora-multiarch]: _posts/2023-06-15-cross-architecture-userspace-workflow-on-fedora-38.md

[^2]: Again, for a short intro to meson's cross-files see my post covering
      [cross-architecture work on Fedora][amboar-fedora-multiarch]

```
$ meson setup --cross-file=... builds/${arch}/${compiler}
$ meson compile -C builds/${arch}/${compiler}
```

And once the build completes:

```
$ cp builds/${arch}/${compiler}/current.dump abi/${arch}/${compiler}.dump
```

### When should I update the reference ABI dumps?

Currently there are three circumstances:

1. When tagging a new release of the library
2. When stabilising a function[^3]
3. When deleting a function from the `deprecated` visibility class[^4]

[^3]: That is, moving a function from the `testing` visibility class to the
      `stable` visibility class by changing its visibility annotation from
      `LIBPLDM_ABI_TESTING` to `LIBPLDM_ABI_STABLE`

[^4]: Note that [functions must never be deleted directly from the `stable`
      visibility class][libpldm-working-with-libpldm].

[libpldm-working-with-libpldm]: https://github.com/openbmc/libpldm#working-with-libpldm

The ABI dumps capture the library version in their metadata. This drives the
need to update the dumps whenever a new release tag is created.

Stabilising a function is the explicit promise not to break its API or ABI going
forward. It is thus criticial that it be integrated into the reference dumps so
this promise can be upheld by CI.

Finally, once a function is deleted there's no need to continue carrying its
description in the referece dumps.

And with that, hopefully the lifecycle of the reference dumps is now clear!
