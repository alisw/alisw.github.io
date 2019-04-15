---
title: Make AliRoot releases
layout: main
categories: infrastructure
---

Versioning scheme
=================
The current release process of AliRoot was agreed with the Data Preparation group. It has two main
release models:

* **Validated releases.** AliRoot tags are in the format **v5-08-13**. The minor number is increased
  and no letters are added after the name. Tags are from the master branch. The DPG validation is
  available [here](https://alijenkins.cern.ch/job/AliRootDPGValidation/)

* **Releases not requiring validation.** AliRoot tags are in the format **v5-08-13abc**, note the
  letters. They can be either Such releases are meant for quick fixes or fixes that are known not to
  require any validation.

The intermediate step before tagging is a "release candidate". Release candidates have their names
in the format **v5-08-13-rcXXXX** and they are exclusively published on `alice-nightlies.cern.ch`.

AliPhysics releases have the same name as AliRoot ones with **-01** appended, for instance
**v5-08-13abc-01**.

As agreed with the DPG, all important changes need to have a corresponding Jira "porting request"
associated.


Drafting releases with Jira
===========================
Releases are created on Jira by following [this
page](https://alice.its.cern.ch/jira/projects/ALIROOT?selectedItem=com.atlassian.jira.jira-projects-plugin%3Arelease-page&status=released-unreleased).

The same page is used to mark the release as "released".

As agreed with the DPG, release planning is done by opening a new Jira issue on the AliRoot project
as a "porting request", by specifying in the **Fix version(s)** field what is (are) the target
release(s).


Release validation
==================
The release validation is started from [here](https://alijenkins.cern.ch/job/AliRootDPGValidation/).
In general no field needs to be customized, except two:

* `JIRA_ISSUE`: must point to a valid Jira issue where the release validation bot will post the
  results
* `DONT_MENTION`: make sure it is **unchecked** if you want the relevant users to be added as
  watchers to the ticket

All the release validation scripts are [here](https://github.com/alisw/release-validation),
including the Jenkins pipeline and the scripts interacting with Jira.


Tagging, recipes, continuous integration
========================================
All AliRoot and AliPhysics releases must be tagged at the same time. In case of a validated release,
the tag should be the same of the validated "rc" one.

When tagging an AliRoot/AliPhysics release, the following two use cases will **not** use it
automatically:

* The Continuous Integration (pull requests) (see
  [configuration](https://gitlab.cern.ch/ALICEDevOps/ali-marathon/tree/master/aurora))
* The Daily Tags (see [Jenkins
  job](https://alijenkins.cern.ch/job/DailyBuilds/job/DailyAliPhysics/))

In order to make it work, one must download [alidist](https://github.com/alisw/alidist) and change
all the references to the old tags to the new ones. Something like this (the weird `sed` syntax
works on Mac and Linux):

```bash
sed -i.deleteme -e 's/v5-09-47/v5-09-48/g' *.sh; rm -f *.deleteme
git commit -avm 'Bump AliRoot to v5-09-48'
```

and then you can push and open a Pull Request.

When the pull request is merged, the C.I. will start using the new release for testing AliPhysics
pull requests, but the Daily Tags will still not use it. This is done by **tagging** alidist:

```bash
git checkout master
git pull
git tag 'AliPhysics-v5-09-48-01'
```

**It is essential that the tag name starts with `AliPhysics-`. The daily builder will build upon
the latest `AliPhysics-` tag found.**


Building AliRoot/AliPhysics releases
====================================
When a new release is made, and all the corresponding alidist PRs have been merged, [this Jenkins
job](https://alijenkins.cern.ch/job/BuildAliPhysicsRoot5And6/) takes care of building them.

One should normally not change any field there. It will use the master branch from alidist, and the
correct defaults for launching two parallel ROOT 5 and ROOT 6 builds are already set.

Based on the rules from [this
file](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish.conf), the releases will be
published on CVMFS.
