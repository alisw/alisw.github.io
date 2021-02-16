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

For the RPM case, in particular, the script runs in [Aurora](infrastructure-aurora) in the [`build/mesosdaq/prod/rpm_creation`](https://aliaurora.cern.ch/scheduler/mesosdaq/prod/rpm_creation) job. The script runs asynchrounously and
publishes packages as specified by the configurations for [parallel](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-rpms.conf) and [updateable](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-updatable-rpms.conf) RPMs.

## Troubleshooting

### Updating which aliPublish

In order to update aliPublish you need to ssh on the builder:

```bash
aurora task run build/mesosdaq/prod/rpm_creation/0 "cd ali-bot && git pull --rebase" 
```
