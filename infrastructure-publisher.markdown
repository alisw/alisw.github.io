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

* SSH as your username to `cvmfs-alice.cern.ch` or
  `cvmfs-alice-nightlies.cern.ch` depending on
  what you are publishing.
* Follow the instructions given at the logon by the MOD
* Logs are in `~/publisher/log`

## Troubleshooting ALIEN

* SSH to `alimonitor.cern.ch` as `monalisa`
* Logs are in `~/publisher/log`

## Troubleshooting RPMS

RPMS are also created using `aliPublish`. The script runs in `aurora` and you can access it via:

```
aurora task ssh -l root build/mesosdaq/prod/rpm_creation/0
```

or by using the aliaurora web UI. 

## Add new packages:

### CVMFS

Packages published on CVMFS can be configured from [https://github.com/alisw/ali-bot/tree/master/publish/aliPublish.conf](). Apart from having your changes in master, no further action is required.

### RPMS

If you need to generate a new RPM package, the configuration is in `ali-bot/publisher/aliPublish-rpms.conf` and `aliPublish-updatable-rpms.conf`. Once you have updated it and merged in [alisw/ali-bot](https://github.com/alisw/ali-bot). You need to update the deployment by running:

```
aurora task run -l root build/mesosdaq/prod/rpm_creation/0 "cd ali-bot && git fetch origin && git reset --hard origin/master"
```

