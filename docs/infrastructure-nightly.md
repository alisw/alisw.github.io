---
title: Nightly builds
layout: main
categories: infrastructure
---

# Integration builds

So called integration (or nightly) builds are the pivotal part of a continuos
integration system. They get triggered as often as possible using the tip of
the branch of a repository and they usually include as many tests as possible which
can help making sure that the branch is in the best possible state.

In order to drive ALICE builds we use [Jenkins](https://jenkins-ci.org) while
results of the builds are shown in <https://alijenkins.cern.ch/job/DailyBuilds/>.

## Troubleshooting

### `aliBuild` errors out with a conflict on S3

`aliBuild` uploads tarballs to [its remote store](#nightly-repository).
However, it cannot just upload one file -- it needs to upload the tarball itself under `TARS/<arch>/store/`, a "symlink" to it under `TARS/<arch>/<package>/` and various "symlinks" specifying the new tarball's dependencies under `TARS/<arch>/dist{,-direct,-runtime}/<package>/<package>-<version>-<revision>/`.

When aliBuild is killed while uploading tarballs, it can leave the repository in a partial state.

If this happens, you need to manually delete the partial symlinks:

```bash
arch=slc9_x86-64 package=O2 version=daily-20240313-0100 revision=1
s3cmd rm "s3://alibuild-repo/TARS/$arch/$package/$package-$version-$revision.$arch.tar.gz"
s3cmd rm -r "s3://alibuild-repo/TARS/$arch/dist/$package/$package-$version-$revision/"
s3cmd rm -r "s3://alibuild-repo/TARS/$arch/dist-runtime/$package/$package-$version-$revision/"
s3cmd rm -r "s3://alibuild-repo/TARS/$arch/dist-direct/$package/$package-$version-$revision/"
```

Since the main tarball in `TARS/<arch>/store/` is uploaded last, it likely does not exist in this case, but if it does, find its name using:

```bash
s3cmd ls -r "s3://alibuild-repo/TARS/$arch/store/" | grep -F "/$package-$version-$revision.$arch.tar.gz"
2024-03-13 00:28   1682231289  s3://alibuild-repo/TARS/slc9_x86-64/store/f2/f242f6e4ef58e8f15f3b410729bee6fdb562d790/O2-daily-20240313-0100-1.slc9_x86-64.tar.gz
```

The file name to delete is shown in the output above:

```bash
s3cmd rm 's3://alibuild-repo/TARS/slc9_x86-64/store/f2/f242f6e4ef58e8f15f3b410729bee6fdb562d790/O2-daily-20240313-0100-1.slc9_x86-64.tar.gz'
```

# Nightly repository

In order to speed up builds we maintain a nightly repository which caches packages which were already built previously.
This consist of a CERN S3 bucket, `s3://alibuild-repo/`, which is also accessible over HTTP at <https://s3.cern.ch/swift/v1/alibuild-repo/>.
You can browse it in [CERN OpenStack](https://openstack.cern.ch/project/containers/container/alibuild-repo), when switched to the "ALICE Release Testing" project.

In order to upload tarballs to the repository, use `--remote-store b3://alibuild-repo::rw` with `aliBuild`, while having the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` environment variables set.
