---
title: Release publishing
layout: main
categories: infrastructure
---
# Publishing of packages

Packages are published using `ali-bot/publisher` scripts. This happens in
different places depending if we are deploying on CVMFS or Alien. The procedure
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
