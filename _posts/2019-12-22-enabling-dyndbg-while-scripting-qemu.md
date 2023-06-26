---
title: Enabling Linux Dynamic Debug Statements when Scripting QEMU
author: Andrew
---

Trying to debug intermittent data corruption under QEMU can be fairly painful,
especially if you're trying to reproduce with stock boot images. I happened to
be in this dark corner recently and wanted to enable some extra debug output in
the provided kernel. Thankfully Linux supports enabling dynamic debug
statements on the kernel commandline, though depending on which statements you
want and how you're trying to enable them this can be a straightforward or like
trying to run a Rube Goldberg machine in reverse.

There were several hurdles in my case:

1. I was trying to automate reproduction of the bug and so was scripting the
   guest boot process under QEMU using `expect(1)`
2. The expect script was driving the u-boot shell to indirectly set `bootargs`
   via a u-boot script
3. I needed to enable dynamic debug statements for a pattern containing spaces

As it turns out there are several relevant points that fall out as a
consequence:

1. u-boot does not sensibly handle escaping of nested quotes. Also it will eat
   all your single-quotes nested inside double-qoutes.
2. As it turns out, dyndbg will handle octal representation of characters just
   fine, so you can encode your spaces as `\040` to avoid badly split strings
   arguments on the kernel commandline.
3. You'll still have to do a lot of escaping to wedge this all into an `expect`
   script that does the right thing.

In my case the result ended up looking like:

```
send "setenv bootargs \$bootargs debug systemd.log_target=null dyndbg=\"\\\"format UBI\\\\040DBG\\\\040wl +p\\\"\"\n"
```
