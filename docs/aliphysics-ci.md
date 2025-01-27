---
title: AliPhysics CI and Release Validation
layout: main
categories: developer
---

## Checking the status of the daily builds

Daily build are built with Jenkins. You can check the status of the current
build and the logs of previous one by going to:

<https://alijenkins.cern.ch/job/daily-builds/job/daily-aliphysics-github/>


## Trying out a Release Validation

If a release validation fails, one can try out the release candidate by sourcing the
nightly environment from CVMFS:

    source /cvmfs/alice-nightlies.cern.ch/etc/login.sh
    export PATH=/cvmfs/alice-nightlies.cern.ch/bin:$PATH

then you can query and use nighly releases the usual way:

    alienv q | grep <name of the release>
    alienv enter <name of the release>
