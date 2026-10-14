---
title: Generation of RPMs for the O2 packages
layout: main
categories: infrastructure
---

## Rationale

Some clients of O2 software need to use RPMs to deploy O2 software and its
dependencies. We support two kind of RPM builds:

* Parallel installable RPMs, where the version of the packages is 
 actually part of the name, allowing parallel installation a-la CVMFS.
* Updatable RPMs, which behave like common RPMs for Centos / RHEL.

The RPM creation is not a rebuild of the tarballs which are usually deployed on CVMFS,
but a mere repackaging of the tarballs, via the same script which does the publishing
on CVMFS. The script itself is called `aliPublish` and as usual it's part of the 
[alisw/ali-bot](https://github.com/alisw/ali-bot/tree/master/publish) repository.

For the RPM case, in particular, the script runs in [Nomad](infrastructure-nomad.md) as the
`publish-rpm` job.
The script runs asynchrounously and publishes packages as specified by the configurations for
parallel installable ([Alma 9](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-rpms-cc9.conf),
[Alma 10](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-rpms-cc10.conf)) and
[updateable](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-s3-updatable-rpms.conf) RPMs.

## Nomad job management

RPMs for every architecture are published by a single Nomad service job, declared as plain HCL in
`ci-jobs/rpm-creation/rpm-creation.nomad` and deployed with `nomad job run` like any other
non-templated job.

The job runs [`ali-bot/publish/run-all-rpms.sh`](https://github.com/alisw/ali-bot/blob/master/publish/run-all-rpms.sh),
passing it a comma-separated list of architectures followed by one or more configuration files from
`ali-bot/publish/`, for example:

```sh
run-all-rpms.sh el9.x86_64,el10.x86_64 \
  aliPublish-rpms-cc9.conf \
  aliPublish-rpms-cc10.conf \
  aliPublish-s3-updatable-rpms.conf
```

The architecture list does *not* select what gets published: `aliPublish` publishes whatever each
configuration's `architectures:` map declares.
The list only says which per-architecture repository listings to diff, in order to announce newly
published RPMs, and which per-architecture status files to clear once publishing is done.
It should therefore mention every architecture the given configurations can produce.

The script loops, reading each configuration in turn, so updates to `aliPublishS3` or the
configuration files are picked up automatically between runs.
