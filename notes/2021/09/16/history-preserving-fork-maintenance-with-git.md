## History-preserving fork maintenance with git

Working with long-term forks in git can be painful. A fundamental choice exists
around which strategy should be used to manage downstream-only patches:

1. Merge upstream into your fork periodically, interleaving patches, obscuring
the number and location of your downstream changes

2. Squash-merge upstream into your fork, destroying its history for the benefit
of minimising the noise

3. Rebase your downstream changes then force-push, impacting everyone else
consuming your fork

All three have benefits and detractions. Picking any one needs to be done by
considering the trade-offs in the context of what you're trying to achieve.

There is a fourth strategy that goes slightly off the beaten track - a history
preserving rebase. The benefit of this approach is your downstream patch stack
always exists at the top of your history like it would for strategy 3 above,
but it removes the requirement for a force-push (and the associated pain for
users of your fork). These two properties are achieved by using both a merge
_and_ a rebase, ensuring the old state of your tree is reachable from the tip
of master. Again, this means users of your tree will not have to adjust their
workflow to deal with the fallout of a force-push as they would in strategy 3.

The mechanics of this history-preserving rebase strategy start with a merge of
our fork\'s master into upstream\'s master using the [`ours` merge strategy][1],
followed by the rebase:

```
ours
    This resolves any number of heads, but the
    resulting tree of the merge is always that of the
    current branch head, effectively ignoring all
    changes from all other branches. It is meant to be
    used to supersede old development history of side
    branches. Note that this is different from the
    -Xours option to the _recursive_ merge strategy.
```

[1]: https://git-scm.com/docs/git-merge#_merge_strategies

Assuming we start with a clean fork of upstream:

```
$ git clone ...
$ cd ...
```

We begin by marking the upstream commit for easy reference:

```
$ git tag base
```

Over time we add patches to our fork on top of the `base` tag. Eventually we
want to integrate changes from upstream (e.g. upstream make a new release):

```
$ git fetch origin
$ git checkout origin/master
$ git merge -s ours master
$ git tag next-base
```

At this point we've joined the histories of our fork and upstream, but
discarded any downstream state. We're ready to forward-port our downstream
patches onto the new upstream state:

```
$ git checkout master
$ git rebase --onto next-base base master
$ git tag -f base next-base
$ git tag -d next-base
```

An important detail here is the use of `git rebase --onto`. `--onto` allows us
to transplant a series of patches onto an arbitrary new base by disconnecting
the series from its history. In effect it's a range-based `cherry-pick` with a
tidier command-line. Disconnecting the patch series in this way prevents `git`
from analysing the move in terms of history as it would in a normal rebase,
which means it cannot "see" that the patches already exist in the right side of
the `ours` merge. Instead the test becomes whether the patch applies, which it
should, modulo any conflicts with the new upstream code.
