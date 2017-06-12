---
title: AliPhysics CI and Release Validation
layout: main
categories: developer
---

# Trying out a Release Validation

If a release validation fails, one can try out the release candidate by sourcing the
nightly environment from CVMFS:

    source /cvmfs/alice-nightlies.cern.ch/etc/login.sh
    export PATH=/cvmfs/alice-nightlies.cern.ch/bin:$PATH

then you can query and use nighly releases the usual way:

    alienv q | grep <name of the release>
    alienv enter <name of the release>
