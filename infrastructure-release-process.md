---
title: Release process
layout: main
categories: infrastructure
---

This is to document the release process of AliRoot / AliPhysics.

## Creating a release

The easiest way to create a release is to trigger the corresponding GitHub Actions in each repository. It is important to do this in the following order!

For each workflow, replace the `XXy` string in the release tag with the one you'd like to build. This will be two digits and a lowercase letter.

1. ["Prepare AliPhysics tag"][alidist tag] in alidist
2. ["Prepare AliRoot tag"][aliroot tag] in AliRoot
3. ["Prepare AliPhysics tag and start build"][aliphysics build] in AliPhysics

The last of these should be set up to automatically start a build on Jenkins automatically.

[alidist tag]: https://github.com/alisw/alidist/actions?query=workflow%3A%22Prepare+AliPhysics+tag%22
[aliroot tag]: https://github.com/alisw/AliRoot/actions?query=workflow%3A%22Prepare+AliRoot+tag%22
[aliphysics build]: https://github.com/alisw/AliPhysics/actions?query=workflow%3A%22Prepare+AliPhysics+tag+and+start+build%22

Once the release is built, you can create a pull request in `alidist` from the branch `AliPhysics-<aliroot-tag>-01-patches` so that `master` uses a specific release. You might also want to move the dailies to such a release.

## Patching old (≤ AliRoot-v5-08) releases

The transition from `AliRoot-v5-08` to `AliRoot-v5-09` involved a cleanup of large files, which were removed from the history of the official Github repository and left in a [CERN-hosted GitLab repository](https://gitlab.cern.ch/alisw/AliRoot-legacy).

For this reason any patch release on top of an old release should be tagged from that repository. In particular you have the following branches:

 * *v5-09-18b-01* (SLC5, alidist: legacy/v5-09-18)
 * *v5-08-13zm-01-cookdedx* (SLC5, alidist: IB/v5-08/prod)
 * *v5-08-13q-p14-01-cookdedx* (SLC5, alidist: IB/v5-08/prod)
 * *v5-08-13q-p14-01* (SLC5, alidist: IB/v5-08/prod)
