---
title: Release publishing
layout: main
categories: infrastructure
---
# Publishing of packages

Packages are published using [ali-bot/publish](https://github.com/alisw/ali-bot/tree/master/publish) scripts. 
This happens in different places depending if we are deploying on CVMFS or Alien. The procedure
is automatic and in principle there is no need for babysitting it, however in
case of troubles you can do the following:

## Troubleshooting CVMFS

### Access to the publisher machines

* SSH as your username to `cvmfs-alice.cern.ch` or
  `cvmfs-alice-nightlies.cern.ch` depending on
  what you are publishing (ask to be added to the self managed lxcvmfs-alice for authorization).
* Follow the instructions given at the logon by the MOD
* Logs are in `~/publisher/log`

### Issue with missing ROOT libraries

For some yet to be understood reason sometimes ROOT is published with a `-<N>` revision, while `-<N+1>` is available.
This leads to errors like:

```
o2-analysistutorial-histograms-full-tracks: error while loading shared libraries: libGenVector.so.6.24: cannot open shared object file: No such file or directory
```

when running in hyperloop. In order to workaround the issue, for the moment, the solution is to add by hand a symlink to
the existing folder. This can be done by:

* `ssh cvmfs-alice.cern.ch`
* `sudo -i -u cvalice`
* Go to the ROOT folder and start a transaction

```
cd /cvmfs/alice.cern.ch/el7-x86_64/Packages/ROOT/
cvmfs_server transaction alice.cern.ch
```
* Find the missing ROOT and create a symlink to the next available one. E.g. `ln -s v6-24-02-24 v6-24-02-23`.
* **Close the transaction** `cd && cvmfs_server transaction alice.cern.ch`.

Things should be then back in business in a few minutes, time for CVMFS to propagate the changes everywhere.

## Troubleshooting ALIEN

See the [`publish-alien` Nomad job](https://alinomad.cern.ch/ui/jobs/publish-alien@default?desc=true&sort=submitTime).

## Troubleshooting RPMS

RPMS are also created using `aliPublishS3` in [Nomad](infrastructure-nomad).
[See the dedicated documentation.](infrastructure-rpms)

## Add new packages:

### CVMFS

Packages published on CVMFS can be configured via the [`aliPublish.conf`][aliPublish-conf] file. Apart from having your changes in master, no further action is required.

Builds of O2PDPSuite from the `async` branch are also published on CVMFS using [`aliPublish-async.conf`][async-conf].

In addition, the AliDPG package is handled specially and published to `noarch` using [`aliPublish-noarch.conf`][noarch-conf].

[aliPublish-conf]: https://github.com/alisw/ali-bot/blob/master/publish/aliPublish.conf
[async-conf]: https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-async.conf
[noarch-conf]: https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-noarch.conf

### RPMS

If you need to generate a new RPM package, the configuration is in `ali-bot/publisher/aliPublish-rpms.conf` and `aliPublish-updatable-rpms.conf`. Once you have updated it and merged in [alisw/ali-bot](https://github.com/alisw/ali-bot), the publisher will pick your changes up automatically.

## Start publishing on a new architecture

In order to publish Run 2 packages on another architecture on CVMFS, symlinks for the AliDPG package must be in place.

AliDPG is built on CentOS 8, but published to a special `noarch` architecture on CVMFS and AliEn.

AliDPG is always loaded explicitly, as the last package in a list, like `AliPhysics/v5-09-59a-01-1,AliDPG/prod-202209-01-1`. Becase `alienv` loads all packages from the same architecture, the availability of the AliPhysics package determines where we look for the AliDPG package. As AliDPG just provides a shell script and some uncompiled ROOT macros, it doesn't matter which architecture it was built on, so we can share AliDPGs between all architectures by putting them in `noarch`.

To facilitate this, the following symlinks must be created (remember to wrap this in a CVMFS transaction):

```bash
arch='<the new architecture you want to add, e.g. "el8-x86_64">'
cd /cvmfs/alice.cern.ch
ln -svnf ../../../noarch/Modules/modulefiles/AliDPG "$arch/Modules/modulefiles/AliDPG"
ln -svnf ../../noarch/Packages/AliDPG "$arch/Packages/AliDPG"
```

**TODO**: This hasn't been done for O2DPG yet; that package is pulled in by O2PDPSuite, so the publishing procedure is more complicated there.
