---
author: Andrew
---

## Bitbake and Git Submodules Done Right

A part of OpenBMC has a use-case where we needed to build a project composed of
multiple repositories arranged as submodules with a submodule tree depth
greater than one. The catch is that we don't need to initialise submodules
below the first layer, and the approach of initialising them anyway means a
large time and bandwidth penalty downloading unnecessary information.

Bitbake has many fetchers for source code, including several fetchers for
git repositories. The main git fetcher, associated with the `git://` bitbake
URI scheme, performs just the actions necessary to set up a local clone of a
target repository and does not initialise submodules. A second git fetcher,
associated with the `gitsm://` scheme, uses the main git fetcher to grab the
primary repository and then performs exhaustive recursive submodule
initialisation to acquire related repositories.

As it turns out the most effective way to solve the problem isn't to [use a
hacked up gitsm:// fetcher][1], but rather explicit specification of the
submodules of interest in `SRC_URI` of the project's bitbake recipe. Below is
the [example kindly provided by Alexander Kanavin on the bitbake list][2]:

```
SRCREV_FORMAT = "glm_gli"
SRCREV_glm = "01f9ab5b6d21e5062ac0f6e0f205c7fa2ca9d769"
SRCREV_gli = "8e43030b3e12bb58a4663d85adc5c752f89099c0"
SRCREV = "ae0b59c6e2e8630a2ae26f4a0b7a72cbe7547948"

SRC_URI = "git://github.com/SaschaWillems/Vulkan.git \
           git://github.com/g-truc/glm;destsuffix=git/external/glm;name=glm \
           git://github.com/g-truc/gli;destsuffix=git/external/gli;name=gli \
           file://0001-Don-t-build-demos-with-questionably-licensed-data.patch \
"
```

While it doesn't reuse any of git's built-in submodule infrastructure it very
effectively meets our corner case of not wanting recursion but still needing
the necessary source to be present.

[1]: http://lists.openembedded.org/pipermail/bitbake-devel/2019-August/020297.html
[2]: http://lists.openembedded.org/pipermail/bitbake-devel/2019-August/020301.html
