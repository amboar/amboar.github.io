---
title: Happenings in obmc-console
author: Andrew
---

[obmc-console][] is quite a slow-paced project relative to others in the OpenBMC
ecosystem, but recently I've merged quite a few changes. The bad news is not
all of them have have kept things in working order in the OpenBMC distro, so
let's look at what's happened, what's broken, and what we need to do to fix it.

[obmc-console]: https://github.com/openbmc/obmc-console

## A new coat of (build system) paint

`obmc-console` follows [the distro's C style guidelines][openbmc-docs-c-style]
but until now nothing enforced this, which makes it tricky to follow if you're
not used to kernel style or don't have the `astyle` invocation handy. C is also
notoriously difficult to write without invoking undefined-, unspecified- or
implementation-defined behaviour.

[openbmc-docs-c-style]: https://github.com/openbmc/docs/blob/master/CONTRIBUTING.md#c

A set of separate observations is that [the distro has a
preference][openbmc-prefers-meson] for using [meson][meson-build] as the build
system for its subprojects. Somewhat aligned with this preference is that the CI
scripts have much stricter requirements on projects using `meson`, with better
exploitation of [valgrind][], [sanitizers][google-sanitizers], [clang-tidy][]
and code-coverage than the support for other build systems such as `autotools`
or `cmake`. In addition to sanitizers, the CI scripts also support formatting of
the source code through [clang-format][].

[openbmc-prefers-meson]: https://github.com/openbmc/technical-oversight-forum/issues/4
[meson-build]: https://mesonbuild.com/
[valgrind]: https://valgrind.org/
[google-sanitizers]: https://github.com/google/sanitizers
[clang-tidy]: https://clang.llvm.org/extra/clang-tidy/
[clang-format]: https://clang.llvm.org/docs/ClangFormat.html

To better communicate expectations on others for their contributions to
`obmc-console` I did the work to [enable
`clang-format`][obmc-console-clang-format], [enable
`clang-tidy`][obmc-console-clang-tidy], and to then exploit the additional
features in CI [by converting the project from `autotools` to
`meson`][obmc-console-meson]. After [fixing up the `bitbake`
recipe][openbmc-obmc-console-build] I've [removed the `autotools` build
system][obmc-console-remove-autotools].

[obmc-console-clang-format]: https://gerrit.openbmc.org/c/openbmc/obmc-console/+/62605
[obmc-console-clang-tidy]: https://gerrit.openbmc.org/c/openbmc/obmc-console/+/62660
[obmc-console-meson]: https://gerrit.openbmc.org/c/openbmc/obmc-console/+/62575
[openbmc-obmc-console-build]: https://gerrit.openbmc.org/c/openbmc/openbmc/+/62744
[obmc-console-remove-autotools]: https://gerrit.openbmc.org/c/openbmc/obmc-console/+/62859

In parallel to this effort [another push began][openbmc-tof-26] to shift
straggling projects in the distro over to `meson`. With `obmc-console` now
converted we've knocked at least one project off that hit-list.

[openbmc-tof-26]: https://github.com/openbmc/technical-oversight-forum/issues/26

## An improvement to testability

In a recent post I outlined how we can [synthetically test the functionality of
`obmc-console` using `socat`][amboar-obmc-console-testing]. I won't say too much
about that here, other than to mention [the patches are still in
review][obmc-console-pty].

[amboar-obmc-console-testing]: notes/2023/05/02/testing-obmc-console-with-socat.md
[obmc-console-pty]: https://gerrit.openbmc.org/q/topic:pty

## A new set of requirements

IBM has quite a few internal changes to [bmcweb][]. As part of trying to reduce
the size of our downstream patch stack we're trying to upstream a bunch of
functionality.

[bmcweb]: https://github.com/openbmc/bmcweb/

### Restricting console connections to members of a specific group on the BMC.

[webui-vue]: https://github.com/openbmc/webui-vue

The ability to restrict consoles to a given group ended up being a matter of
[separating `obmc-console`'s configuration of
`dropbear`][openbmc-hostconsole-group] out from the generic use of `dropbear` as
OpenBMC's remote shell service.

[openbmc-hostconsole-group]: https://gerrit.openbmc.org/q/topic:hostconsole-group

### Support for multiple serial consoles in `bmcweb`

`obmc-console` has supported [concurrent console
servers][amboar-obmc-console-service-units] for some time now. While this
enabled exposing all of those servers onto the network via SSH using
[dropbear][] it fell short of providing appropriate support for exposing these
consoles via `bmcweb`. `bmcweb`'s general approach to handling the dynamic
presence of features in the system is to use [the
mapper][openbmc-docs-object-mapper] to find DBus objects implementing the
relevant interface, and to then use the basename of the resulting object paths
as a tag to identify the specific instance in the system. `obmc-console-server`
has had DBus-based functionality for some time now[^1], however, no ability to
connect to the console over DBus was provided.

[openbmc-docs-object-mapper]: https://github.com/openbmc/docs/blob/master/architecture/object-mapper.md

[^1]: To [expose the ability to control the UART baud rate][obmc-console-dbus-baud]

[obmc-console-dbus-baud]: https://gerrit.openbmc.org/c/openbmc/obmc-console/+/16619

[amboar-obmc-console-service-units]: notes/2023/03/31/exploiting-obmc-console-service-units-for-multiple-host-consoles.md
[dropbear]: https://matt.ucc.asn.au/dropbear/dropbear.html

## Trouble: Broken consoles

My effort to enable concurrent instances of `obmc-console-server` failed to
address the fact that the bus name claimed was not unique to the console, and
neither was the hosted object path. We've fixed this in [f9c8f6ca864a ("Fixed
broken dbus interface for multiple consoles")][obmc-console-unique-dbus-names]
by exploiting the `socket-id` for use as the final element of the connection
name and object path. However, that approach requires that a `socket-id` is
specified, which was not a requirement that existed previously.

While [ec7cab9378f5 ("Add socket-id for the first
console")][openbmc-add-socket-id] tried to address the new requirement it fell
short of fixing up the client-side configuration in some circumstances, and
`obmc-console` suppport over SSH is now broken for some consoles on a number of
platforms. Console access is also broken via `bmcweb` for all platforms, as the
connection logic inside `bmcweb` relied on a hard-coded name for
`obmc-console-server`'s Unix domain socket, which while still deterministic,
cannot be deduced without access to the `obmc-console`'s configuration files.

In short, this has become a bit of an integration disaster.

[obmc-console-unique-dbus-names]: https://gerrit.openbmc.org/c/openbmc/obmc-console/+/62901
[openbmc-add-socket-id]: https://gerrit.openbmc.org/c/openbmc/openbmc/+/62712

## Double Trouble: A false start in API design

Disregarding the integration woes above, we at least have the opportunity to
introduce a new DBus interface to salvage the broader problems encountered by
`bmcweb`. This interface will allow `bmcweb` or any other application to connect
to the console of the server instance hosting the object. The ideal interface
design would be something akin to how `obmc-console-client` operates - an
abstraction over the transport mechanism. It would be nice to provide an
interface design yields a file descriptor that `bmcweb` can immediately read and
write, e.g. a `Connect()` method that returns a file descriptor.

My initial pass at working out whether that was feasible didn't turn up anything
that would help. The way forward appeared as if we needed some mechanism for the
`Connect()` call to trigger `obmc-console-server` to connect to its own
`AF_UNIX` listening socket, which turns into a twisty mess of IO. The
implementation and maintenance complexity of that solution feels like it lands
on the wrong side of the trade-off analysis, and so it was abandoned.

Instead we landed on an interface that exposes a byte array containing the name
of the abstract Unix domain socket on which other processes could connect[^2].
This requires those processes [construct the socket][man7-man-2-socket] and call
[connect(2)][man7-man-2-connect] in addition to the mapper lookup and fetching
the property value. This approach was defined by
`xyz.openbmc_project.Console.Access` in [927d0930ca70 ("Add a new dbus interface
to get list of consoles")][phosphor-dbus-interfaces-consoles] and implemented in
[b14ca19cf380 ("Added new dbus interface to query console
info")][obmc-console-access-interface].

[^2]: The property is a byte array rather than a string due to a unique quirk of
    abstract Unix domain sockets, which is that [the first character of the
    socket's name is the `NUL` byte][man7-man-7-unix], and more generally,
    `NUL`s are not significant in the abstract socket name. The initial `NUL`
    would terminate the name as an empty string if directly interpreted as a C
    string.

[man7-man-7-unix]: https://man7.org/linux/man-pages/man7/unix.7.html
[man7-man-2-socket]: https://man7.org/linux/man-pages/man2/socket.2.html
[man7-man-2-connect]: https://man7.org/linux/man-pages/man2/connect.2.html
[phosphor-dbus-interfaces-consoles]: https://gerrit.openbmc.org/c/openbmc/phosphor-dbus-interfaces/+/61486
[obmc-console-access-interface]: https://gerrit.openbmc.org/c/openbmc/obmc-console/+/62496

It was in the process of [discussing the associated change to
`bmcweb`][discord-obmc-console-lookups] that [Jeremy][code-construct] swung by
to point out the existence of [socketpair(2)][man7-man-2-socketpair], the
missing piece of the puzzle that enables our idealised `Connect()` interface.

[discord-obmc-console-lookups]: https://discord.com/channels/775381525260664832/1083551792094249051/1103570006610038834
[code-construct]: https://codeconstruct.com.au/
[man7-man-2-socketpair]: https://man7.org/linux/man-pages/man2/socketpair.2.html

## Moving Forward

So we have a few things to fix. The first are pretty concrete and already in
progress:

1. Revert the patch that introduces the `xyz.openbmc_project.Console.Access`
   interface to `phosphor-dbus-interfaces`
2. Revert the implementation of the `xyz.openbmc_project.Console.Access`
   interface in `obmc-console`
3. Prototype `xyz.openbmc_project.Console.Access` as an interface with a
   `Connect()` method returning an immediately-usable file descriptor for the
   console.

Thankfully, the patch to `bmcweb` had not yet been merged. We will hold off on
that until we implement the `Connect()`-based interface in `obmc-console`, at
which point we can rework the `bmcweb` patch and then get it reviewed and
merged.

Less obvious is what we do to resolve the integration woes that have landed us
with broken consoles. Tentatively, I'm proposing that instead of requiring a
`socket-id` be defined by configuration, we provide a default value that can be
overridden by configuration. This will fix SSH console access for all platforms
in the tree that don't already supply a client configuration file. It will not
fix the `bmcweb` console, but that will be resolved by merging the patch
exploiting `mapper` lookups for `xyz.openbmc_project.Console.Access`.

## In Summary

It turns out software development is hard. It's hard to discover the useful
system calls for a design. It's hard to comprehend integration complexity, and
it's hard to test other people's platforms continue to behave correctly in the
face of change.

Something that I can do better is to lean on our other maintainers in the
OpenBMC ecosystem, to draw on their experience and point me in the right
direction. A quick chat with Jeremy would have avoided at least one of the pain
points outlined here. Working with others to exercise the configuration changes
might have also have helped identify the resulting breakage before we hit
submit.
