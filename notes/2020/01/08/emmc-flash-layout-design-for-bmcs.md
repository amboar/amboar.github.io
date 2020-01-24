## Secure and Robust eMMC Flash Layout Design for BMCs

Recently a few of us were interested in designing an eMMC flash layout that
allowed for secure boot of a BMC's userspace while also catering to robustness
across updates. This post covers a script I developed to road-test secure
rootfs eMMC images under QEMU. The script appears at the end after the
discussion of how we implement it.

Addressing the requirements, robustness across updates at least requires
something akin to partitioning the storage into A and B devices for the rootfs,
where rootfs updates are written to whichever partition isn't the running
session's root. As for secure boot, the kernel needs some way to verify content
from the root filesystem before executing it. Handily, Linux already has a
couple of approaches to achieving this, one being the [Integrity Measurement
Architecture
(IMA)](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/security/integrity/ima/Kconfig?h=v5.4#n4)
and another being
[dm-verity](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/verity.rst?h=v5.4).
As a big handwave, IMA performs as a much finer-grain implementation of
dm-verity, as dm-verity applies to an entire read-only filesystem on a
block-basis.

As it turns out, read-only filesystems are fairly popular in embedded
scenarios and the device-mapper functionality fits nicely with our requirement
for robustness, so dm-verity was chosen as the way forward. We will be applying
device-mapper and dm-verity to squashfses of the root filesystems.

The kernel's device-mapper infrastructure essentially functions as a
translation layer mapping logical to physical blocks on a storage device. This
abstraction enables a whole host of behaviours to be transparently implemented
in the kernel, such as [disk
encryption](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/dm-crypt.rst?h=v5.4)
and
[RAID](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/dm-raid.rst?h=v5.4)
or testing and robustness concepts like
[unreliable](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/dm-dust.txt?h=v5.4)
[IO](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/dm-flakey.rst?h=v5.4).
Our interests are rather more boring though, and we'll just make use of the
basic
[linear](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/linear.rst?h=v5.4)
functionality to implement our partitions and dm-verity to provide security.

Working from the inside-out, lets get our squashfs rootfs set up with
dm-verity. This is done via
[`veritysetup`](http://man7.org/linux/man-pages/man8/veritysetup.8.html), part
of the [cryptsetup repo](https://gitlab.com/cryptsetup/cryptsetup). Here we
meet a choice - `veritysetup` can store the metadata either appended to the data
or in another device altogether. We'll chose the simpler route and just append
the metadata to the squashfs root to save configuring ourselves another
device-mapper device. Assuming our rootfs is 7843840 bytes, we would issue:

```sh
$ veritysetup format \
     --hash sha256 \
     --data-block-size 4096 \
     --hash-block-size 4096 \
     --hash-offset 7843840 \
     rootfs.squashfs rootfs.squashfs
```

The data and hash block sizes in this case are in terms of the page size of the
target system to avoid overhead in the kernel. The metadata must not become
embedded in the last data block as this will cause an integrity check fail, so
we must position the metadata as aligned on the block boundary subsequent to
the end of the data. As it turns out, `7843840 mod 4096 = 0` so the align-up
operation is a no-op in this example.

The `veritysetup` invocation outputs essential information such as the root
hash and a randomly chosen salt value:

```sh
$ veritysetup format --hash sha256 --data-block-size 4096 --hash-block-size 4096 --hash-offset 7843840
VERITY header information for rootfs.squashfs
UUID:                   d3efa8ec-5668-4b8d-826e-1b2f1122cc5a
Hash type:              1
Data blocks:            1915
Data block size:        4096
Hash block size:        4096
Hash algorithm:         sha256
Salt:                   dddbe28bbd4b9def1fe48f69b912971cdae51295aea3afec5dcafae5ca9a8d1d
Root hash:              a68000e541d0980b89cd289d3b4325a1d1972ce90bbdcf8e434dbc2bfcf7e996
```

For secure-boot to function we must take the provided root hash, sign it with a
private key recognised by the kernel, then inject the root hash, its signature
and the salt value into the system's boot environment.

A this point we're finished with dm-verity, so lets now look at how to describe
linear targets to the kernel. One way is to use an initrd and a script calling
[`dmsetup`](http://man7.org/linux/man-pages/man8/dmsetup.8.html). Another way
is to describe the linear mapping table [on the kernel
commandline](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/dm-init.rst).
Using the commandline approach requires fewer cogs in the machine, so this is
the direction we will go.

Here's an example that creates a linear device called `a` over a contiguous
region at the start of `/dev/mmcblk0`, and lets say we want a device that
exactly fits our dm-verity-protected image. With the dm-verity metadata
appended to our rootfs it is now 7913472 bytes in size. Beware that the
`dm-mod.create` offset and size parameters are described in terms of 512 byte
sectors as opposed to the 4096 byte blocks used for `veritysetup`: We need to
make sure our output image size is aligned-up to a multiple of both the block
and sector sizes to avoid corruption. As it turns out `7913472 mod 4096 = 0`
and `4096 mod 512 = 0` so we're safe, and our 7913472 bytes represent 15456
sectors.

On the kernel commandline we can now create a linear device `a` for our
verity-protected image. Using the [construction parameters for the linear
target](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/linear.rst?h=v5.4#n9):

```
Parameters: <dev path> <offset>
    <dev path>:
     Full pathname to the underlying block-device, or a
        "major:minor" device-number.
    <offset>:
     Starting sector within the device.
```

We put together the following:

```
dm-mod.create="a,,,rw, 0 15456 linear /dev/mmcblk0 0"
```

The creation of a `b` partition that takes on the role of accepting rootfs
updates is a matter of defining another linear entry in the device-mapper table
that doesn't physically overlap with the blocks assigned to `a` on
`/dev/mmcblk0`.

The device `a` device is exposed as `/dev/dm-0` by the kernel. With that
information at hand we can describe a dm-verity device nested in the linear
device. Using the [construction parameters for the verity
target](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/admin-guide/device-mapper/verity.rst?h=v5.4#n9):

```
    <version> <dev> <hash_dev>
    <data_block_size> <hash_block_size>
    <num_data_blocks> <hash_start_block>
    <algorithm> <digest> <salt>
    [<#opt_params> <opt_params>]
```

And substituting the values in from the `veritysetup` output above we arrive
at:

```
dm-mod.create="root,,,ro, 0 15320 verity 1 /dev/dm-0 /dev/dm-0 4096 4096 1915 1916 sha256 a68000e541d0980b89cd289d3b4325a1d1972ce90bbdcf8e434dbc2bfcf7e996 dddbe28bbd4b9def1fe48f69b912971cdae51295aea3afec5dcafae5ca9a8d1d"
```

Note that the hash start block value is one more than the number of data
blocks, and the 15320 value represents the number of sectors occupied by the
squashfs (i.e. excluding the verity metadata). `/dev/dm-0` is listed for both
the `dev` and `hash_dev` as we configured `veritysetup` to append the metadata
to the data blocks.

Finally, to get the result to boot in QEMU we need to deal with some quirks of
the MMC stack. QEMU assumes that the backing file for MMC storage devices is a
multiple of a size recorded in the Card Specific Data register, and different
MMC modes lead to varying expected sizes. The cheapest way out is to ensure the
image size is a multiple of 512 kilobytes.

To tie that all together, here is a script to automate all the steps above and
produce a QEMU commandline to boot the result under an ASPEED AST2600 EVB
machine:

```sh
$ cat smash
#!/bin/sh

# SPDX-License-Identifier: Apache-2.0
# Copyright 2019 IBM Corp.

SRC_IMG=${SRC_IMG:-rootfs.squashfs}
DST_IMG=$(mktemp)
MMC_IMG=${MMC_IMG:-image.bin}

SECTOR_SIZE=512
BLOCK_SIZE=4096
CSD_SIZE=$((1 << (9 + 9 - 1 + 2)))

cleanup() {
	rm -f $DST_IMG
}

trap cleanup EXIT

align_up() {
	local offset=$1
	local size=$2

	echo $(((($offset + ($size - 1)) / $size) * $size))
}

verity_get_meta() {
	local needle="$1"
	local haystack="$2"

	echo "$haystack" | grep "$needle" | cut -d: -f2 | tr -d '[ \t]'
}

rm -f $MMC_IMG

dd if="$SRC_IMG" of="$DST_IMG" 2> /dev/null

VERITY_HASH_OFFSET=$(align_up $(stat --format=%s $SRC_IMG) $BLOCK_SIZE)
VERITY_HASH_BLOCKS=$(($VERITY_HASH_OFFSET / $BLOCK_SIZE))

VERITY_ALGO=sha256
VERITY_META="$(veritysetup format \
	--hash $VERITY_ALGO \
	--data-block-size $BLOCK_SIZE \
	--hash-block-size $BLOCK_SIZE \
	--hash-offset $VERITY_HASH_OFFSET \
	"$DST_IMG" "$DST_IMG")"

VERITY_SALT=$(verity_get_meta Salt "$VERITY_META")
VERITY_ROOT=$(verity_get_meta Root "$VERITY_META")

A_SECTORS=$(($(align_up $(stat --format=%s $DST_IMG) $BLOCK_SIZE) / $SECTOR_SIZE))
A_LINEAR="a,,,rw, 0 $A_SECTORS linear /dev/mmcblk0 0"
ROOT_SECTORS=$(($(align_up $(stat --format=%s $SRC_IMG) $BLOCK_SIZE) / $SECTOR_SIZE))
ROOT_VERITY="root,,,ro, 0 $ROOT_SECTORS verity 1 /dev/dm-0 /dev/dm-0 $BLOCK_SIZE $BLOCK_SIZE $VERITY_HASH_BLOCKS $(($VERITY_HASH_BLOCKS + 1)) $VERITY_ALGO $VERITY_ROOT $VERITY_SALT"

# Make an appropriately sized image for MMC
fallocate -l $(align_up $(stat --format=%s $DST_IMG) $CSD_SIZE) $MMC_IMG
dd if="$DST_IMG" of=$MMC_IMG conv=notrunc 2> /dev/null

echo export SMASH_DM_MOD_CREATE="'dm-mod.create=\"$A_LINEAR; $ROOT_VERITY\"'"
echo export SMASH_MMC_IMG=$MMC_IMG
echo

>&2 echo Example QEMU commandline:
>&2 echo
>&2 echo qemu-system-arm -M ast2600-evb -m 1024 -kernel zImage -dtb aspeed-ast2600-evb.dtb -nographic -drive file=sd1.img,if=sd,format=raw -drive file=sd2.img,if=sd,format=raw -drive file=\${SMASH_MMC_IMG},if=sd,format=raw -append '"console=ttyS4,1152008n earlyprintk debug $SMASH_DM_MOD_CREATE root=/dev/dm-1"'
```
