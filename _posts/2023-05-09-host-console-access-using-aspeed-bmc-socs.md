---
title: Host Console Access using Aspeed BMC SoCs
author: Andrew
---

(Updated 2023-06-15: Added the ASCII art diagram)

Experience suggests that configuring an Aspeed BMC for host console access can
be a confusing task. The UART capabilities provided by the SoCs allow access to
the host console both via physical connectors on the rear of the chassis and
also via Serial-over-LAN (SOL).

To understand what needs to be done, before we even discuss UARTs, we need to
discuss the bus topolgy of a platform that includes an Aspeed BMC SoC.

## External and Internal Bus Topology

Connecting the host console to the BMC usually involves the use of one of two
peripheral busses on the host's part:

1. LPC - Low Pin Count, a cut-down ISA bus
2. PCIe

To align with my experience we'll consider the BMC as an LPC peripheral of the
host. This also allows some discussion of other finer points of the SoC
configuration.

Separate to the external busses, we also have the internal busses:

1. AHB - Advanced High-performance Bus
2. APB - Advanced Peripheral Bus

High-speed peripherals such as ethernet and USB tend to be attached to the AHB,
while the APB is transparently bridged onto the AHB and caters to peripherals
with lower throughput requirements. The LPC peripheral is hooked up to the AHB
in the BMC SoCs. Relevant logic is also exposed via the APB in some cases.

## Classes of UART

The Aspeed SoCs provide three separate classes of UARTs[^1]:

[^1]: Actually more like 5, after we account for the PCIe XVUARTs, and the
    Pass-through UART (PUART) available on the AST2400 and AST2500.  The PUART
    allows the BMC to configure and enable UARTs 1 and 2 on the host side of the
    LPC interface without host firmware support.

1. System UARTs (SUARTs)
2. Virtual UARTs (VUARTs)
3. Private UARTs

It's important to keep these classes in mind as we dig through the rest of the
properties involved in configuring SOL.

## UART Instances and Affinities

The AST2400 and AST2500 SoCs both come with four SUARTs and a single VUART. The
AST2600 also sports four SUARTs but adds a second VUART. In addition to the
SUARTs and VUARTs, all of the AST2400, AST2500 and AST2600 provide at least one
extra private UART whose register interface is not exposed on the LPC bus. In
the case of the AST2600, it has not just one, but an additional nine UARTs. One
of these private UARTs is generally used for the BMC's own console, typically
UART5.

| Aspeed SoC | SUARTs | Private UARTs | VUARTs |
|------------|--------|---------------|--------|
| AST2400    | 4      | 1             | 1      |
| AST2500    | 4      | 1             | 1      |
| AST2600    | 4      | 9             | 2      |

A complication is that the datasheet of each SoC often doesn't distinguish
between these classes of UARTs (except the VUART(s), which are always labelled
VUART). Instead it refers to labels such as UART1 or UART5. For all SoCs, UARTs
1, 2, 3, and 4 are all exposed on the LPC bus and are capable of being driven by
the host. Again in all cases, the reset sources for UARTs 1 and 2 default to the
LPC bus' `LPCRST#` signal, effectively mapping them to the host out of BMC
reset[^2].

[^2]: See `HICR9[4:7]` on all SoCs, [though there is a complication with the
    clocks][discord-openbmc-jdoman-uart-clocks]

[discord-openbmc-jdoman-uart-clocks]: https://discord.com/channels/775381525260664832/775381525260664836/1102996061238349824

| UART ID | AST2400 | AST2500 | AST2600 | Reset Affinity   |
|---------|---------|---------|---------|------------------|
| UART1   | SUART   | SUART   | SUART   | Host (`LPCRST#`) |
| UART2   | SUART   | SUART   | SUART   | Host (`LPCRST#`) |
| UART3   | SUART   | SUART   | SUART   | BMC              |
| UART4   | SUART   | SUART   | SUART   | BMC              |
| UART5   | Private | Private | Private | BMC              |
| UART6   | -       | -       | Private | BMC              |
| ...     | -       | -       | ...     | ...              |
| UART13  | -       | -       | Private | BMC              |

## SUART Behaviours

A crucial point regarding the SUARTs, alluded to by the reset affinity, is that
whether an SUART is used by the host is a matter of agreement between the host
and BMC firmware implementations[^3]. Put another way, for UARTs 1 through 4,
the host and the BMC both drive the same instance of UART logic, but via
different busses. To implement SOL using the SUARTs they must be considered in
pairs in order to maintain the contract of standard UART behaviour.

[^3]: Effectively forming a "platform ABI" for the firmware by hardware
    assignment

This leads to the question of how to pair them together, and the answer to that
is the UART mux. But before we consider the UART mux, we need to tease appart
the concept of a UART and its associated physical IO connector.

## The SUART Perspective

```
            AHB                           LPC

             │                             │
             ├─────────────────┐   ┌───────┤
             │                 │   │       │
             ├───────┐   ┌─────┼───┼───────┤
             │       │   │     │   │       │
             │     ┌─▼───▼─┐ ┌─▼───▼─┐     │
             │     │       │ │       │     │
             │     │ SUART │ │ SUART │     │
             │     │       │ │       │     │
             │     └───▲───┘ └───▲───┘     │
             │         │         │         │
             │         │         │         │
             │      ┌──▼─────────▼───┐     │
             │      │                │     │
             ├──────►      MUX       │     │
             │      │                │     │
             │      └──▲─────────▲───┘     │
             │         │         │         │
                       │         │
                    ┌──▼───┐ ┌───▼──┐
                    │      │ │      │
                    │  IO  │ │  IO  │
                    │      │ │      │
                    └──▲───┘ └───▲──┘
                       │         │
                       │         │
                 ──────▼─────────▼─────
                         Chassis
                        Backplate
```

## SUARTs and Physical IOs

Typically a UART is paired directly with a physical connector (commonly DB-9,
[or DE-9, bowing to wikipedia pedantry][wikipedia-d-subminiature]) to enable the
RS232 transmission. The Aspeed BMC SoCs transcend this common relationship, and
for the AST2400 and AST2500 ship six UART RS232 IO pin sets for the five
available UARTs that internally implement RS232 signalling (i.e. all but the
VUARTs). By default the association between the UARTs and the IOs is a linear
mapping (i.e. UART1 is associated with IO1, etc). However, the relationships are
configurable, and this again leads us to the UART mux.

[wikipedia-d-subminiature]: https://en.wikipedia.org/w/index.php?title=D-subminiature&oldid=1150195395

## The UART Mux

(We will consider the UART mux in the AST2400 and AST2500 to nail down the
concepts without invoking some of the complexity of the AST2600.)

The UART mux allows us to create complex associations between the two classes of
serial entities in the SoC: The physical RS232 IO pin sets and the SUARTs.
There is also the restricted case of the private UART (UART5) but we will
disregard that for the moment.

Conceptually, the mux operates by considering each instance of two entities as a
pair: The local entity and the remote entity. The mux control fields in LPC
registers `HICR9` and `HICRA` allow us to configure the remote entity as the
input to the local entity. Again, we're using the abstract language "entity"
here because the pair could be any of:

| Local Entity Class | Remote Entity Class |
|--------------------|---------------------|
| UART               | UART                |
| UART               | IO                  |
| IO                 | UART                |
| IO                 | IO                  |

Concretely, at SoC reset, the mux configuration defaults to the linear mapping
between UARTs and IOs for the first five UARTs and IOs:

| Local Entity Instance | Remote Entity Instance | Comment                     |
|-----------------------|------------------------|-----------------------------|
| UART1                 | IO1                    | UART1's input source is IO1 |
| IO1                   | UART1                  | IO1's input source is UART1 |
| ...                   | ...                    | ...                         |
| UART5                 | IO5                    | UART5's input source is IO5 |
| IO5                   | UART5                  | IO5's input source is UART5 |

The power of the mux start to become visible with the extra IO6. The reset
configuration for the SoC is that UART1's output is duplicated to both IO1 and
IO6, allowing anything plugging into a connector wired to IO6 to snoop the
output from UART1 without knowledge of the entity connected on IO1.

| Local Entity Instance | Remote Entity Instance | Comment                      |
|-----------------------|------------------------|------------------------------|
| IO6                   | UART1                  | IO6's input source is UART1  |
| -                     | IO6                    | IO6 can't be an input source |

## Implementing SOL with the SUARTs and the UART Mux

Using the information above, implementing SOL using the UART mux simply exploits
the arbitrary association of UARTs via the mux. Let's develop a configuration as
a concrete example.

Given that UARTs 1 and 2 are mapped to the host by default while UARTs 3 and 4
are mapped to the BMC we'll pick UART1 and UART3 to use as a pair. From there we
configure the mux such that we have:

| Local Entity Instance | Remote Entity Instance | Comment                       |
|-----------------------|------------------------|-------------------------------|
| UART1                 | UART3                  | UART1's input source is UART3 |
| UART3                 | UART1                  | UART3's input source is UART1 |

With that in place we leave the host firmware to drive UART1 and open UART3 on
the BMC. If we deploy [obmc-console][] to do the job of opening UART3 then we
immediately have a fully functional SOL stack.

[obmc-console]: https://github.com/openbmc/obmc-console/

As a final point it's worth noting that [Linux has a driver for the UART
mux][linux-driver-aspeed-uart-mux] that is [configured through
sysfs][linux-abi-aspeed-uart-mux]. This might save some wading through the
datasheet and interpretation of the various details outlined here.

[linux-driver-aspeed-uart-mux]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/soc/aspeed/aspeed-uart-routing.c?h=v6.3
[linux-abi-aspeed-uart-mux]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/Documentation/ABI/testing/sysfs-driver-aspeed-uart-routing?h=v6.3

## Limitations of the SUARTs for simple SOL requirements

With SUART SOL covered, it's worth mentioning that the LPC base address, SerIRQ
ID, and enabled state for a given SUART are configured through the LPC-based
SuperIO controller. The configuration is applied using logical devices `0x2`,
`0x3`, `0xB` and `0xC` for UARTs 1, 2, 3 and 4 respectively. However, a
complicating factor for the AST2400 and AST2500 is that leaving SuperIO enabled
to allow configuration of the SUARTs gives some exposure to
[CVE-2019-6260][][^4]

[CVE-2019-6260]: https://nvd.nist.gov/vuln/detail/CVE-2019-6260
[^4]: The confidentiality issue is mitigated in the AST2600

Going forward, the recommended approach for implementing SOL where there are
only simple requirements is to use the VUARTs instead.

## VUARTs: Avoiding the UART mux for simple SOL requirements

To start with, the VUARTs require none of the the SuperIO malarky, but at the
cost of leaving the host-side configuration entirely up to the BMC. So to handle
that we've traded off one set of platform ABI definitions for another: The BMC
must now set up the VUART to meet the host firmware's assumptions. The only way
around this is [for the host to exploit the iLPC2AHB bridge][CVE-2019-6260] to
drive transactions onto the BMC's AHB to configure the VUARTs itself, which
isn't particularly tasteful.

Setting aside that wart, the implementation of the VUART is a pair of UART
logics, one for each of the APB and LPC interfaces with [the UART FIFOs crossing
the clock domains of the two busses][vuart-logic-is-broken]. With this design
there's no need to pair SUARTs via the UART mux, or accidentally stomp on each
other's use of one of the SUARTs. The host accesses the LPC-exposed UART logic
instance, while the BMC accesses the APB-exposed UART logic instance. If the
host firmware allows it, use of the VUART(s) is simply a matter of enabling them
on the BMC side and then running `obmc-console-server` against the resulting TTY
device.

[vuart-logic-is-broken]: https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?h=v6.3&id=df8f2be2fd0b
