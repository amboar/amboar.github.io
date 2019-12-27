## Adventures in Debugging SQUASHFS Decompression Errors Under QEMU

For quite some time now the Witherspoon OpenBMC firmware has experienced
intermittent squashfs decompression failures when running under QEMU:

```
[   99.332020] SQUASHFS error: xz decompression failed, data probably corrupt
[   99.332338] SQUASHFS error: squashfs_read_data failed to read block 0x945a74
[   99.333368] SQUASHFS error: xz decompression failed, data probably corrupt
[   99.333696] SQUASHFS error: squashfs_read_data failed to read block 0x945a74
```

The squashfs in the Witherspoon BMC image contains the root filesystem, and due
to certain design choices the boot path mounts root directly from the kernel
rather than via an initrd. Thus if the squashfs fails in some manner we have no
recovery path other than falling back to the secondary flash chip on hardware,
or restarting QEMU. We don't run continuous integration tests against the
Witherspoon image, partly as a consequence of these failures and the resulting
lack of reliability.

But this problematic relationship runs both ways - QEMU recently released v4.2
and I wanted to update [openbmc/qemu](https://github.com/openbmc/qemu)
accordingly. As OpenBMC uses QEMU for CI I need to at least verify it boots the
images used for CI before pushing, but I want to verify other images that we
frequently use for development also continue to function.

The test process lead to the above errors with irritating frequency when
booting the Witherspoon image. Unfortunately no further information was
available and this lead to a number of questions that needed answers:

1. What data was being accessed?
2. How much data was being requested?
3. Where in the data did the corruption occur?
4. How was the corruption detected?

Curiously, we only seem to encounter this issue for Witherspoon, which is
unique among other platforms supported by OpenBMC as it uses the kernel's
Unsorted Block Images (UBI) subsystem for flash management (where other
platforms tend to just use a static flash layout).

As the failure was intermittent it was time for some automation. We can drive
qemu and the guest with an [`expect`](https://core.tcl-lang.org/expect/index)
script:

```shell
$ cat squashfs-errors.exp
#!/usr/bin/expect -f

spawn qemu-system-arm -M witherspoon-bmc -m 512 -drive file=primary,if=mtd,format=raw -net nic -net user,hostfwd=:127.0.0.1:2222-:22,hostfwd=:127.0.0.1:2443-:443,hostname=qemu,tftp=. -nographic

expect -timeout -1 "witherspoon login: "
send "root\n"
expect "assword: "
send "0penBmc\n"
expect "root@witherspoon:~# "
send "while ! systemctl is-system-running --no-pager | grep -q '\\(running\\|degraded\\)'; do systemctl is-system-running --no-pager; sleep 1; done\n"
expect -timeout -1 "root@witherspoon:~# "
send "journalctl -b -k --no-pager\n"
expect -timeout -1 "root@witherspoon:~# "
# Kill qemu
send "^Ax"
```

And then wrap that in shell script that files each run into "corrupt" or
"clean" bins (and contains a couple of tricks that we'll come to later):

```shell
$ cat angus-taylor
#!/bin/bash -x

for i in `seq 1 10`
do
	run=$(date '+%s')
	cp flash-witherspoon.clean primary
	./squashfs-errors${1}.exp | tee $run.log
	cp aspeed.fmc-ast2500.dump $run.aspeed.fmc-ast2500.dump
	if grep -q '\(SQUASHFS\|UBIFS\) error' $run.log
	then
		cp primary $run.img.corrupt
	else
		cp primary $run.img.clean
	fi
done
```

An interesting data-point was that inspecting the corrupt images with
[ubi_reader](https://github.com/jrspruitt/ubi_reader) failed to show any
significant issues with the UBI volumes, to the point that the squashfs could
be extracted from the "corrupt" image and unpacked without issue despite being
the source of the failures in the guest. This indicated that the problem might
involve some form of memory corruption, and thankfully we now have some really
nice tools to track this down in the form of [ASAN
](https://en.wikipedia.org/wiki/AddressSanitizer) - [at least,
almost](/notes/2019/12/27/arm-kasan.html). Running a KASAN kernel with and
without an ASAN-enabled QEMU build (`./configure --enable-sanitizers`) didn't
produce any ASAN-related failures, which indicated the corruption was likely
caused by a kernel or qemu behavioural error rather than poor memory-management
in either case.

So at this point it was time to dive into [debugging
UBI](/notes/2019/12/22/debugging-ubi-corruption-at-boot.html), which turned up
the insight that across multiple QEMU runs the corrupted data was consistently
the first 4 bytes of varying length accesses that were performed at arbitrary
offsets inside a Physical Erase Block (PEB). That is, the size and offset of
the access had no effect on the fact that it was consistently the first four
bytes that were incorrectly read while the remainder of the data was as
expected. In debugging mode the UBI IO system raises errors in e.g. the write
path by writing the data to flash, then reading it back and comparing the
read-back data against the input buffer.

The fact that the data trailing the first four bytes was always correct raised
the question of whether we would receive the correct data if we performed a
second read of the first four bytes, and if so, whether there was any pattern
to the incorrect data from the first read.

To capture patterns in space and in time I needed to capture the first word of
all accesses to flash. The amount of data generated in this case is irritating
to handle, which lead to [paravirtualized debugging of the flash
controller](/notes/2019/12/22/abusing-qemu-device-models.html) to funnel the
guest data out onto the host filesystem:

### QEMU
```patch
diff --git a/hw/ssi/aspeed_smc.c b/hw/ssi/aspeed_smc.c
index c8713f3e3347..4910dbd34694 100644
--- a/hw/ssi/aspeed_smc.c
+++ b/hw/ssi/aspeed_smc.c
@@ -1254,6 +1254,10 @@ static void aspeed_smc_write(void *opaque, hwaddr addr, uint64_t data,
         s->regs[addr] = DMA_FLASH_ADDR(s, value);
     } else if (s->ctrl->has_dma && addr == R_DMA_LEN) {
         s->regs[addr] = DMA_LENGTH(value);
+    } else if (addr == (0x20 / 4)) {
+        uint32_t ldata = data;
+        ssize_t rc = write(s->dumpfd, &ldata, sizeof(ldata));
+        (void)rc;
     } else {
         qemu_log_mask(LOG_UNIMP, "%s: not implemented: 0x%" HWADDR_PRIx "\n",
                       __func__, addr);
@@ -1372,6 +1376,11 @@ static void aspeed_smc_realize(DeviceState *dev, Error **errp)
     if (s->ctrl->has_dma) {
         aspeed_smc_dma_setup(s, errp);
     }
+
+    snprintf(name, sizeof(name), "%s.dump", s->ctrl->name);
+    s->dumpfd = open(name, O_WRONLY | O_CREAT | O_TRUNC, S_IRUSR | S_IWUSR);
+    if (s->dumpfd < 0)
+        perror("open");
 }
 
 static const VMStateDescription vmstate_aspeed_smc = {
diff --git a/include/hw/ssi/aspeed_smc.h b/include/hw/ssi/aspeed_smc.h
index 6fbbb238f158..4b15e0a3ccf8 100644
--- a/include/hw/ssi/aspeed_smc.h
+++ b/include/hw/ssi/aspeed_smc.h
@@ -117,6 +117,8 @@ typedef struct AspeedSMCState {
 
     uint8_t snoop_index;
     uint8_t snoop_dummies;
+
+    int dumpfd;
 } AspeedSMCState;
 
 #endif /* ASPEED_SMC_H */
```

### Kernel
```patch
diff --git a/drivers/mtd/spi-nor/aspeed-smc.c b/drivers/mtd/spi-nor/aspeed-smc.c
index ff367c70001d..b6c798559896 100644
--- a/drivers/mtd/spi-nor/aspeed-smc.c
+++ b/drivers/mtd/spi-nor/aspeed-smc.c
@@ -615,6 +616,18 @@ static ssize_t aspeed_smc_read(struct spi_nor *nor, loff_t from, size_t len,
        memcpy_fromio(read_buf, chip->ahb_base + from, len);
 
 out:
+
+       if (len >= 4) {
+               u32 a = (read_buf[3] << 24) | (read_buf[2] << 16) | (read_buf[1] << 8) | (read_buf[0]);
+               u32 b;
+
+               memcpy_fromio(&b, chip->ahb_base + from, sizeof(b));
+
+               writel(from, chip->ctl + 0x10);
+               writel(cpu_to_be32(a), chip->ctl + 0x10);
+               writel(cpu_to_be32(b), chip->ctl + 0x10);
+       }
+
        return len;
 }
```

The kernel patch results in tuples of three words written to the host-side file
through our sneaky register: (address, first-read, second-read). We can view
the file  with some helpful `hexdump` hackery (`hexdump`'s formatting DSL is
brazenly esoteric):

```shell
$ cat hd
#!/bin/sh 

echo "offset  |  address   first   second"
echo ---------+---------------------------
hexdump -e '/1 "%08_ax"" | "' -e '3/4 "%08x ""\n"' "$@" |
        grep -v -e '^\*$' -e '\([a-f0-9]\+\) \1$'
```

The `grep` eliminates cases where the first-read value matches the second-read
value - these are only interesting if there isn't some other strange pattern in
the data. Here's some output from an example run that ended in decompression
failure:

```shell
$ ./hd data.5/1576840009.aspeed.fmc-ast2500.dump
 offset  |  address   first   second
---------+---------------------------
00000000 | 00080000 08308de5 55424923
00017754 | 00016c00 00000000 00000d00
00019194 | 0004ec00 00000000 ffffffff
00019b54 | 0169bf80 02020202 00020000
```

The ASCII representation of `0x55424923` is `UBI#`, which is the expected magic
for the UBI superblock. Surprisingly, not only was the UBI superblock magic
consistently "corrupt", but its corrupted value was consistently `0x08308de5`.
Note this is at offset `0x0` in the `.dump` file produced by the
paravirtualised flash controller debugging, so this is the first access that
the kernel's SPI-NOR driver has performed against the BMC flash after
completing timing calibration (which doesn't use the regular access routines).
The fact that the access's flash offset is `0x80000` [indicates that it's
driven by UBI parsing the on-flash
structures](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/arm/boot/dts/aspeed-bmc-opp-witherspoon.dts?h=v5.4#n216),
which fits neatly with needing to mount the (squashfs) root that lives in a
static UBI partition. The implication of _this_ fact is that either the
calibration routine has left the flash controller in an odd state, or that
QEMU's flash (controller) models have bugs. The implication of the fact that
_further_ corruption intermittently occurs after the initial case indicates
that can't be just calibration that is at fault, and that it might be time to
start inspecting the QEMU models.

At this point it all became a blur of `printk()`, `printf()` and `gdb` at 2am
Saturday morning, which eventually resulted in the realisation that QEMU's
m25p80 flash chip model was stuck in read-state and was ignoring the
command/address phase of the kernel's attempt to access the data at `0x80000`
on the flash. The "corrupt" value of `0x08308de5` was not some random heap
memory value that we deterministically ended up reading, but rather data
straight from a deterministic flash address - `0x1a1008` - due to the behaviour
of the calibration routine in the flash-controller driver presumably some bugs
in the flash controller model.

QEMU's modelling of the ASPEED flash controller's command-mode [asserts the
chip-select, performs the 1/2/4 byte access and then deasserts the
chip-select](https://git.qemu.org/?p=qemu.git;a=blob;f=hw/ssi/aspeed_smc.c;h=f0c7bbbad302d2a7037d7a6889e89c6d9433adab;hb=b0ca999a43a22b38158a222233d3f5881648bb4f#l716).
Deasserting the chip-select resets the m25p80 model's state machine back ready
for the command/address phase, and if you recall we found that data subsequent
to the first 4-byte word was the correct data. As it turns out, [explicitly
deasserting the chip-select before asserting it upon executing a command-mode
read](https://patchwork.ozlabs.org/patch/1214193/) gives us the stability we've
sort for so long, which suggests we failed to deassert it correctly elsewhere.
