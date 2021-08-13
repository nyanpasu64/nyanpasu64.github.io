---
layout: post
title: The missing guide for Arch Linux PKGBUILD's pkgver() version numbers
date: 2021-08-13 07:21 -0700
---

Pacman's version comparison algorithm was designed over a decade ago to properly sort many categories of real-world version numbers, and is now set in stone, quirks and all. Later on, the AUR developed `pkgver()` conventions and templates which turn Git commits into version numbers that would sort properly in Pacman. But what are Pacman's requirements for sorting real-world version numbers, how does Pacman's version comparison algorithm work, and how are AUR `pkgver()` built around the algorithm?

# How Pacman compares versions

`vercmp` is a command-line utility which takes two string arguments and compares them using Pacman's version comparison algorithm.

The `vercmp` executable exposes the algorithm used by Pacman to determine whether a different package version is newer than what you have currently installed. Sadly, https://man.archlinux.org/man/vercmp.8 (as well as the pacman manpage) is inadequate and fails to explain the algorithm, only providing a few examples.

## Requirements for comparing versions

Pacman needs to compare the versions of real-world software programs and its own conventions correctly:

- 1.0-beta < 1.0 (from semver)
	- pacman and vercmp fail to fulfill this requirement, because it interprets `-beta` as build metadata (see `parseEVR()` `-release`).
- 1.0beta < 1.0 (Arch labels pre-release packages as 1.0beta rather than 1.0-beta)
- 1.0 < 1.0.1
- 1.0.1 < 1.0.2

Pacman's version comparison algorithm also has incidental properties that I don't consider to be first principles. However, AUR `pkgver()` depend on certain ones to generate unusual-looking unintuitive version numbers that nonetheless sort properly in Pacman.

- 1.0 < 1.0.0 (I think they should be equal)
- alpha < beta < 1.0
- 1.0 < 1.0.alpha (it's strange that 1.0 < 1.0.alpha < 1.0.0)
- 1.0.alpha < 1.0.0
- 1.0.alpha < 1.0.1

## Algorithm implementation

The algorithm is implemented in `alpm_pkg_vercmp()` in the Pacman source code ([`:lib/libalpm/version.c`](https://gitlab.archlinux.org/pacman/pacman/-/blob/master/lib/libalpm/version.c)). The file is 260 lines of code, with multiple functions dedicated to different aspects of version comparison. The algorithm is written in raw C, with *glorious* null-terminated strings, and string slicing implemented via `const`-incompatible null byte insertion. 😿

### Epoch, version, and release

`parseEVR()` parses Arch package versions using the format `[epoch:]version[-release]`. More specifically, all characters after the last hyphen form the release (even if there are colons afterwards), and the epoch is "0" unless the first non-digit is a colon. If no epoch is present, the epoch is labeled as 0.

`parseEVR()` allows only numbers in the epoch field. It is usually absent, but can be used as a "major version" to ensure that newer program versions compare higher, even if the newer program's version number (stored in the version field) is *lower* than in older versions.

The release field is an optional location for "build metadata". A version with no release field is considered equal to otherwise-identical versions with any release field, but two otherwise-identical versions with different release fields use the release field to break ties.

### Comparing versions

Each field is then compared using `rpmvercmp()`. Missing epochs are assumed to be 0, and missing releases are assumed to be equal to any numbered release.

`rpmvercmp()` decomposes its input into "segments", where each segment starts with 0 or more "separator" characters (any non-alphanumeric character), which are followed by 1 or more "body" characters (each body contains either alphabetic characters or numeric characters, so "1a" is 2 segments). The input may be terminated by a "dangling" segment with only separator characters and no body (but realistic version numbers will not have a dangling segment).

This can be modeled as the regex `([^a-zA-Z0-9]* ([a-zA-Z]+ | [0-9]+) )* [^a-zA-Z0-9]*` more or less.

Both inputs are split into segments (including dangling segments), starting at the beginning. The algorithm loops over segments from both inputs, starting with the first segment from each, until either input runs out of segments entirely (one or both segments are absent).

Each loop iteration receives one segment from each version, for as long as both versions have segments remaining:

- All leading separators are trimmed off both segments. Results:
	- 1.1 = 1_1
- If either segment is empty after trimming separators (because it's a dangling segment), the loop breaks.
- If one segment started with more separator characters, it's a larger version. Note that the Pacman developers believe that realistic version numbers do not have multiple separator characters in a row, and Pacman isn't designed to handle this situation perfectly. Results:
	- 1 < .1 = _1 < ..1
	- 1.1 < 1..1
	- 1.a < 1..a
	- 1rev < 1.rev < 1..rev
	- a10 < a.10
- Alphabetic segments are sorted lexicographically, and sort before numeric segments (sorted numerically). Results:
	- a < aa < z < zz < 1 = 01 < 9 < 10

The function returns immediately if the loop finds a pair of segments that compare unequal. Otherwise the loop stops (without stripping separators) when one or both inputs reach the end of line, or breaks (after stripping separators) when one or both inputs reach a final dangling segment.

At this point, one of these is true:

- at least one version has no segment.
- no versions have missing segments, but at least one version has a dangling segment (causing both segments to be stripped, so at least one version *now* has no segment).

The segments are compared as follows:

- none = none
- none > alpha
- none < separator or number
- alpha < none
- separator or number > none

The algorithm is complete.

All dangling segments compare equal to one another, but come after "segment with text" and "no segment" and before "segment with number".

- 1a < 1 < 1.a < 1. = 1.. < 1.0

- '' < '.' = '..'
- 1 < 1. = 1_ = 1..

Unfortunately this algorithm has a cycle, caused by how more leading separators wins a version comparison (even if followed by a losing body) if both segments have bodies, but gets ignored if one or both segments are empty after trimming.

- 1.0 < 1..a (more leading separators wins since both segments have bodies)
- 1..a < 1. (leading separators ignored since 1. is empty after trimming, 'a' < '')
- 1. < 1.0 (leading separators ignored since 1. is empty after trimming, '' < '1')

Note that 1. and 1... are interchangeable, because the dangling separators get stripped out either way.

The Pacman developers commented, "Fun example :) Like I said, having multiple delimiters in a row doesn't make a lot of sense, so that is pretty much undefined behaviour"

## Testing the requirements

Dangling segments and multiple separators don't occur in real-world version numbers and can be ignored. Does this algorithm properly order real-world versions?

- 1.0beta < 1.0

Yes, "beta" < "".

- 1.0 < 1.0.1

Yes, "" < ".1".

- 1.0.1 < 1.0.2

Yes, ".1" < ".2".

- 1.0 < 1.0.alpha

Yes, "" < ".alpha"

# What is PKGBUILD and pkgver?

PKGBUILD files are shell scripts which define how to build a binary package. They contain a pkgver variable (if the PKGBUILD is specific to a version) or function (if the PKGBUILD fetches the latest commit of a VCS repo) serving as the version number of the PKGBUILD and the package produced. If pkgver is a function, the version of the package may change when the PKGBUILD is rerun and clones/pulls the target's VCS repo to a new commit.

If pkgver is a variable, then an unmodified pkgver means the package has not been updated. But if it's a function, in order for an AUR helper to determine if the installed package is outdated, it must re-clone/pull the VCS repo listed in source=() and call pkgver() again.

# Building a `pkgver()` so Pacman sorts Git repositories correctly

Git repositories in the wild have a lot of variance; some don't have tags, some have tags that sort properly, and some have tags in the wrong order. And some repositories start with no tags, but create tags later on when they make their first release.

## Requirements for comparing versions

What are the requirements for generating version numbers from a Git repository?

- As a repository without tags creates more commits, the version number should increase.
- When a repository creates its first release/tag, the version number should increase.
- As a repository with tags creates more commits, the version number should increase.
- If the most recent tag changes from 1.0 to 1.1, the version number should increase.
- If the most recent tag changes from 1.0 to 1.0.1, the version number should increase.

How can we achieve these criteria, given how Pacman works?

## Arch Wiki templates

[The Arch wiki](https://wiki.archlinux.org/index.php/VCS_package_guidelines#The_pkgver()_function) provides copy-paste snippets of example pkgver() functions, but fails to explain the underlying concepts (what `git describe` outputs, what the sed expression does, how the resulting expression is evaluated by `vercmp` and `pacman`).

### Untagged Git repositories

In a Git repo where the history of `master` has no tags, the recommended `pkgver()` counts commits:

```sh
pkgver() {
  cd "$pkgname"
  printf "r%s.%s" "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}
```

This produces a string `r{number of commits}.{commit hash}`.

Any letter would work equally well for the version comparison algorithm, `r` was chosen because it sounds like "revision". But what is the purpose of a letter?

### Tagged Git repositories

If the repo has tags like 0.2.5 which begin with a number (no leading "v" prefix like v0.2.5), `git describe --long --tags` can be used as the root source for the version:

```sh
pkgver() {
  cd "$pkgname"
  git describe --long --tags | sed 's/\([^-]*-g\)/r\1/;s/-/./g'
}
```

`git describe --long` produces a string with format `{most recent tag}-{commits since tag}-g{commit hash}`.

```sh
git checkout master
git describe --long --tags  # v2.4-25-ga240b43
git checkout v2.4  # or git checkout HEAD~25
git describe --long --tags  # v2.4-0-g51e51f4
```

The sed expression turns it into `{most recent tag}.r{commits since tag}.g{commit hash}`.

## Testing the requirements

- As a repository without tags creates more commits, the version number should increase.

"r###" < "r###+1"? Trivially so, as the "r" segment is the same, but the "number of commits" segment increases.

- When a repository creates its first release/tag, the version number should increase.

"r###" < "1.0.r###"? Yes. The version of the untagged repository starts with a "r" segment. The version of the tagged repository starts with a numeric segment (taken from the tag), which comes after.

- As a repository with tags creates more commits, the version number should increase.

"1.0.r###" < "1.0.r###+1"? Yes. "most recent tag" is unchanged, ".r" is unchanged, and "commits since tag" increases.

- If the most recent tag changes from 1.0 to 1.1, the version number should increase.

"1.0.r###" < "1.1.r###"? Yes. "most recent tag" increases.

- If the most recent tag changes from 1.0 to 1.0.1, the version number should increase.

"1.0.r###" < "1.0.1.r###"? Yes. "1"="1", ".0"=".0", and ".r" < ".1".

### Why not "."?

If tag-based versions were `{most recent tag}.{commits since tag}...`, then if the most recent tag changes from 1.0 to 1.0.1, the version would change from "1.0.###" to "1.0.1.###", where ".1" sorts *before* ".###" despite 1.0.1 being a newer program version.

This was first brought up by @diabonas:archlinux.org:

> You need it because otherwise 1.0.500 (where 500 is the revision count) would be newer than 1.0.1.30 (again, 30 is the revision count) - this doesn't happen with 1.0.r500, which is older than 1.0.1.r30 because a letter is always older than a digit

### Why not "r"?

> It's 1.0.1.r30 - the dot is important as 1.0.1r30 would be older than 1.0.1, but 1.0.1.r30 is newer - it's a revision after 1.0.1 after all. And yeah, 1.0r31 is a revision after 1.0, but before the next upstream release 1.0.1, whole 1.0.1.r31 is a revision after 1.0.1, so newer than 1.0.r30

## The Arch Wiki is wrong

The Arch wiki's stated requirements for generating version numbers are:

> It is recommended to have following version format: *RELEASE.rREVISION* where *REVISION* is a monotonically increasing number that uniquely identifies the source tree (VCS revisions do this).

The Arch wiki is wrong; given the "RELEASE.RELEASE.rREVISION" convention recommended by the wiki, for Pacman to properly identify older and newer packages, REVISION does not need to be globally monotonic, only within a given RELEASE. And the Arch wiki even breaks its own rules: the example "Git with tags" `pkgver()`'s REVISION is not monotonic except within a given RELEASE (Git tag).

Even if the Arch wiki was changed to say that REVISION needs to be monotonic within a given RELEASE, it states that `0.1.r456 > r454` but `0.1.456 < 454`, without explaining the algorithm used to compare revisions. This only serves to confuse the reader.
