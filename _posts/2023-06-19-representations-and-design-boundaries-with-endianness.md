---
title: Representations and Design Boundaries with Endianness
author: Andrew
---

The nuts and bolts of [endianness][wikipedia-endianness] are a bit fiddly.
Keeping value endianness in mind when reading through memory dumps is annoying
but not intractable. In my mind, a more important concern is deciding where to
address endianness in a system design. The answer to that is very likely "at the
boundaries", but this also requires knowing where the boundaries are in a system
design.

[wikipedia-endianness]: https://en.wikipedia.org/wiki/Endianness

```
         Big Endian   MS  Little Endian
      +-------------- 12 --------------+
      |  +----------- 34 -----------+  |
      |  |  +-------- AB --------+  |  |
      |  |  |  +----- CD -----+  |  |  |
      |  |  |  |      LS      |  |  |  |
      v  v  v  v      ||      v  v  v  v
MS | 12 34 AB CD | LS || LS | CD AB 34 12 | MS | Value
   | 0  1  2  3  |    ||    | 0  1  2  3  |    | Address
```

Endianness is at its most important when data is being exchanged. An exchange
can take place in space, e.g. between two systems over a network, or in time,
e.g. by being written to and read from persistent storage by distinct releases
of an application. However, equally important is when it's unimportant: When
data is ephemeral, contained entirely within a process, we can pretend the
problem of endianness doesn't exist. It hasn't gone away, as the CPU will
operate in a specified endianness, but it can feel invisible.

Unpicking this latter point further, we call this representation "host"
endianness. It doesn't matter whether the CPU is running as big-endian or
little-endian, what matters is that the representation is unchanging over time.

Programming in host endianness is the most intuitive and frictionless way to
work with values, precisely because it feels invisible. Separately, when
designing a software system, we want the design to feel intuitive and
frictionless because to do otherwise would make working with the system tedious
and error-prone. It's already hard enough to write software that's error-free,
so we should work to avoid tedious and error-prone designs.

This strongly suggests that as much as possible, we should be working with
values in host endianness.

But what if we are working with data that's being exchanged in space or time?
Doesn't that put a stop to our "make the endianness invisible" design principle?
Not if we consistently apply the approach of handling
[reified][merriam-webster-reify] endianness at the system's boundaries. But
where are a system's boundaries?

[merriam-webster-reify]: https://www.merriam-webster.com/dictionary/reify

A system's boundaries are where any effect first becomes our system's problem,
or first becomes another system's problem.

[Consider a library that handles encoding and decoding messages exchanged
between two systems][openbmc-libpldm]. A "system" in the context of the library
includes the library and its calling application. For the library's API design
to be intuitive and frictionless, it is required that values passed from or to
the caller must be in host endianness. To do otherwise would be tedious -
forcing endian conversion at the all call sites, and error-prone - performing
endianness conversion at the call sites is easily forgotten. However, a
requirement on the library's implementation is that it correctly encode values
into the wire format for the message in order to [conform with the
specification][dmtf-pmci-pldm]. The same applies in the obverse case: As above
it's a requirement that values passed from a received message to the caller of
the library's APIs be in host endianness. However, it's also a requirement that
the library correctly decode values from the wire format for the message in
order to conform to the specification.

Together these requirements "squeeze" the design space in which the endian
conversion must take place: It must occur at the system boundary, in the
library's implementation, where the encoding and decoding of messages takes
place. These concerns must not be forced on callers of the library's API[^1].

[openbmc-libpldm]: https://github.com/openbmc/libpldm
[dmtf-pmci-pldm]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0240_1.1.0.pdf

[^1]: If the library fails to adhere to the principle of handling endianness at
    the boundaries it is also outsourcing a core promise of protocol correctness
    to the caller. In effect, under such a design the library can never claim to
    be a correct implementation, as the statement must be extended to cover the
    context of all call sites for the library's APIs, or assume caller
    correctness.
