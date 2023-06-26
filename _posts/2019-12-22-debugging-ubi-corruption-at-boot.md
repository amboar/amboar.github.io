---
author: Andrew
---

## Debugging UBI Corruption At Boot

Unsorted Block Images is a magic Linux kernel subsystem for handling wear
levelling, data integrity management and dynamic partitioning of raw flash
devices. It's magic in the sense that it does a lot of work under the covers,
frequently shuffling data around to uphold the desirable properties of the
subsystem.

For a long time the several of us had been observing intermittent data
corruption when running the Witherspoon OpenBMC firmware under the
witherspoon-bmc QEMU platform model. Witherspoon is somewhat unique in the list
of supported platforms in OpenBMC in that it exploits UBI for flash layout
management rather than using a simple static layout. This brings flexibility
but also complexity, and it's always worse when complex things go wrong, which
they did.

UBI as it turns out has a handy set of debug options, though these are only
exposed via debugfs, which implies that you've actually managed to boot your
system. Unfortunately for me this was not the case so this patch came in handy:

```
diff --git a/drivers/mtd/ubi/build.c b/drivers/mtd/ubi/build.c
index d636bbe214cb..84be669af1fb 100644
--- a/drivers/mtd/ubi/build.c
+++ b/drivers/mtd/ubi/build.c
@@ -883,6 +883,11 @@ int ubi_attach_mtd_dev(struct mtd_info *mtd, int ubi_num,
        ubi->dev.class = &ubi_class;
        ubi->dev.groups = ubi_dev_groups;
 
+       ubi->dbg.disable_bgt = 1;
+       ubi->dbg.chk_gen = 1;
+       ubi->dbg.chk_io = 1;
+       ubi->dbg.chk_fastmap = 1;
+
        ubi->mtd = mtd;
        ubi->ubi_num = ubi_num;
        ubi->vid_hdr_offset = vid_hdr_offset;
```

This force enables various debug options for all UBI devices upon being
attached to the associated MTD device. The `chk_io` output was particularly
useful to me:

```
[  118.049917] ubi0 error: ubi_io_write: self-check failed for PEB 353:49024, len 256
[  118.050462] ubi0: data differ at position 0
[  118.050764] ubi0: hex dump of the original buffer from 0 to 256
[  118.051798] 00000000: 00 02 00 00 00 00 f8 91 52 1a 3a 19 af c9 90 9e 00 00 00 00 00 00 11 00 00 00 60 61 be 00 00 6e  ........R.:...............`a...n
[  118.052711] 00000020: 31 18 10 06 d8 da f9 d8 45 04 00 00 00 00 00 00 2e 07 00 00 01 00 00 00 6e 00 00 00 2e 00 00 20  1.......E...............n...... 
[  118.053553] 00000040: 00 00 00 00 00 00 00 00 00 10 00 00 01 00 00 00 00 01 38 9f 00 00 00 00 00 00 e1 46 dc 37 7b 70  ..................8........F.7{p
[  118.054678] 00000060: 8a a0 d0 a2 00 80 00 07 a2 16 59 8b c3 29 96 12 b8 a4 98 01 08 00 0a 4c af 38 4d c3 de 8c 00 a5  ..........Y..).........L.8M.....
[  118.055516] 00000080: bc 01 06 32 75 57 fc e8 c1 86 2e a0 dc 01 07 10 30 4f 05 c3 88 9a 0c 98 a9 bc 03 07 81 1c 38 bf  ...2uW..........0O............8.
[  118.056341] 000000a0: eb fd 5a 37 78 c0 bc 01 08 ff 9d 48 6c a5 5c 28 b3 28 24 01 80 02 06 c5 3d a9 35 31 71 ec 7a 80  ..Z7x......Hl.\(.($.....=.51q.z.
[  118.057166] 000000c0: dc 01 07 7a c4 8d 1d 88 43 09 cb d0 25 bc 03 07 62 ef 75 3e d2 12 0d 76 18 26 bc 01 06 f8 9a 47  ...z....C...%...b.u>...v.&.....G
[  118.057972] 000000e0: dc 13 7a 51 ca 68 dc 01 06 c8 64 34 b7 08 cf 62 27 c0 dc 01 07 a7 ba c8 09 00 1c 32 34 18 27 bc  ..zQ.h....d4...b'..........24.'.
[  118.058851] ubi0: hex dump of the read buffer from 0 to 256
[  118.059258] 00000000: 02 02 02 02 00 00 f8 91 52 1a 3a 19 af c9 90 9e 00 00 00 00 00 00 11 00 00 00 60 61 be 00 00 6e  ........R.:...............`a...n
[  118.060058] 00000020: 31 18 10 06 d8 da f9 d8 45 04 00 00 00 00 00 00 2e 07 00 00 01 00 00 00 6e 00 00 00 2e 00 00 20  1.......E...............n...... 
[  118.060850] 00000040: 00 00 00 00 00 00 00 00 00 10 00 00 01 00 00 00 00 01 38 9f 00 00 00 00 00 00 e1 46 dc 37 7b 70  ..................8........F.7{p
[  118.061638] 00000060: 8a a0 d0 a2 00 80 00 07 a2 16 59 8b c3 29 96 12 b8 a4 98 01 08 00 0a 4c af 38 4d c3 de 8c 00 a5  ..........Y..).........L.8M.....
[  118.062452] 00000080: bc 01 06 32 75 57 fc e8 c1 86 2e a0 dc 01 07 10 30 4f 05 c3 88 9a 0c 98 a9 bc 03 07 81 1c 38 bf  ...2uW..........0O............8.
[  118.063271] 000000a0: eb fd 5a 37 78 c0 bc 01 08 ff 9d 48 6c a5 5c 28 b3 28 24 01 80 02 06 c5 3d a9 35 31 71 ec 7a 80  ..Z7x......Hl.\(.($.....=.51q.z.
[  118.064398] 000000c0: dc 01 07 7a c4 8d 1d 88 43 09 cb d0 25 bc 03 07 62 ef 75 3e d2 12 0d 76 18 26 bc 01 06 f8 9a 47  ...z....C...%...b.u>...v.&.....G
[  118.065151] 000000e0: dc 13 7a 51 ca 68 dc 01 06 c8 64 34 b7 08 cf 62 27 c0 dc 01 07 a7 ba c8 09 00 1c 32 34 18 27 bc  ..zQ.h....d4...b'..........24.'.
```
