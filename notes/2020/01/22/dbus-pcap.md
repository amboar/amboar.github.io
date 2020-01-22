## `dbus-pcap`: Debugging OpenBMC with `busctl capture`

[@jessfraz](https://twitter.com/jessfraz) recently wrote an [ACM Queue article
on BMCs and the availability of open-source BMC
firmware](https://queue.acm.org/detail.cfm?id=3378404). OpenBMC gets a mention,
though the article also points out that it's modular design interconnected with
D-Bus "makes the BMC software more complex to debug, audit, and put into
production."

So, how do we debug complex interactions between OpenBMC components? Assuming
that we can reproduce the issue, one approach is to increase the log levels in
each daemon of interest. This requires that all the right logging has been put
in place a-priori, or that the issue can be reproduced in a development
environment where we can make changes to the system.

Another approach that doesn't have the log level limitations is to record the
conversations between the daemons - this is something that is possible with the
centralised message bus and existing tools provided by systemd. Using `busctl
capture` we can generate a pcap of all D-Bus traffic which we can post-process
to extract timing metadata and view both method calls and signals.

At least, in theory. There is a D-Bus decoder for Wireshark, but I found
filtering was awkward, especially when tracing method calls. To fix that I've
written
[dbus-pcap](https://github.com/openbmc/openbmc-tools/blob/master/amboar/obmc-scripts/dbus-pcap/dbus-pcap),
a script that parses D-Bus pcaps with [scapy](https://scapy.net/) and allows
specification of [standard D-Bus
matches](https://dbus.freedesktop.org/doc/dbus-specification.html#message-bus-routing-match-rules)
to filter the data. Additionally, the tool can output each D-Bus message
encoded as JSON, which is handy for post-processing via
[jq](https://stedolan.github.io/jq/) if the D-Bus match support doesn't provide
enough control over the output.

Here are some example invocations:

```sh
dbus-pcap -h
usage: dbus-pcap [-h] [--json] [--no-track-calls]
                 file [expressions [expressions ...]]

positional arguments:
  file              The pcap file
  expressions       DBus message match expressions

optional arguments:
  -h, --help        show this help message and exit
  --json            Emit a JSON representation of the messages
  --no-track-calls  Make a call response pass filters
```

Basic output, which tries to give friendly names to fields:

```sh
$ dbus-pcap dbus_capture.pcap | head -n 3
1579115731.066975: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=1, flags=0, version=1, length=0, cookie=1), fields=[Field(type=<MessageFieldType.PATH: 1>, data='/org/freedesktop/DBus'), Field(type=<MessageFieldType.MEMBER: 3>, data='Hello'), Field(type=<MessageFieldType.INTERFACE: 2>, data='org.freedesktop.DBus'), Field(type=<MessageFieldType.DESTINATION: 6>, data='org.freedesktop.DBus'), Field(type=<MessageFieldType.SENDER: 7>, data=':1.113')]), body=[])
1579115731.066986: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=2, flags=1, version=1, length=11, cookie=4294967295), fields=[Field(type=<MessageFieldType.REPLY_SERIAL: 5>, data=1), Field(type=<MessageFieldType.SENDER: 7>, data='org.freedesktop.DBus'), Field(type=<MessageFieldType.DESTINATION: 6>, data=':1.113'), Field(type=<MessageFieldType.SIGNATURE: 8>, data='s')]), body=[':1.113'])
1579115731.066999: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=4, flags=1, version=1, length=31, cookie=4294967295), fields=[Field(type=<MessageFieldType.SENDER: 7>, data='org.freedesktop.DBus'), Field(type=<MessageFieldType.PATH: 1>, data='/org/freedesktop/DBus'), Field(type=<MessageFieldType.INTERFACE: 2>, data='org.freedesktop.DBus'), Field(type=<MessageFieldType.MEMBER: 3>, data='NameOwnerChanged'), Field(type=<MessageFieldType.SIGNATURE: 8>, data='sss')]), body=[':1.113', '', ''])
```

JSON output, which just gives [raw field numbers that need to be
interpreted](https://dbus.freedesktop.org/doc/dbus-specification.html#message-protocol):

```sh
2 11:07:14 andrew@mistburn:~/env/openbmc/openbmc/issues/SW482644$ ~/src/openbmc/openbmc-tools/amboar/obmc-scripts/dbus-pcap/dbus-pcap --json dbus_capture.pcap | head -n 3
[[[108, 1, 0, 1, 0, 1], [[1, "/org/freedesktop/DBus"], [3, "Hello"], [2, "org.freedesktop.DBus"], [6, "org.freedesktop.DBus"], [7, ":1.113"]]], []]
[[[108, 2, 1, 1, 11, 4294967295], [[5, 1], [7, "org.freedesktop.DBus"], [6, ":1.113"], [8, "s"]]], [":1.113"]]
[[[108, 4, 1, 1, 31, 4294967295], [[7, "org.freedesktop.DBus"], [1, "/org/freedesktop/DBus"], [2, "org.freedesktop.DBus"], [3, "NameOwnerChanged"], [8, "sss"]]], [":1.113", "", ""]]
```

Basic output with a simple match expression (`interface=org.openbmc.HostIpmi`)
applied:

```sh
$ dbus-pcap dbus_capture.pcap interface=org.openbmc.HostIpmi | head -n 3
1579115731.237553: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=1, flags=0, version=1, length=0, cookie=30), fields=[Field(type=<MessageFieldType.PATH: 1>, data='/org/openbmc/HostIpmi/1'), Field(type=<MessageFieldType.MEMBER: 3>, data='setAttention'), Field(type=<MessageFieldType.INTERFACE: 2>, data='org.openbmc.HostIpmi'), Field(type=<MessageFieldType.DESTINATION: 6>, data='org.openbmc.HostIpmi'), Field(type=<MessageFieldType.SENDER: 7>, data=':1.93')]), body=[])
1579115731.237623: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=2, flags=1, version=1, length=8, cookie=9), fields=[Field(type=<MessageFieldType.REPLY_SERIAL: 5>, data=30), Field(type=<MessageFieldType.DESTINATION: 6>, data=':1.93'), Field(type=<MessageFieldType.SIGNATURE: 8>, data='x'), Field(type=<MessageFieldType.SENDER: 7>, data=':1.4')]), body=[0])
1579115814.248163: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=4, flags=1, version=1, length=14, cookie=10), fields=[Field(type=<MessageFieldType.PATH: 1>, data='/org/openbmc/HostIpmi/1'), Field(type=<MessageFieldType.INTERFACE: 2>, data='org.openbmc.HostIpmi'), Field(type=<MessageFieldType.MEMBER: 3>, data='ReceivedMessage'), Field(type=<MessageFieldType.SIGNATURE: 8>, data='yyyyay'), Field(type=<MessageFieldType.SENDER: 7>, data=':1.4')]), body=[1, 6, 0, 36, [65, 1, 0, 2, 112, 23]])
```

Basic output with a compound match expression
(`interface='org.freedesktop.DBus.Properties',path='/xyz/openbmc_project/Hiomapd'`)

```sh
$ dbus-pcap dbus_capture.pcap "interface='org.freedesktop.DBus.Properties',path='/xyz/openbmc_project/Hiomapd'" | head -n 3
1579115731.237206: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=4, flags=1, version=1, length=76, cookie=19), fields=[Field(type=<MessageFieldType.PATH: 1>, data='/xyz/openbmc_project/Hiomapd'), Field(type=<MessageFieldType.INTERFACE: 2>, data='org.freedesktop.DBus.Properties'), Field(type=<MessageFieldType.MEMBER: 3>, data='PropertiesChanged'), Field(type=<MessageFieldType.SIGNATURE: 8>, data='sa{sv}as'), Field(type=<MessageFieldType.SENDER: 7>, data=':1.103')]), body=['xyz.openbmc_project.Hiomapd.Protocol.V2', [['DaemonReady', 0]], []])
1579115766.484496: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=4, flags=1, version=1, length=104, cookie=20), fields=[Field(type=<MessageFieldType.PATH: 1>, data='/xyz/openbmc_project/Hiomapd'), Field(type=<MessageFieldType.INTERFACE: 2>, data='org.freedesktop.DBus.Properties'), Field(type=<MessageFieldType.MEMBER: 3>, data='PropertiesChanged'), Field(type=<MessageFieldType.SIGNATURE: 8>, data='sa{sv}as'), Field(type=<MessageFieldType.SENDER: 7>, data=':1.103')]), body=['xyz.openbmc_project.Hiomapd.Protocol.V2', [['DaemonReady', 1], ['ProtocolReset', 1]], []])
1579115814.263394: CookedMessage(header=CookedHeader(fixed=FixedHeader(endian=108, type=4, flags=1, version=1, length=104, cookie=23), fields=[Field(type=<MessageFieldType.PATH: 1>, data='/xyz/openbmc_project/Hiomapd'), Field(type=<MessageFieldType.INTERFACE: 2>, data='org.freedesktop.DBus.Properties'), Field(type=<MessageFieldType.MEMBER: 3>, data='PropertiesChanged'), Field(type=<MessageFieldType.SIGNATURE: 8>, data='sa{sv}as'), Field(type=<MessageFieldType.SENDER: 7>, data=':1.103')]), body=['xyz.openbmc_project.Hiomapd.Protocol.V2', [['DaemonReady', 1], ['ProtocolReset', 1]], []])
```

Messages are printed only if they meet both properties of a compound match
expression.

Multiple match expressions can be provided; messages are printed if they match
any of the supplied expressions. This allows tracking messages between multiple
daemons over time, filtering the chaff from complex method and signal
interactions.

The script has helped me catch subtle timing issues and generally help me
visualise system state when debugging OpenBMC. As implied by Jessie, maybe
life could be better if this complexity didn't exist at all, but given that it
does we at least now have a useful tool to deal with it.
