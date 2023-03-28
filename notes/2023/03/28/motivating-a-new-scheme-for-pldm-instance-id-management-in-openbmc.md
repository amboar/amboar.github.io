# Motivating a New Scheme for PLDM Instance ID Management in OpenBMC

Recently [Rashmica][sthbrx-rashmica] has been doing some work to enable use of
[Linux's `AF_MCTP` sockets][linux-doc-af-mctp] in OpenBMC. Until now we've
relied on a userspace implementation of [MCTP][dmtf-dsp0236-mctp] through
[libmctp][libmctp], but this rapidly hit limitations at the kernel/userspace
interface boundary. To fix that, [Code Construct did the work to move MCTP into
the kernel][code-construct-mctp].

[sthbrx-rashmica]: https://sthbrx.github.io/author/rashmica-gupta.html
[linux-doc-af-mctp]: https://www.kernel.org/doc/html/latest/networking/mctp.html
[dmtf-dsp0236-mctp]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0236_1.3.1.pdf
[libmctp]: https://github.com/openbmc/libmctp
[code-construct-mctp]: https://codeconstruct.com.au/docs/mctp-on-linux-introduction/

A consequence of using `libmctp` as the MCTP implementation in OpenBMC is that
other components in the distro had to make use of the `AF_UNIX` socket provided
by the `mctp-demux-daemon`. These components include the requester API of
`libpldm`. That isn't so much of a concern in itself as the `mctp-demux-daemon`
design was (intentionally) socket-based, but a further problem was that the
design of the requester API baked in some assumptions that the underlying MCTP
transport implementation was the `AF_UNIX` socket provided by
`mctp-demux-daemon`.

So, a new requester-related API is needed to transition `libpldm` from using the
`AF_UNIX` socket from `mctp-demux-daemon` to `AF_MCTP` sockets provided by the
kernel. A side-goal of this new interface is to also allow the use of [The Other
PLDM Transport Binding, PLDM over RBT][dmtf-dsp0222-ncsi].

[dmtf-dsp0222-ncsi]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0222_1.1.1.pdf

At this point a tangle of problems emerged:

1. The current requester API isn't really a requester API by the definition of
   the [PLDM specification][dmtf-dsp0240-pldm], rather is a transport API. It's
   a transport API as it doesn't implement any of the timeout and retry
   semantics layed down in the base specification
2. The `libpldm` APIs to encode a PLDM message require that an instance ID is
   provided, conflating serialisation of the message structure with framing the
   serialised message for transport
3. Instance IDs are used by the PLDM protocol for message correlation across
   delays and retries
4. If there were to be a requester API that abstracted over the required timeout
   and retry semantics, such an API would encapsulate any instance ID detail

[dmtf-dsp0240-pldm]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0240_1.1.0.pdf

So instead of redesigning a requester API for `libpldm`, the job became an
effort to design two new lower-level APIs. The two live side-by-side and
together can be used to compose an eventual requester API implementation:

1. A PLDM transport API to abstract over MCTP vs RBT details
2. An API to manage the instance ID resource pool for a given terminus

## Managing the pools of Instance IDs

As mentioned above, the instance IDs are used by the protocol for message
correlation and to drive the timeout/retry state machine. That's
straight-forward enough, until we consider that:

1. In OpenBMC, multiple applications link against `libpldm` for a variety of
   purposes
2. `libpldm` doesn't provide a true requester API, forcing its users to
   implement the behaviour themselves
3. Those users need access to a global instance ID allocator to implement the
   timeout/retry behaviour

A subtle point here is that as a result of its current API design `libpldm`
embeds the instance ID in the PLDM message at the point of serialisation, which
is a separate concern from exchanging serialised messages. As such, allocation
and deallocation of instance IDs is decoupled from any timeout/retry semantics.
Technically we cannot assume that the expiry of an instance ID can be measured
from the point at which it was allocated.

We also need to consider that applications can crash and leak any instance ID
resources they had allocated but were yet to release. That leads us to the
current architecture for instance ID management:

1. `pldmd` exposes a DBus object implementing
   `xyz.openbmc_project.PLDM.Requester` to satisfy the requirement for a global
   instance ID allocator
2. Any applications that are not `pldmd` request an instance ID through the DBus
   interface provided by `pldmd`

Like the existing requester API in `libpldm`, the implementation of the
`xyz.openbmc_project.PLDM.Requester` interface in `pldmd` is unfortunately tied
to implementation details of `mctp-demux-daemon`. We can observe by the
[interface definition][openbmc-pdi-pldm-requester] that no method to return an
instance ID to TID's pool is provided[^1]. It turns out `mctp-demux-daemon` has
the (unfortunate?) behaviour of sending all messages to all connections on its
`AF_UNIX` socket for the given message type *if the traffic is destined for the
local EID*. This *includes responses from remote endpoints*.

[openbmc-pdi-pldm-requester]: https://github.com/openbmc/phosphor-dbus-interfaces/blob/a1b26a4bdf606df141695561a92a4f060cdd0156/yaml/xyz/openbmc_project/PLDM/Requester.interface.yaml

In this manner, `pldmd` snoops on the response traffic intended for another
application to reclaim the instance ID it had handed out. This snooping
behaviour also allows it to infer when a response hasn't been received in a
timely fashion, and to expire the allocation accordingly.

Thus we have the properties that:

1. Instance ID allocations are expired and returned to the TID's pool as soon as
   a response is received, minimising the chance of exhaustion
2. Instance ID allocation lifetime is bounded by the retry/timeout
   behaviour specified in [DSP0240][dmtf-dsp0240-pldm]

A problem we have is that [`AF_MCTP` sockets do not allow `pldmd` to snoop the
traffic in this fashion][linux-doc-af-mctp-sendto], and so we cannot uphold
property 1:

[linux-doc-af-mctp-sendto]: https://www.kernel.org/doc/html/latest/networking/mctp.html#sendto-sendmsg-send-transmit-an-mctp-message

> Sockets will only receive responses to requests they have sent (with TO=1) and
> may only respond (with TO=0) to requests they have received.

As a result, it's not enough to provide an instance ID allocation API in
`libpldm` that abstracts over DBus calls to `GetInstanceId` on `pldmd`'s
`xyz.openbmc_project.PLDM.Requester` interface. Instead, to migrate to `AF_MCTP`
as the PLDM transport implementation we have to either:

1. Add the ability to explicitly release an instance ID back to the pool to the
   DBus API, or
2. Develop a new scheme for an instance ID database that doesn't rely on
   implementation details of `mctp-demux-daemon` and is robust against
   application crashes

Option 1 requires that `libpldm` take a dependency on e.g. `libsystemd` for it's
`sd_bus_*` APIs to handle the DBus traffic for the instance ID lifecycle.
Further, the need for IO to acquire the instance ID means we need to design the
API such that it can be used asynchronously. Finally, implementing the API in
terms of DBus also prevents `pldmd` from exploiting the API to implement the
DBus interface (`pldmd` would call into itself via the DBus interface, ending
either in recursion or deadlock).

None of these are particularly appealing.

The question is then whether option 2 is feasible. Going that path, the work for
the instance ID allocation API becomes:

1. Find an IO-free scheme for managing instance IDs that is robust against
   application crashes
2. Define an instance ID allocator API for `libpldm`
3. Implement the `libpldm` instance ID allocation API in terms of the new
   scheme from 1
4. Rework `pldmd`'s implementation of the `xyz.openbmc_project.PLDM.Requester`
   DBus interface to be in terms of the `libpldm` API
5. Refactor the users of `libpldm` to call the new instance ID API directly
   instead of all independently implementing the calls to the current DBus
   interface

The task order is important. Implementing the instance ID allocation API first
in terms of the DBus interface is invalid: The conversion of `pldmd` must be not
before the API implementation has switched to the new scheme. By contrast, if an
application is reworked to use the API before the switch to the new scheme, the
act of switching the implementation to the new scheme to enable the conversion
of `pldmd` causes the application and `pldmd` to lose coherency. Thus, the
conversion of `pldmd` must be first, and the API need never be implemented in
terms of the DBus interface.

With that done, we can then progress the effort to switch over to using
`AF_MCTP` as the MCTP transport implementation.

[^1]: Eagle-eyed readers would also note that `GetInstanceId` is defined in
    terms of the destination MCTP EID and not the destination TID as specified,
    putting an architectural ding in the desire to support RBT as a PLDM
    transport
