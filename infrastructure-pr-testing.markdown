---
title: Pull Request Builders
layout: main
categories: infrastructure
---

The ALICE Github PR builder is implemented as a set of independent
agents which looks at a portion of the phase space of the PRs available
for testing. This means that if there are 4 builders deployed each one
of them will build on average 1/4th of the PRs. This avoids the need
for a centralised scheduling service. Moreover, since the process of
checking a PR is not done once a given PR is built, but only when it's
merged, builders keep retrying to build even PR which were previously
successfully built. This is because the boundary condition of the PR
checking could be different (e.g. merge conflict might happen a
previously unresponsive service is back to production) and therefore the
test of a given PR is considered complete only when the PR is merged.

By default the builders will behave in the following manner:

- Wait for a configurable number of seconds
- Check if there is one or more untested PR. If yes, start building them
  and then go back to the starting point.
- Check if there are pull requests which were previously checked and had
  a failure. If yes, test one of them and then go back to the starting point.
- Check if there are pull requests which were previously checked and had did
  not have a failure. If yes, test one of them and then go back to the starting
  point.

We use Apache Aurora to deploy the builders on linux while builders on macOS are deployed 
directly.

# Essential operations guide

The following documentation applies to the Linux pull request checkers deployed with Aurora. 
Checkers on macOS do not use Aurora, instead they are run directly as scripts. 
**Before you do anything make sure you are familiar with [Apache Aurora command line
 interface](http://aurora.apache.org/documentation/0.16.0/reference/client-commands/)** and
make sure **you set up your [ALICE Aurora environment](https://alisw.github.io/infrastructure-aurora) correctly**.

* [Listing active PR checker](#list-checkers)
* [Deploying a PR checker](#deploy-checker)
* [Restart a PR checker](#restart-checker)
* [Removing a PR checker](#remove-checker)
* [Monitoring the checkers](#monitor-checkers)
* [macOS checkers](#mac-checkers)

## Listing active PR checkers
{: list-checkers}

In order to see the list of the running prs you can do:

    $ aurora job list build/mesosci

where the resulting job names will follow the convention:

    build_<Package>_<alibuild-default>

## Deploying a PR checker
{: deploy-checker}

Deploying the builders is done via the Apache Aurora instance. You can
find instructions on how to set it up [here](infrastructure-apache).
Because of the way the system is partitioned, we can ensure fully
covered scale up / scale down operations in the following way. Say that
we want to go from 4 workers to 8, one can start the new 4 workers by
doing:

    # Start the new 4 workers by doing.
    aurora update start build/mesosci/devel/aliphysics_github_ci/4-7 aurora/continuos-integration.aurora

    # Those workers do the same job as worker 2 and 3 in the previous configuration.
    # Once they are up and running, we can therefore safely redeploy 2 and 3.
    aurora update start build/mesosci/devel/aliphysics_github_ci/2-3 aurora/continuos-integration.aurora

    # The new 2 and 3 will do the job of the old 1, which we can now redeploy
    aurora update start build/mesosci/devel/aliphysics_github_ci/1 aurora/continuos-integration.aurora

    # Finally we restart 0
    aurora update start build/mesosci/devel/aliphysics_github_ci/0 aurora/continuos-integration.aurora

## Restarting the builders
{: restart-checker }

In some cases, builders need restart. The Aurora command for restarting a builder does not require
any `.aurora` file as an option, and the builder will be restarted as it was deployed.

All our production builders are under the tree `build/mesosci/devel`, so if you want to restart the
builder called `build_AliRoot_el6native` we would do:

    aurora job restart build/mesosci/devel/build_AliRoot_el6native

## Removing a PR checker
{: remove-checker}

First of all make sure the pr checker you want to kill uses the same job description as the one you have in `ali-marathon`. The only differences allowed are in the `Resource` and in the `Owner` fields.

This can be done with `aurora job diff <ID> aurora/continuos-integration.aurora`. E.g.:

```bash
aurora job diff build/mesosci/devel/build_O2_o2-dev-fairroot aurora/continuos-integration.aurora
```

If there are no significant changes (ask if in doubt) you can kill all the jobs with:

```bash
aurora job killall <ID>
```

or alternatively you can kill one instance with:

```bash
aurora job kill <ID>
```

## Monitoring the checkers
{:monitor-checkers}

Builders are monitored in [Monalisa](http://alimonitor.cern.ch/display?page=github/combined).
In particular you can use aliendb9 and look at the `github-pr-checker/github.com/github-api-calls`
metric to know how many API calls are being done by the system.

You can also get a detailed view the activity of the builders in our [Build Infrastructure Cockpit](https://alisw.cern.ch/cockpit).

## Builders for macOS
{: mac-checkers}

There is a dedicated process to run validation of pullrequests for macos.The routines therein are implemented within a [runscript](https://github.com/alisw/ali-bot/blob/master/ci/run-continuous-builder.sh) inside [ali-bot](https://github.com/alisw/ali-bot). This script currently needs to be started manually using a SREEN session to ensure it stays open. A launchd version is wip. 

```bash
SCREEN -dmS <session_name> /build/ali-bot/ci/run-continuous-builder.sh <config_file_argument>
```
There are four dedicated worker nodes:
* `alimacwn` `alimacwn2` process pull requests for o2 and are launched by providing argument [`o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-0.sh) resp. [`o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-1.sh)
* `alimacwn3` `alimacwn4` process pull requests for alidist and are launched by providing argument [`alidist_o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-0.sh) and [`alidist_o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-1.sh)
