---
author: Andrew
---

# Fixing Formatting CI Failures in OpenBMC Projects

Pushing patches for review to `gerrit.openbmc.org` automatically triggers CI
jobs on `jenkins.openbmc.org`. Almost always this triggers builds and a bunch of
linters to run over the change. Many of the linters are also formatters, such as
[prettier][], [black][] or [clang-format][].

[prettier]: https://prettier.io/
[black]: https://github.com/psf/black
[clang-format]: https://clang.llvm.org/docs/ClangFormat.html

As it stands any differences introduced by the linters causes a build failure.
This is checked with `git diff` in the build scripts:

```
...
    Formatting code under /data0/jenkins/workspace/ci-repository/openbmc/obmc-console
    markdownlint: cannot find .markdownlint.yaml; using /data0/jenkins/workspace/ci-repository/openbmc/openbmc-build-scripts/config/markdownlint.yaml
    prettier: cannot find .prettierrc.yaml; using /data0/jenkins/workspace/ci-repository/openbmc/openbmc-build-scripts/config/prettierrc.yaml
    Running clang_format
    Running markdownlint
README.md:1 MD041/first-line-heading/first-line-h1 First line in a file should be a top-level heading [Context: "## To Build"]
README.md:5 MD040/fenced-code-language Fenced code blocks should have a language specified [Context: "```"]
README.md:12 MD040/fenced-code-language Fenced code blocks should have a language specified [Context: "```"]
README.md:20 MD040/fenced-code-language Fenced code blocks should have a language specified [Context: "```"]
README.md:29 MD040/fenced-code-language Fenced code blocks should have a language specified [Context: "```"]
README.md:40 MD040/fenced-code-language Fenced code blocks should have a language specified [Context: "```"]
    Failed markdownlint; temporarily ignoring.
    Running prettier
README.mdREADME.md 50ms
CHANGELOG.mdCHANGELOG.md 10ms
.travis.yml.travis.yml 30ms
    Result differences...
diff --git a/CHANGELOG.md b/CHANGELOG.md
index e7619d4..985cbfc 100644
--- a/CHANGELOG.md
+++ b/CHANGELOG.md
@@ -3,15 +3,16 @@
 All notable changes to this project will be documented in this file.
 
 The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
-and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
+and this project adheres to
+[Semantic Versioning](https://semver.org/spec/v2.0.0.html).
 
 Change categories:
 
-* Added
-* Changed
-* Deprecated
-* Removed
-* Fixed
-* Security
+- Added
+- Changed
+- Deprecated
+- Removed
+- Fixed
+- Security
 
 ## [Unreleased]
Format: FAILED
...
```

So somehow we have to apply the same change that the CI has generated to our
local source tree. One approach is to grab [openbmc-build-scripts][] and run
the whole show yourself with `./scripts/build-unit-test-docker`.

[openbmc-build-scripts]: https://github.com/openbmc/openbmc-build-scripts

Alternatively, in most cases you can just copy the diff as emitted by the CI
output and plumb it straight into [git apply][git-apply]:

[git-apply]: https://git-scm.com/docs/git-apply

```
andrew@fedora:~/obmc-console $ git status
On branch server-socket-id
nothing to commit, working tree clean
```
```
andrew@fedora:~/obmc-console $ git apply <<EOF
> diff --git a/CHANGELOG.md b/CHANGELOG.md
index e7619d4..985cbfc 100644
--- a/CHANGELOG.md
+++ b/CHANGELOG.md
@@ -3,15 +3,16 @@
 All notable changes to this project will be documented in this file.
 
 The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
-and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
+and this project adheres to
+[Semantic Versioning](https://semver.org/spec/v2.0.0.html).
 
 Change categories:
 
-* Added
-* Changed
-* Deprecated
-* Removed
-* Fixed
-* Security
+- Added
+- Changed
+- Deprecated
+- Removed
+- Fixed
+- Security
 
 ## [Unreleased]
> EOF
```
```
andrew@fedora:~/obmc-console $ echo $?
0
```
```
andrew@fedora:~/obmc-console $ git status
On branch server-socket-id
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
        modified:   CHANGELOG.md

no changes added to commit (use "git add" and/or "git commit -a")
```
```
andrew@fedora:~/obmc-console $ git diff
diff --git a/CHANGELOG.md b/CHANGELOG.md
index 2fda54842128..8c0b0862e884 100644
--- a/CHANGELOG.md
+++ b/CHANGELOG.md
@@ -3,16 +3,17 @@
 All notable changes to this project will be documented in this file.
 
 The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
-and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).
+and this project adheres to
+[Semantic Versioning](https://semver.org/spec/v2.0.0.html).
 
 Change categories:
 
-* Added
-* Changed
-* Deprecated
-* Removed
-* Fixed
-* Security
+- Added
+- Changed
+- Deprecated
+- Removed
+- Fixed
+- Security
 
 ## [Unreleased]
 
andrew@fedora:~/obmc-console $ 
```

From here it's just a `git commit --all --amend` and a `git push ...` to resolve
the formatting pain.
