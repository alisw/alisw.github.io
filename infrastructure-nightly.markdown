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

In order to drive {{site.experiment}} builds we use [Jenkins](https://jenkins-ci.org) while
results of the builds are shown in [https://alijenkins.cern.ch/job/DailyBuilds/](
https://alijenkins.cern.ch/job/DailyBuilds/).

The job which schedules the build is
[schedule-all-ib](https://{{site.exp_prefix}}jenkins.cern.ch/job/schedule-all-ib/)
which in the end executes [`schedule.py`](https://github.com/alisw/ali-bot/blob/master/schedule.py)
which creates `.ini` files which are used to trigger the actual build jobs
[build-any-ib](https://{{site.exp_prefix}}jenkins.cern.ch/job/build-any-ib).
The configuration of which build needs to happen is defined in
[`config.yaml`](https://github.com/alisw/ali-bot/blob/master/config.yaml) in
particular in the section `integration_rules`.

# Nightly repository

In order to speed up builds we maintain a nightly repository which caches
packages which were already built previously. This consist of an rsync server,
`rsync://repo.marathon.mesos/store` which is deployed via Marathon. You can
look at it's status in Marathon at
[https://{{site.exp_prefix}}marathon.cern.ch/#apps/%2Frepo]()

In case you need to deploy again the application, you can do so by doing:

    git clone https://gitlab.cern.ch/eulisse/ali-marathon.git
    cd ali-marathon 
    ./deploy rsync
