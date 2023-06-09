## Naming Functions in C

To steal from wikipedia, [C is an imperative procedural
language][wikipedia-c-lang]. Given the language's lack of formal support for
more abstract constructs like object-oriented classes, it's easy to reach the
view that its ecosystem is a grab-bag of loosely related functions. Developing
applications and libraries with this perspective can lead to choices that feel
kind of arbitrary. We need to be conscious that what we're doing is imposing a
structure on the code in order to organise our own thoughts and those of others.

[wikipedia-c-lang]: https://en.wikipedia.org/w/index.php?title=C_(programming_language)&oldid=1158956104

Something my lecturers at university drilled into me is that the the most
important thing in software is how you model relationships between - and the
operations that can be applied to - data. Focussing on the data relationships
and operations is what gives hope that we can develop abstractions that don't
leak. With this in mind, we can think about how to impose a structure that has
the feel of coherence and move away from a grab-bag of loosely related
functions.

Despite C and its imperative mood, it doesn't prevent us from considering things
in terms of objects with operations. We can still ask:

1. What type of object am I dealing with?
2. In what ways can I manipulate it?

In a language with explicit support for object-oriented design we would tend to
map the answers to the above onto a class with methods. We can also do this in
C, and it reads as "struct with supporting functions"[^1]. A fundamental
question under this split model[^2] is:

How can I immediately infer the relationship between a struct and a function
when reading the code?

As ever in C, we need to fall back to a naming convention to maintain the
feeling of coherence. My approach to this is to prefix the names of functions
that are operations on an object with the name of the object's struct (again, if
it has one[^1]). For example:

```c
struct foo {
    ...
};

int foo_frob(struct foo *ctx)
{
    ...
}

int foo_whack(struct foo *ctx)
{
    ...
}
```

The effect of this is as if we've defined a non-virtual method on a class in
e.g. C++[^3].

Note that in this light, the concept of "utility" functions tends to disappear.
By spending some time considering what data our utility function is operating
on, we have an opportunity to improve its name, where it's defined in the
project source code, *and* a reader's ability to comprehend its behaviour. It
follows that the strategy of naming symbols at object-level granuality also
helps with symbol namespace management inside large projects, improving
comprehension at the project level.

### A detour for libraries

As C has a flat symbol namespace, it's important for e.g. library authors to
put some effort towards [ensuring their library's symbols don't clash with any
other][drepper-goodpractice]. Having a library export a function whose symbol is
merely `crc8`, for instance, is asking for trouble. Collision avoidance is
typically implemented by choosing a symbol prefix and applying it pervasively,
in addition to the object prefix outlined above. Assuming we have some fictional
library `libquux` with our `foo` object, the combination yields:

[drepper-goodpractice]: http://www.akkadia.org/drepper/goodpractice.pdf

```c
struct quux_foo {
    ...
};

int quux_foo_frob(struct quux_foo *ctx)
{
    ...
}

int quux_foo_whack(struct quux_foo *ctx)
{
    ...
}
```

As a concrete example, picking on [libmctp][] we find it prefixes all its public
symbols with `mctp_`. By contrast, [libpldm][] (currently[^4]) inconsistently
applies a `pldm_` prefix. It exposes symbols that feel risky and also does not
tend to follow the symbol naming scheme above[^5].

[libmctp]: https://github.com/openbmc/libmctp
[libpldm]: https://github.com/openbmc/libpldm

(Thanks to [Zev][zevweiss] for review and feedback)

[zevweiss]: https://honk.bewilderbeest.net/u/zev

[^1]: The object may not be a struct. Consider a file descriptor, which is
    just an integer, yet it has a constructor (`open()`), a destructor
    (`close()`) and a set of operations to manipulate it (`read()`, `write()`,
    `lseek()` and so forth). This also serves to highlight how C just appears as
    a grab-bag of loosely related functions, because nothing in the naming
    indicates that they are operations on on the file descriptor class of
    objects.

[^2]: Structs and functions defined separately

[^3]: The extension of this is virtual methods, which translate to function
    pointers inside `struct foo`. The functions assigned to function pointers in
    a struct should also be prefixed even if they're declared `static`, as that
    way it's easy to identify the type of object the function is associated in a
    backtrace.

[^4]: [C library hygiene: Use symbol prefixes as a namespace mechanism to
    mitigate the risk of clashes][libpldm-issue-2]

[libpldm-issue-2]: https://github.com/openbmc/libpldm/issues/2

[^5]: Something that I intend to fix over time.
