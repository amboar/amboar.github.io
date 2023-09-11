---
title: Disabling the Lenovo P14s Touchscreen
author: Andrew
---

New job, new laptop, and I no longer an M1 MacBook Pro. This time around I have
a Lenovo P14s, which unfortunately comes with a touchscreen. Touchscreens are
terrible, so let's disable it.

This is relatively straight-forward using udev. The first step is to identify
the device we care about:

```
$ sudo dmesg | grep -i touchscreen
[    2.092460] input: ELAN901C:00 04F3:2EDE Touchscreen as /devices/pci0000:00/0000:00:15.1/i2c_designware.1/i2c-2/i2c-ELAN901C:00/0018:04F3:2EDE.0002/input/input9
```

Having identfied that, we inspect the udev attributes exposed for us to match
against:

```
$ udevadm info -a -p /sys/class/input/input9

  ...

  looking at device '/devices/pci0000:00/0000:00:15.1/i2c_designware.1/i2c-2/i2c-ELAN901C:00/0018:04F3:2EDE.0002/input/input9':
    KERNEL=="input9"
    SUBSYSTEM=="input"
    DRIVER==""
    ATTR{capabilities/abs}=="3273800000000003"
    ATTR{capabilities/ev}=="1b"
    ATTR{capabilities/ff}=="0"
    ATTR{capabilities/key}=="400 0 0 0 0 0"
    ATTR{capabilities/led}=="0"
    ATTR{capabilities/msc}=="20"
    ATTR{capabilities/rel}=="0"
    ATTR{capabilities/snd}=="0"
    ATTR{capabilities/sw}=="0"
    ATTR{id/bustype}=="0018"
    ATTR{id/product}=="2ede"
    ATTR{id/vendor}=="04f3"
    ATTR{id/version}=="0100"
    ATTR{inhibited}=="0"
    ATTR{name}=="ELAN901C:00 04F3:2EDE"
    ATTR{phys}=="i2c-ELAN901C:00"
    ATTR{power/async}=="disabled"
    ATTR{power/control}=="auto"
    ATTR{power/runtime_active_kids}=="0"
    ATTR{power/runtime_active_time}=="0"
    ATTR{power/runtime_enabled}=="disabled"
    ATTR{power/runtime_status}=="unsupported"
    ATTR{power/runtime_suspended_time}=="0"
    ATTR{power/runtime_usage}=="0"
    ATTR{properties}=="2"
    ATTR{properties}=="2"

  ...
```

The `ATTR{id/*}` properties seem like a good way to constrain the match. From
there, the key observation is that input devices [ present the `inhibited`
attribute in sysfs][linux-sysfs-input-inhibited], which we can frob via udev.

[linux-sysfs-input-inhibited]: https://docs.kernel.org/input/input-programming.html#inhibiting-input-devices

Using these properties we can write a udev rule to automatically inhibit the
device for us when the driver is bound to the device:

```
$ cat /etc/udev/rules.d/99-inhibit-touchscreen.rules 
ATTR{id/bustype}=="0018", ATTR{id/product}=="2ede", ATTR{id/vendor}=="04f3", ATTR{id/version}=="0100", ATTR{inhibited}="1"
```
