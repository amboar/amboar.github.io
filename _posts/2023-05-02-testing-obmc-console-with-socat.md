---
title: Testing obmc-console with socat
author: Andrew
---

This is a bit of a gross hack. However, it serves to demonstrate a way to test
[the `obmc-console` stack][obmc-console] without requiring integration into a
BMC and booting its host (or some equally tedious arrangement).

[obmc-console]: https://github.com/openbmc/obmc-console

First off, [some patches are required to make our lives
easier][openbmc-gerrit-topic-pty].

[openbmc-gerrit-topic-pty]: https://gerrit.openbmc.org/q/topic:pty

With the patches in place we can [dummy up an environment using `socat`][socat].
To do so we invoke it to set up a PTY, connecting `stdio` to the current shell,
and ask it to make a symlink for the slave side.  Because the world is upside
down already anyway (*gestures broadly at everything*), we then attach
`obmc-console-server` to the slave side of the PTY and then use
`obmc-console-client` as usual. In this configuration the `socat` process is
acting as e.g. a host console.

[socat]: http://www.dest-unreach.org/socat/

This arrangement is best done using three separate shells sessions for sanity,
all in the same working directory:

### "Host Console" session

```
$ socat PTY,link=pts,wait-slave -,rawer
```

### `obmc-console-server` session

```
$ obmc-console-server -i test $(realpath pts)
```

### `obmc-console-client` session

```
$ obmc-console-client -i test
```

## Result

From here, any data entered in the "Host Console" session should appear in the
`obmc-console-client` session, and vice versa.

The test environment can be cleaned up by terminating the `obmc-console-server`
process (e.g. with `Ctrl-C`). The two other processes will also exit as a
consequence.
