## Deprecated, Stable and Testing ABIs for `libpldm`

Developing and maintaining libraries is a very different ballgame to
applications. Internal functions of an application tend to have a closed set of
call-sites. Under these conditions refactoring is often straight-forward: Rework
your internal APIs and then clean up the resulting compiler errors. By contrast
libraries rarely have a closed set of call-sites for their APIs. This means
breaking an API impacts a potentially unknowable number of applications, and
makes for a bad experience for the library's users when they try to update.

That said, we need to develop library APIs somehow, and gain experience with
them in order to understand whether they're the right "shape". One approach is
to iterate on patches without merging them, and consider the API stable once the
patch introducing it is merged. However, a downside to this approach is it's
hard to scale effort across multiple people exercising the API in their library
or application, as everyone needs to locate and apply the latest version of the
patch before getting to work.

A separate problem to having to somehow cook up perfect APIs from the get-go is
that it's not always possible for internal functions to be marked static. The
result is that these internal symbols can appear in the symbol table for the
library. Sneaky users can start exploiting these symbols despite their absence
from the library headers, ossifying the internals of the implementation and
forcing the hand of anyone attempting to refactor the library's code.

A solution to this latter problem is to control what's visible to the linker
with GCC's [-fvisibility=hidden][man-1-gcc]:

[man-1-gcc]: https://www.man7.org/linux/man-pages/man1/gcc.1.html

> -fvisibility=[default\|internal\|hidden\|protected]
>
>   Set the default ELF image symbol visibility to the specified
>   option---all symbols are marked with this unless overridden
>   within the code.  Using this feature can very substantially
>   improve linking and load times of shared object libraries,
>   produce more optimized code, provide near-perfect API export
>   and prevent symbol clashes.  It is strongly recommended that
>   you use this in any shared objects you distribute.

As stated, adding `-fvisibility=hidden` to `CFLAGS` prevents all symbols
appearing in the symbol table by default. By enabling `-fvisibility=hidden` and
failing to mark internal symbols as visible we prevent our sneaky users from
ossifying the implementation. From there, annotating functions we wish to expose
with `__attribute__((visibility(default)))` makes them visible in the symbol
table and thus available for use.

One problem solved!

However, we're not finished with exploiting this capability yet. We can now
return to our original problem of wanting to merge new APIs for people to
develop against and provide feedback on, without committing to maintaining them
in their original form forever. We can do this by controlling the visibility of
these new APIs based on how we intend the built shared object to be used: If the
library is built for the purpose of exploring a new API, then mark the new API
as visible. Otherwise if the library is built for production purposes, do not
mark the new API as visible, which removes it from the symbol table.

This now gives us a library ABI that's divided into two classes:

1. Stable APIs that are always available in the symbol table
2. Testing APIs that are sometimes available in the symbol table

We can implement this in [meson][mesonbuild] with an option that takes an array
containing a constrained set of choices:

[mesonbuild]: https://mesonbuild.com/

```
option('abi', type: 'array', choices: ['stable', 'testing'], value: ['stable'])
```

We then hook this option into the build configuration with some configuration
data:

```
conf = configuration_data()
visible = '__attribute__((visibility(default)))'
# always expose stable APIs in the ABI regardless of abi option
conf.set('LIBPLDM_ABI_STABLE', visible)
if get_option('abi').contains('testing')
    conf.set('LIBPLDM_ABI_TESTING', visible)
else
    conf.set('LIBPLDM_ABI_TESTING', '')
endif
configure_file(output: 'config.h', configuration: conf)
```

Then in our library source we annotate our functions as we see fit:

```
#include "config.h"

...

LIBPLDM_ABI_STABLE
int pldm_instance_db_init(struct pldm_instance_db **ctx, const char *dbpath)
{
	struct pldm_instance_db *l_ctx;
	struct stat statbuf;
	int rc;

    ...
}

LIBPLDM_ABI_TESTING
pldm_requester_rc_t pldm_transport_poll(struct pldm_transport *transport,
					int timeout)
{
	struct pollfd pollfd;
	int rc = 0;

    ...
}
```

Now if we want access to `pldm_transport_poll()` we need to configure the build
with `meson setup -Dabi=stable,testing ...` before the symbol is available for
use. We have reached the happy place of being able to merge tentative APIs
without committing to them.

Finally, even then we're still not done. We can also take care of the other end
of symbol lifecycle: Deprecation. By adding a `LIBPLDM_ABI_DEPRECATED`
annotation and the associated machinery in the meson build configuration we give
ourselves a ratchet to move consumers forward: Disabling their access to to
deprecated symbols yields compilation errors in all the right places and allows
for straight-forward cleanup.

I've implemented the techniques above for `libpldm` in [libpldm: Explicit
deprecated, stable and testing ABI classes][libpldm-abi-control].

[libpldm-abi-control]: https://gerrit.openbmc.org/c/openbmc/libpldm/+/63974
