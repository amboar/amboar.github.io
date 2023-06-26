---
title: An opkg-based OpenBMC development workflow
author: Andrew
---

Previously [I talked about the mechanics of how I develop bits and pieces of
userspace for OpenBMC][1]. What I will discuss this time is an alternative
flow that replaces the use of `devtool deploy-target` with `opkg`.

### `opkg`?

Under the covers `bitbake` leverages one of several package managers to deploy
the recipe artifacts into a rootfs via [Package Feeds][3], one of them being
`opkg`. As of [605c37cb989a ("bitbake: Use IPK packaging for rootfs
assembly")][4] OpenBMC prefers the use of `ipk` packages, the format handled by
`opkg`.

`opkg` grew out of [OpenWrt][2] as a lightweight package manager implemented for
embedded environments and its development appears to now have been taken up by
the Yocto project. By contrast to `apt` and `dnf` for `.deb` and `.rpm` package
handling respectively, `opkg` is relatively straight-forward to configure and
control and doesn't require much (any) extra effort to work with
cross-architecture package archives.

### But what are the benefits?

There are several alternatives for deploying your changes when testing your
work:

1. Use the SDK to build and then manually locate and `scp` artifacts out of your
   application build tree
2. Use the `devtool modify ...` / `devtool deploy-target ...` workflow [outlined
   in the previous post][1]
4. Use `bitbake` to generate a complete firmware image and deploy

The first requires installing an appropriate SDK for the target environment.
Installing the hulking SDKs is annoying, building them is worse, and manually
locating binaries to copy out of your build tree is tedious. This gets more
fiddly and annoying with the more dependencies you have beteween your artifacts
(e.g. a co-req with a systemd unit file change, or some shared libraries).

This second improves on the first because it doesn't require an SDK - bitbake
builds you the correct environment so your binaries are always accurate for
the target. However, `devtool deploy-target` doesn't perform any dependency
resolution, leaving this approach short for deploying things like tracing and
debugging tools that you don't necessarily bake into the production image.

The third tends to have a much higher latency than the earlier two, though does
provide all the consistency needed.

The approach described in this post using `opkg` tries to solve all of the
problems outlined above:

1. No tedious hunting and copying of specific artifacts, and integration of
   configuration and unit files into the target environment is automated
2. Dependency resolution and installation is automatically applied
3. Only deploys the specific packages required for the configuration, nothing
   more, nothing less.

However, its still a trade-off, as implementing the `opkg` approach does involve
some complexity.

### Exposing the bitbake package archive and configuring `opkg`

So by including `opkg` in the target image we can exploit the package feeds that
bitbake intrinsically generates for development purposes on the BMC. Here I'll
cover how to piece things together into an effective workflow.

Generally your OpenBMC build machine and target environment are not the same. As
such we need a way to expose the bitbake package feeds on the build machine for
`opkg` to access from the target environment. The most pragmatic way to do this
is to use [Python's built-in webserver from the `http` module][5]:

```sh
$ python3 -m http.server
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Exploiting this for the package archive we run the following from the root of
our bitbake build tree:

```sh
$ python3 -m http.server --directory tmp/deploy/ipk
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Now we need to tell `opkg` how to reach our package archive. For this I use the
awful shell function that follows. Paste this into your `~/.bashrc` and try to
forget about the details:

```sh
gen_opkg_conf () 
{ 
    local ipk_url="$1";
    local build_dir="$2";
    cat "${build_dir}"/tmp/deploy/ipk/**/Packages |
        grep --color=auto '^Architecture' |
        sort -u |
        awk "BEGIN { arch_prio=1 } { printf(\"arch %s %d\n\", \$2, arch_prio); arch_prio += 5; printf(\"src/gz %s %s/%s\n\", \$2, \"${ipk_url}\", \$2) }" |
        sort
}
```

The key pieces it needs as arguments are:

1. A URL that can be used to reach the webserver we're hosting out of the
   bitbake package archive directory, and
2. The path to the root of your bitbake build directory.

An example invocation looks like the following. The `192.168.0.10` IP address
must be replaced with the address of your build machine, which must also be
reachable from the BMC:

```sh
$ bitbake package-index
...
$ gen_opkg_conf http://192.168.0.10:8000 .
arch all 1
arch armv7ahf-vfpv4d16 6
arch foo 11
arch x86_64-nativesdk 16
src/gz all http://192.168.0.10:8000/all
src/gz armv7ahf-vfpv4d16 http://192.168.0.10:8000/armv7ahf-vfpv4d16
src/gz foo http://192.168.0.10:8000/foo
src/gz x86_64-nativesdk http://192.168.0.10:8000/x86_64-nativesdk
```

The [`bitbake package-index` command][8] is necessary to ensure the package
archive is in a usable state before we parse the data with `gen_opkg_conf`.

Paste the output from `gen_opkg_conf ...` into a file in the target environment.
In this example we'll use the path `~/opkg.conf`. At this point `opkg` is ready
to go, save for some minor details like read-only root filesystems.

### Preparing the root filesystem

Often embedded environments make use of read-only root filesystems. This makes
using a package manager difficult, as it will want to deploy the packages there.
To fix this we can make use of tmpfs overlays. The [`overlay` script][6] makes
life a bit easier here, and we'll use it in the snippets below. Copy it onto the
target.

In addition to deploying the package artifacts `opkg` maintains a database to
track package states. As we're deploying the packages into tmpfs mounts, we
should also make the package database disappear if the tmpfs overlays disappear.
To fix it all up on the target in one fell swoop, run:

```
# mkdir -p /var/{lib,cache}/opkg &&
> rm -rf /var/{lib,cache}/opkg/* &&
> ./overlay add /var/{lib,cache}/opkg /usr /lib /bin /sbin
```

### Building your package and updating the archive

At this point we're ready to install things onto the target using `opkg`.
However, if we're doing development iterations on a package we need to make sure
the package archive contains our work. Assuming the package has already been set
up using `devtool modify ...`, build using the `do_package_write_ipk` task.
Using `dbus-sensors` as an example [like we did last time][1]:

```sh
$ bitbake dbus-sensors:do_package_write_ipk &&
> bitbake package-index
```

The `do_package_write_ipk` task drops the package into the package archive, and
then the `package-index` recipe takes care of fixing up the archive metadata.

The `package-index` recipe needs to be explicitly executed in a subsequent build
as it has no dependency on the `do_package_write_ipk` task.

Both commands need to be executed each time you wish to deploy your work.
   
### (Re-)Installing packages using `opkg`

`opkg` is configured, we're exposing our package archive via HTTP and our
tmpfs overlays are in-place in our target environment. We're ready to install
some packages!

By default `opkg` looks for its configuration at `/etc/opkg.conf`. We placed
ours at `~/opkg.conf` in our target environment, so all our `opkg` invocations
need to account for that. `opkg` is told where its configuration lives using the
`-f` or `--conf` option.

With that we can prime the package metadata and install something.

At least, almost. As a once-off before we begin installing things we need to
clean up some breakage from the `base-files` package, which [removes `/var/lock`
in its preinstall script][7]:

```
# opkg -f ~/opkg.conf update &&
> opkg -f ~/opkg.conf install --force-overwrite base-files &&
> systemd-tmpfiles --create --exclude-prefix=/dev
```

With that fixed up, to iterate on our package we use:

```
# opkg -f ~/opkg.conf update &&
> opkg -f ~/opkg.conf install --volatile-cache --force-reinstall dbus-sensors
```

The `--volatile-cache` option ensures that subsequent installs don't try to
reuse a stale package. The `--force-reinstall` option ensures that if the
package is already installed it is first removed, then the updated package
installed.

From here it's a matter of iterating between rebuilding using the `bitbake`
steps outlined earlier, and then reinstalling the package using the commands
outlined above.

### Debugging your work

Now that we have a working `opkg`, you can have debug symbols on the target too!
`bitbake` generates several additional packages for each recipe: a `...-dbg`
package containing the debug symbols for the corresponding binary package, and a
`...-src` package containing the sources used to generate binary and `...-dbg`
packages.

To figure out why `nvmesensor` is crashing, we can:

```
# opkg -f ~/opkg.conf install gdb gcc-runtime-dbg
# opkg -f ~/opkg.conf install --volatile-cache --force-reinstall dbus-sensors-dbg
# gdb -q nvmesensor
...
```

Similarly, you can install `valgrind`, `perf` or any other tool that's useful
for understanding what's going on. If the tool doesn't appear in the package
archive, a `bitbake ...:do_package_write_ipk && bitbake package-index` takes
care of it!

### Searching for packages with `opkg`

Finally, sometimes the package names aren't intuitive and you need to search.
This is handled by `opkg`'s `find` subcommand, which can take a glob:

```
# opkg -f ~/opkg.conf find 'phosphor-power*'
phosphor-power - 1.0+git0+c94ccc0945-r1
phosphor-power-control - 1.0+git0+c94ccc0945-r1
phosphor-power-dbg - 1.0+git0+c94ccc0945-r1
phosphor-power-dev - 1.0+git0+c94ccc0945-r1
phosphor-power-ibm-ups - 1.0+git0+c94ccc0945-r1
phosphor-power-psu-monitor - 1.0+git0+c94ccc0945-r1
phosphor-power-regulators - 1.0+git0+c94ccc0945-r1
phosphor-power-sequencer - 1.0+git0+c94ccc0945-r1
phosphor-power-src - 1.0+git0+c94ccc0945-r1
phosphor-power-utils - 1.0+git0+c94ccc0945-r1
```

In this case the recipe implements a split package arrangement. Note that the
`...-dbg` and `...-src` packages are relative to the recipe name, not the split
package name!

[1]: /notes/2022/01/13/openbmc-development-workflow.html
[2]: https://openwrt.org/
[3]: https://www.yoctoproject.org/docs/3.1/overview-manual/overview-manual.html#package-feeds-dev-environment
[4]: https://github.com/openbmc/openbmc/commit/605c37cb989a95c02633fcb93efb45102781b4bb
[5]: https://docs.python.org/3/library/http.server.html
[6]: https://github.com/openbmc/openbmc-tools/blob/master/overlay/overlay
[7]: https://git.yoctoproject.org/poky/tree/meta/recipes-core/base-files/base-files_3.0.14.bb?h=yocto-3.4.4#n72
[8]: https://docs.yoctoproject.org/ref-manual/tasks.html#do-package-index
