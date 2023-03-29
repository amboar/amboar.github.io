# A Global PLDM Instance ID Allocator in Userspace for `libpldm`

In [Motivating a New Scheme for PLDM Instance ID Management in
OpenBMC][amboar-notes-motivating-a-new-scheme] I talked about why we need to
change how instance IDs are managed in OpenBMC. Underpinning it is the shift to
using `AF_MCTP` sockets provided by Linux.

[amboar-notes-motivating-a-new-scheme]: https://amboar.github.io/notes/2023/03/28/motivating-a-new-scheme-for-pldm-instance-id-management-in-openbmc.html

Conceptually, instance IDs should be allocated right before a serialised PLDM
message first enters the transmit pipeline and deallocated once the response
deadline state machine has reached a final state, either with the response
message received or exhaustion of timeouts and retries. However, as discussed in
the previous post, this is not how the APIs of `libpldm` are currently designed:
Message serialisation and framing are conflated into one operation, infecting
all of the encoding APIs with the transmission detail of instance IDs.

Given this, lifecycle management of instance IDs is an important concern for all
users of `libpldm`. However, `libpldm` currently punts on that problem too, and
users are required to each implement a call to a DBus API hosted by `pldmd` to
allocate an instance ID for each message they wish to send.

We aim to fix that by giving `libpldm` an instance ID API with the following
requirements and considerations:

1. The instance ID management should be global - all applications linking
   against `libpldm` in a machine session should have a coherent view of the
   pool state
2. Instance ID management should be robust to resource leaks, e.g. through
   application crashes
3. As a library component, the API needs to cater to applications running with
   arbitrary EUID/EGIDs, root cannot be assumed
4. As a library component, the API should not force constraints on the IO
   strategy (synchronous vs asynchronous) of the application

Regarding 4, the winning strategy would be to avoid any IO at all, eliminating
the concern. Certainly a DBus-based implementation fails that desire, along with
most other forms of IPC. If not IPC, our choices are mostly limited to the
kernel or the filesystem, and [pushing the problem into the kernel was already
ruled out][jk-roasts-arj]. So we're left with trying to exploit the filesystem,
while somehow also avoiding performing IO if possible.

[jk-roasts-arj]: https://discord.com/channels/775381525260664832/778790638563885086/1078558597744709693

With the filesystem conclusion in mind, point 3 puts constraints on how we
access the database. Allowing access from arbitrary EUIDs/EGIDs means we have to
somehow constrain what might be written, which is difficult given applications
might not even use `libpldm` for access. Without care it's feasible for a process to simply
truncate the database or delete it entirely. Further, writing anything at all
fails our no-IO desire from 4, so it's ideal if the whole thing was a
pre-determined size and either marked read-only via permissions, or exposed
through a read-only mount.

The question then becomes how we can maintain state with a read-only database,
and to that end [Matt Johnston swung in with the suggestion to exploit
range-locking on the database file][mkj-saves-the-day]. With some
threading-the-eye-of-the-needle in the implementation, this approach has the
glorious property of solving for all the requirements:

1. Requirement 2, as the kernel tracks the process or file descriptor state and
   releases any locks associated with the resource when it is terminated
2. Requirement 1, by the nature of using a regular file on the filesystem,
   visible to all processes
3. Requirement 4, as range-locking is implemented through
   [fcntl(2)][man-2-fcntl] and isn't an IO operation
4. Requirement 3, as no write operations are required, though there are
   constraints on what type of lock can be acquired based on the access mode.

[mkj-saves-the-day]: https://discord.com/channels/775381525260664832/778790638563885086/1078561344548261898
[man-2-fcntl]: https://man7.org/linux/man-pages/man2/fcntl.2.html

## Threading the Eye of the Needle

An immediate choice is required: POSIX vs Open File Description (OFD) advisory
locking. While OFD is not standardised, `libpldm` is very likely either
running under Linux hosted environment, or in a freestanding environment. For
the latter nothing POSIX matters, and the former supports OFD locking. We will
concentrate on the behaviour of OFD locking under Linux.

### Database Format and Allocation Semantics

While many schemes are possible, such as one-file-per-TID, probably the most
pragmatic scheme is to define a single database file sized on the order of the
Cartesian product of the Terminus ID (TID) and Instance ID (IID) spaces. By
[DSP0240][dmtf-dsp0240-pldm] the Terminus ID space is 8 bits while the Instance
ID space is 5 bits. OFD advisory locking operates at the resolution of bytes,
yielding a `2^8 * 2^5 = 256 * 32 = 8192` byte file arranged as a two-dimensional
array. As a back-of-the-envelope sketch, applying an OFD range-lock on byte
`TID * 32 + IID` allocates the `(TID, IID)` tuple to the file descriptor
associated with the lock.

[dmtf-dsp0240-pldm]: https://www.dmtf.org/sites/default/files/standards/documents/DSP0240_1.1.0.pdf

Again, this approach gives us a single 8192 byte file of zeros that we can
install as read-only at e.g. `/usr/share/libpldm/instance-db/default`. Briefly,
by contrast, the one-database-file-per-TID approach quickly becomes unweildy.
We would need to either pre-emptively generate and install 256 32-byte files, or
try to develop some scheme to safely create them at runtime in a lazy, as-needed
fashion. The latter gets hard on an integrity front when we take into account
requirement 3.

### The Ordeal

OFD advisory locking is a typical `RWLock` scheme. Read (`F_RDLCK`) and write
(`F_WRLCK`) lock types are provided, where read-locked resources can be shared
across multiple read locks while write-locked resources are exclusive. A write
lock cannot be allocated on a resource if at least one read lock is held. Locks
are requested through the (non-blocking) `F_OFD_SETLK` command to `fcntl(2)`.
Further, the lock state of a resource can be queried with `F_OFD_GETLK`.

Allocating an instance ID for a destination terminus naturally fits with the
semantics of `F_WRLCK`: The application wants exclusive access to that instance
ID until the exchange has terminated. However, [the man-page for `fcntl(2)`'s
advisory locking comes with this morsel of a constraint][man-2-fcntl]:

> In order to place a read lock, `fd` must be open for reading. In order to
> place a write lock, `fd` must be open for writing. To place both types of
> lock, open a file read-write.

Installing the database as a read-only stymies our ability to acquire a write
lock as it prevents us from opening the database in the `O_WRONLY` access mode.

### A Hypothetical Scheme

However, [diving back into the `fcntl(2)` man-page][man-2-fcntl] there are more
tidbits:

```
F_OFD_GETLK (struct flock *)
    On input to this call, lock describes an open file
    description lock we would like to place on the file.  If
    the lock could be placed, fcntl() does not actually place
    it, but returns F_UNLCK in the l_type field of lock and
    leaves the other fields of the structure unchanged.  If
    one or more incompatible locks would prevent this lock
    being placed, then details about one of these locks are
    returned via lock, as described above for F_GETLK.
```

Where `F_GETLK` is described as:

```
F_GETLK (struct flock *)
    On input to this call, lock describes a lock we would like
    to place on the file.  If the lock could be placed,
    fcntl() does not actually place it, but returns F_UNLCK in
    the l_type field of lock and leaves the other fields of
    the structure unchanged.

    If one or more incompatible locks would prevent this lock
    being placed, then fcntl() returns details about one of
    those locks in the l_type, l_whence, l_start, and l_len
    fields of lock.  If the conflicting lock is a traditional
    (process-associated) record lock, then the l_pid field is
    set to the PID of the process holding that lock.  If the
    conflicting lock is an open file description lock, then
    l_pid is set to -1.  Note that the returned information
    may already be out of date by the time the caller inspects
    it.
```

While the last sentence of the descripton of `F_GETLK` asserts it's incorrect to
issue the non-atomic command sequence (`F_OFD_GETLK`, `F_OFD_SETLK`) to acquire
an `F_RDLCK`[^1] if no lock exists at the time of `F_OFD_GETLK`, it *may* be
correct to issue the non-atomic sequence (`F_OFD_SETLK`, `F_OFD_GETLK`):

```
    Open file description locks placed via the same open file
    description (i.e., via the same file descriptor, or via a
    duplicate of the file descriptor created by fork(2), dup(2),
    fcntl() F_DUPFD, and so on) are always compatible: if a new lock
    is placed on an already locked region, then the existing lock is
    converted to the new lock type.  (Such conversions may result in
    splitting, shrinking, or coalescing with an existing lock as
    discussed above.)
```

While we cannot ever acquire a `F_WRLCK`, it still might be possible to ask if
such a request would succeed due to the lack of any conflicting locks
regardless of the file descriptor's access mode. If the kernel indicates in its
response to `F_OFD_GETLK` that the lock could be promoted to an `F_WRLCK` by
populating `struct flock`'s `l_type` member with `F_UNLCK` then we know that at
the time of the `F_OFD_GETLK` command that the file descriptor was the only one
holding an `F_RDLCK` at the offset.

### The Reward

[The experiment][the-experiment] showed that the desired behaviour is in-fact
what is implemented by Linux, with the following output:

[the-experiment]: https://gist.github.com/amboar/56765a19694971890f20f58802af34b0

```
3 13:34:04 andrew@mistburn:~/src/scratch/lck (main) $ ./lck 
db.yAG5Ec/db opened for shared locks as O_RDONLY with fd: 4 (sfd)
db.yAG5Ec/db opened for exlusive locks as O_RDONLY with fd: 5 (xfd)
sfd (4): F_OFD_GETLK: F_UNLCK
sfd (4): Acquired shared lock
sfd (5): F_OFD_GETLK: F_UNLCK
xfd (5): F_OFD_GETLK: F_RDLCK
lck: F_SETLK failed: Bad file descriptor
```

The confirmation in-turn [yields this sketch implementation of the `libpldm`
instance ID API][iid-api-sketch].

[iid-api-sketch]: https://gist.github.com/amboar/b8e997de57b88222d010c99ace80bf03

[^1]: the only possible lock we can acquire given the file descriptor must be
    `O_RDONLY`
