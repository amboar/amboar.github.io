## Debugging Hostboot The Hard Way

Maybe you've seen Hostboot dump out something like the following on the console:

```
 36.97093|ISTEP 10. 2 - host_slave_sbe_update
 37.53543|================================================
 37.53863|Error reported by sbe (0x2200) PLID 0x9000002E
 37.54187|  SBE image for current chip EC was not found in PNOR
 37.54188|  ModuleId   0x01 SBE_FIND_IN_PNOR
 37.54189|  ReasonCode 0x2208 SBE_EC_NOT_FOUND
 37.54190|  UserData1  CHIP EC : 0x0000002200000004
 37.54191|  UserData2  PNOR Section ID : 0x0000000153424500
 37.54195|------------------------------------------------
 37.54196|  Callout type             : Procedure Callout
 37.54197|  Procedure                : EPUB_PRC_SP_CODE
 37.54361|  Priority                 : SRCI_PRIORITY_HIGH
 37.54362|------------------------------------------------
 37.54363|  Hostboot Build ID: hostboot-4bd54cb-p8235b67/hbicore.bin
 37.54364|================================================
```

It gives a rough description of what the problem is, but there's a significant
lack of context. In extreme cases Hostboot will terminate the boot process,
which then requires an out-of-band approach to understanding what has gone
wrong. So what can we do?

Thankfully we can make use of Hostboot's `printk` (kernel, sort of) and trace
(general) buffers, and do some abusive things with global address space. We'll cover all of these approaches below.

The printk and trace buffers both stash a lot of useful information, and in
either case we can use [pdbg](https://github.com/open-power/pdbg) to extract
them from the host via the BMC. I will cover printk first, as that's much less
of a moving target than the trace buffers.

### `printk`

[Joel](https://jms.id.au/) has [already written up how to go about
it](https://shenki.github.io/Hostboot-kernel-log/), though as a TL;DR, you can
run the following (beware the printk buffer address may be subject to change,
see Joel's post for how to track down the exact location):

```
# pdbg -p 0 getmem 0x8040278 $((20 * 1024)) > /tmp/hb-log
```

And the result looks like:

```
# cat /tmp/hb-log
Booting Hostboot kernel...


CPU=Nimbus  PIR=0
Valid BlToHbData found at 0x80E2000
Version=900000007
lpc=6030000000000, xscom=603FC00000000
iv_lpc=6030000000000, iv_xscom=603FC00000000, iv_data=0x454f8
HRMOR = 8000000
Hostboot base image ends at 0x9A000...
PageManager end of preserved area at 0XE7000
PageManager page table offset at 0X100000
729 pages.
Debug @ 0x3c4598
Starting init!
Bringing up VFS...done.
Initializing modules.
        Initing module libtrace.so...done.
        Initing module liberrl.so...done.
        Initing module libdevicefw.so...done.
        Initing module libscom.so...done.
        Initing module libxscom.so...done.
        Initing module libinitservice.so...done.
        Initing module libsecureboot_base.so...done.
        Initing module liblpc.so...done.
        Initing module libpnor.so...done.
        Initing module libvfs.so...done.
        Initing module libsio.so...done.
Modules initialized.
init_main: Starting Initialization Service...
InitService entry.
Addr [0], magic[50415254] checksum[0]
forced to true - l_isActiveTOC=1
SECUREBOOT::enabled() state:0
ExtInitSvc entry.
ast2400: VUART config in process
VUART: config SUCCESS
UART: Device initialized.
...
```

Note that the content cannot be trusted if hostboot has handed over to the
PAYLOAD firmware; it is well within the rights of subsequent firmwares to do as
they see fit with the memory ranges in which hostboot was executing. The
content is only relevent if hostboot has unexpectedly stopped (usually to to a
critical error shutting down the ISTEP loop).

Further, the printk buffer is not a ring-buffer - any `printk()` output
exceeding the default buffer size (20KiB) is discarded. So while the printk
buffer is generally a low-noise environment that's trivial to access it is not
a good place to stick debug output if there is going to be lots of it. A better
place to stick high-volume debug output is into the trace buffers.

### Tracing

However, the trace buffers come with a new set of limitations. Hostboot is
architected as a microkernel-based environment with practically all services
running outside the kernel bar message-passing IPC primitives, scheduling and
memory management. Critically, control of the trace buffers is managed by a
userspace tracing service, and so any output required prior to the
instantiation of the trace service must go elsewhere (perhaps into the printk
buffer). This is a pretty narrow constraint though, and most code can ignore it.

[Various macros are available to record trace output](https://github.com/open-power/hostboot/blob/6c30bcf8975825c394a6eb6f687535a9c52c74e8/src/include/usr/trace/interface.H),
with the two significant ones being `TRACFCOMP()` and `TRACDCOMP()`.
`TRACFCOMP()` is the general-purpose component tracing function, while
`TRACDCOMP()` is for debug messages which we can afford to compile out when
building releases.

Unlike the printk buffer the trace buffers are much more dynamic, which
increases their utility but decreases the ease with which we can extract them.
Fortunately hostboot has helper scripts for dumping the traces, and we can
invoke them with a script like so:

```
#!/bin/sh

OP_BUILD="$1"
: ${HB_TIP:=custom}
export HOSTBOOTROOT="${OP_BUILD}"/output/build/hostboot-${HB_TIP}/
export PROJECT_ROOT="${HOSTBOOTROOT}"
export HB_DFRAME="${HOSTBOOTROOT}"/src/build/debug/
${HB_DFRAME}/hb-dump-debug --symsmode=usemem --file=$2 --tool=Trace
```

But for this to work we need something from which to dump the traces. In this
case, we need to extract all 10MiB of hostboot's memory image to pass to the
`hb-dump-debug` tool.

```
# pdbg -p 0 getmem $((128 * 1024 * 1024)) $((10 * 1024 * 1024)) > /tmp/hb.bin
```

[Only](https://github.com/openbmc/linux/commit/5968d6cf37d26fc8ea7c29305b0757cc1331e717)
[recently](https://github.com/openbmc/linux/commit/8cc4caf7b5d15a23f2d9f93ab89e8b4cee7a90c2),
and then [only for some](https://github.com/openbmc/linux/commit/c71662c749dc66a542e717dbd51fefab995c9455)
[systems](https://github.com/openbmc/openbmc/commit/bb3592fc96940a50a9bd8c617a8f5209420a531b)
in concert with [further hacks to
pdbg](https://github.com/ozbenh/pdbg/compare/d4ddacf32977a516f8c636f451ad5df2a27acef9...aaeb1e937d161005622801b0de0793b7d4c980b7),
the extraction process has been reduced to just over a minute, but for any
other configuration the extraction process could take several hours which is
obviously not ideal.

When it works, you'll get output along the lines the following, representing a
fairly fine-grained view of hostboot's execution process:

```
...
182.39529|INITSVC|>>IStepDispatcher::sendSyncPoint
182.39531|INITSVC|I>sendSyncPoint: Istep mode or no base services, returning
182.39531|INITSVC|<<IStepDispatcher::sendSyncPoint
182.39533|ISTEPS_TRACE|callShutdown finished, shutdown = 0x1230000.
182.39534|INITSVC|doShutdown(i_status=0000000001230000)
182.39535|INITSVC|doShutdown> status=0000000001230000
182.39536|INITSVC|notify priority=0x0, queue=0x3c70d8
182.60675|INITSVC|notify priority=0x10, queue=0x3c7228
182.60677|ERRL|I>Got an error log Msg - Type: 0x00000033
182.60678|ERRL|I>Shutdown event received
182.60681|INITSVC|notify priority=0x10, queue=0x3cd158
182.63122|ERRL|I>Shutdown event processed
182.60683|INITSVC|notify priority=0x13, queue=0x207228
182.60684|ISTEPS_TRACE|Pre-Shutdown Inits event received
...
```

And when it doesn't work, you'll see:

```
$ ./hb-dump-debug --symsmode=usemem --file=hb.bin --tool=Trace
Cannot find trace daemon and/or service.
Died at Hostboot/Trace.pm line 87.
```

If the latter case is where you find yourself, don't panic, you still have
options if your problem can be reproduced.

### Tracing to the Console

To work around the failures in extracting traces you can choose to build
hostboot such that it outputs the traces on the console, and from there you can
capture your own logs. There are several options around the console output, all
documented in the [tracing
HBconfig](https://github.com/open-power/hostboot/blob/6c30bcf8975825c394a6eb6f687535a9c52c74e8/src/usr/trace/HBconfig),
and the change required largely amounts to the following fix to your hostboot
build config:

```
-unset CONSOLE_OUTPUT_TRACE
+set CONSOLE_OUTPUT_TRACE
```

Booting this configuration is fairly painful - such a lot of output slows down
the boot process significantly. This may cause you to fall afoul of boot
failure timers, so be wary if things stop working suddenly.

### Dealing with crashes

Sometimes tasks or the kernel will crash out, and you'll need to find why. When
it occurs, the kernel will walk the stack and print the return pointer for each
frame as a real address into the printk buffer:

```
K:Backtrace for 135:
  <-0x80070<-0x40633428<-0x406349C4<-0x4063D294<-0x40637E5C<-0x5BD98<-0x5B700<-0x405F86BC<-0x405E
B58C<-0x400074F4<-0x2588
```

While there are no symbols available, we can make use of another hostboot build
artifact to track down the problem - `hbicore.list.bz2` - which is a compressed
archive of the disassembled linked image with source references. Using the
addresses provided by the stack trace it is possible to work your way through
the file and the call-stack that lead to the issue. Note that as they are
return addresses they're pointing to instruction after that which triggered the
failue.

### Debugging hacks

Finally, while the above covers a decent amount of ground, sometimes it is not
enough. When it's not enough there is still some wiggle room to pull off some
hacks and work around hostboot's convoluted approach to execution. There are at
least two facts are relevant:

1. The kernel [maps](https://github.com/open-power/hostboot/blob/6c30bcf8975825c394a6eb6f687535a9c52c74e8/src/kernel/ptmgr.C#L876) itself such that you can [execute kernel functions like
   `printk()`](https://github.com/open-power/hostboot/blob/6c30bcf8975825c394a6eb6f687535a9c52c74e8/src/kernel/basesegment.C#L84) from [userspace](https://github.com/open-power/hostboot/blob/6c30bcf8975825c394a6eb6f687535a9c52c74e8/src/kernel/segmentmgr.C#L106)
2. There is only one address space - tasks do not have distinct virtual address
   spaces

Regarding 1. this makes it easy to avoid the complication of tracing where
necessary - you can dump entries into the printk buffer from any execution
context.

The power of 2. is that instead of using the message-passing primitives and
"abstraction" provided by the kernel you can influence the execution of
separate tasks by adding, modifying and testing some globals in the appropriate
places. A handy (temporary) use of this technique to limit the console trace
output to specific ISTEPs that are causing you grief, which minimises the time
to boot and also the amount of noise generated by the traces:

```
diff --git a/src/include/usr/trace/interface.H b/src/include/usr/trace/interface.H
index f6be30fef231..6f7e2125d6ee 100644
--- a/src/include/usr/trace/interface.H
+++ b/src/include/usr/trace/interface.H
@@ -377,6 +377,8 @@ tracepp replaces trace_adal_hash() with hash value and reduced format string
 #define TRAC_INIT(des, name, sz, type...) \
                 TRAC_INIT_LINE(des, name, sz, __LINE__, ## type)
 
+extern volatile bool traceConsoleOutput;
+
 namespace TRACE
 {
 /******************************************************************************/
diff --git a/src/usr/trace/service.C b/src/usr/trace/service.C
index 260f71e2b4f9..87836787499f 100644
--- a/src/usr/trace/service.C
+++ b/src/usr/trace/service.C
@@ -44,9 +44,11 @@
 #include <util/sprintf.H>
 #include <debugpointers.H>
 
+volatile bool traceConsoleOutput = false;
 
 namespace TRACE
 {
     Service::Service()
     {
         iv_daemon = new DaemonIf();
@@ -106,6 +108,7 @@ namespace TRACE
         }
 
         #ifdef CONFIG_CONSOLE_OUTPUT_TRACE
+        if ( traceConsoleOutput )
         {
             va_list args;
             va_copy(args, i_args);
```

Followed by, for example:

```
diff --git a/src/usr/isteps/istep21/call_host_start_payload.C b/src/usr/isteps/istep21/call_host_start_payload.C
index c5a650a10346..0ce865ed6613 100644
--- a/src/usr/isteps/istep21/call_host_start_payload.C
+++ b/src/usr/isteps/istep21/call_host_start_payload.C
@@ -291,6 +291,7 @@ errlHndl_t calcTrustedBootState()
 
 void* call_host_start_payload (void *io_pArgs)
 {
+    traceConsoleOutput = true;
     errlHndl_t  l_errl  =   nullptr;
 
     IStepError l_StepError;
```

### Conclusion

To debug hostboot the hard way you're going to need access to `pdbg`, the
hostboot source and the build tree for the image running on the system. Using
`pdbg` you can extract the printk buffer, or pull out the traces after
extracting the entire live hostboot image.

If `pdbg` is too slow for the job you can fall back to dumping the trace output
to the console, and if you're still getting grief from that you can resort to
hacks like abusing `printk()` from userspace or influencing execution of tasks
via globals.

Hostboot series:

* [Hacking Hostboot](/notes/2018/08/17/hacking-hostboot.html)
* [General Architecture of Hostboot](/notes/2018/08/19/hostboot-architecture.html)
