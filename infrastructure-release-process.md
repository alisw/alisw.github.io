---
title: Release process AliRoot / AliPhysics
layout: main
categories: infrastructure
---

This is to document the release process of AliRoot / AliPhysics.

## Prerequisites

You should be roughly familiar on [how to run a Github Actions workflow by hand](https://docs.github.com/en/actions/using-workflows/manually-running-a-workflow#running-a-workflow) and you should ask for permissions to do so.

Unless otherwise specified, always use the **master** branch of the workflow definition.

## Creating a release

The easiest way to create a release is to trigger the corresponding GitHub Actions in each repository. It is important to do this in the following order!

For each workflow, replace the `XXy` string in the release tag with the one you'd like to build. This will be two digits and a lowercase letter. The last workflow (in AliPhysics) will need the same tag as the other two, but with `-01` appended.

1. ["Prepare AliRoot/AliPhysics tag"][alidist tag] in alidist
2. ["Prepare AliRoot tag"][aliroot tag] in AliRoot
3. ["Prepare AliPhysics tag"][aliphysics tag] in AliPhysics
4. Start a [build in Jenkins][jenkins build].
   Change the following settings, replacing `XXy` with the tag name you created above:
   ```
   ALIDIST_SLUG=alisw/alidist@AliPhysics-v5-09-XXy-01
   ARCHITECTURE=slc7_x86-64
   PACKAGE_NAME=AliPhysics
   OVERRIDE_TAGS="AliPhysics=v5-09-XXy-01 AliRoot=v5-09-XXy"
   ```

[alidist tag]: https://github.com/alisw/alidist/actions/workflows/prepare-patch-release-branch.yml
[aliroot tag]: https://github.com/alisw/AliRoot/actions/workflows/main.yml
[aliphysics tag]: https://github.com/alisw/AliPhysics/actions/workflows/release.yml
[jenkins build]: https://alijenkins.cern.ch/job/build-any-ib/build?delay=0sec

## Patching old (≤ AliRoot-v5-08) releases

The transition from `AliRoot-v5-08` to `AliRoot-v5-09` involved a cleanup of large files, which were removed from the history of the official Github repository and left in a [CERN-hosted GitLab repository](https://gitlab.cern.ch/alisw/AliRoot-legacy).

For this reason any patch release on top of an old release should be tagged from that repository. In particular you have the following branches:

 * *v5-09-18b-01* (SLC5, alidist: legacy/v5-09-18)
 * *v5-08-13zm-01-cookdedx* (SLC5, alidist: IB/v5-08/prod)
 * *v5-08-13q-p14-01-cookdedx* (SLC5, alidist: IB/v5-08/prod)
 * *v5-08-13q-p14-01* (SLC5, alidist: IB/v5-08/prod)
