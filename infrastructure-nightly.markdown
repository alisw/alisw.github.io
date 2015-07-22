---
title: Nightly builds
layout: page
categories: infrastructure
---

# Integration builds

So called integration (or nightly) builds are the pivotal part of a continuos
integration system. They get triggered as often as possible using the tip of
the branch of a repository and they usually include as many tests as possible which
can help making sure that the branch is in the best possible state.

In order to drive {{site.experiment}} builds we use [Jenkins](https://jenkins-ci.org) while
results of the builds are shown in [{{site.exp_prefix}}-ci.cern.ch]() (or it's
mirror [{{site.exp_prefix}}sw.gihub.io]()).

The job which schedules the build is
[schedule-all-ib](https://{{site.exp_prefix}}jenkins.cern.ch/job/schedule-all-ib/)
which in the end executes [`schedule.py`](https://github.com/alisw/ali-bot/blob/master/schedule.py)
which creates `.ini` files which are used to trigger the actual build jobs
[build-any-ib](https://{{site.exp_prefix}}jenkins.cern.ch/job/build-any-ib).
The configuration of which build needs to happen is defined in
[`config.yaml`](https://github.com/alisw/ali-bot/blob/master/config.yaml) in
particular in the section `integration_rules`.
