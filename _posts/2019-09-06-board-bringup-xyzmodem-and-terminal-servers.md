---
title: Board Bringup, [XYZ]MODEM and Terminal Servers
author: Andrew
---

You're doing bringup of a board or SoC or have got yourself in a tight spot;
you have no networking and are unable to write to local storage in the runtime
environment. How do you boot a custom firmware or kernel? If you're local the
obvious approach is to use an external tool to write the boot storage, but lets
add to the challenge and say you're doing this remotely. You have your board
hooked up to a terminal server, and are at the u-boot prompt.

The historic approach has been to use the [\*MODEM][0] protocols to do this
over serial, but we have the added impediment of the terminal server. Tools
like CKermit can drive the `lrzsz` tools [as demonstrated by the u-boot
documentation][1] and CKermit also supports e.g. telnet for the terminal
server, but modern Linux distributions (Ubuntu 19.04) don't ship CKermit and
naively attempting to build it from [source][3] with modern compilers fails.

Telnet is a bit sketchy for several reasons, but ignoring security we also have
the issue of data mangling. Thankfully our terminal server offers a "raw" port
where no mangling takes place. With this context, we can dodge building and
learning CKermit with some [socat][4] magic:

In u-boot, prepare the environment for data transfer via e.g. XMODEM:

```
# loadx 0x90000000
```

Then locally, once we have `socat` and `lrzsz` installed:

```
$ socat TCP:ts.example.com:2100 EXEC:"sx -b image"
```

What we should see is:

```
$ socat TCP:ts.example.com:2100 EXEC:"sx -b image"
Sending image, 16 blocks: Give your local XMODEM receive command now.
Bytes Sent:   2176   BPS:269

Transfer complete
$
```

And on the remote end:

```
# loadx 0x90000000
## Ready for binary (xmodem) download to 0x90000000 at 115200 bps...
CCxyzModem - CRC mode, 17(SOH)/0(STX)/0(CAN) packets, 1 retries
## Total Size      = 0x00000809 = 2057 Bytes
#
```

The key insight is that the [XYZ]MODEM tools use synchronization on `stdin` and
`stdout` to ensure data is being sent and received correctly. This means we
can't simply construct a shell pipeline like `sx -b image | socat
TCP:9.41.165.152:2114 -` and expect it to work. If we take this naive approach,
we see output like:

```
Retry 0: Timeout on pathname

Transfer incomplete
```

Or:

```
Sending s12983.lsz, 0 blocks: Give your local XMODEM receive command now.
Xmodem sectors/kbytes sent:   0/ 0kRetry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Timeout on sector ACK
Retry 0: Retry Count Exceeded

Transfer incomplete
```

[0]: https://en.wikipedia.org/wiki/ZMODEM
[1]: https://www.denx.de/wiki/publish/DULG/to-delete/UBootCmdGroupDownload.html#Section_5.9.5.3.
[2]: Ubuntu Dingo Disco
[3]: http://www.kermitproject.org/ck90.html#source
[4]: http://www.dest-unreach.org/socat/
