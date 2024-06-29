---
title: Generation of RPMs for the O2 packages
layout: main
categories: infrastructure
---

# Rationale

Some clients of O2 software need to use RPMs to deploy O2 software and its
dependencies. We support two kind of RPM builds:

* Parallel installable RPMs, where the version of the packages is 
 actually part of the name, allowing parallel installation a-la CVMFS.
* Updatable RPMs, which behave like common RPMs for Centos / RHEL.

The RPM creation is not a rebuild of the tarballs which are usually deployed on CVMFS,
but a mere repackaging of the tarballs, via the same script which does the publishing
on CVMFS. The script itself is called `aliPublish` and as usual it's part of the 
[alisw/ali-bot](https://github.com/alisw/ali-bot/tree/master/publish) repository.

For the RPM case, in particular, the script runs in [Nomad](infrastructure-nomad.md) as a `publish-rpm-*` job.
The script runs asynchrounously and publishes packages as specified by the configurations for [parallel](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-rpms.conf) and [updateable](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-updatable-rpms.conf) RPMs.

## Nomad job management

Just like CI and Jenkins builders, RPM publishers are configured as [templated Nomad jobs](infrastructure-nomad.md#complex-templated-job-declarations-eg-ci).

They are declared using one YAML file for each architecture.
For instance, for the Alma 8 RPMs, see `ci-jobs/rpm-creation/el8.yaml`:

```yaml
---
arch: el8.x86_64
configs:
  - aliPublish-rpms-cc8.conf
  - aliPublish-s3-updatable-rpms.conf
```

This declares the architecture for which RPMs are created as `arch`, and the configuration files from `ali-bot/publish/` to use as `configs`.

Updates to `aliPublishS3` or the configuration files are picked up automatically between runs.
