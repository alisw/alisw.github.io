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

* SSH to `alimonitor.cern.ch` as `monalisa` (ask Costin).
* Logs are in `~/publisher/log`

## Troubleshooting RPMS

RPMS are also created using `aliPublish`. The script runs in `aurora` and you can access it via:

```
aurora task ssh -l root build/mesosdaq/prod/rpm_creation/0
```

or by using the aliaurora web UI. 

## Add new packages:

### CVMFS

Packages published on CVMFS can be configured via the [aliPublish.conf](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish.conf) file. Apart from having your changes in master, no further action is required.

### RPMS

If you need to generate a new RPM package, the configuration is in `ali-bot/publisher/aliPublish-rpms.conf` and `aliPublish-updatable-rpms.conf`. Once you have updated it and merged in [alisw/ali-bot](https://github.com/alisw/ali-bot). You need to update the deployment by running:

```
aurora task run -l root build/mesosdaq/prod/rpm_creation/0 "cd ali-bot && git fetch origin && git reset --hard origin/master"
```

