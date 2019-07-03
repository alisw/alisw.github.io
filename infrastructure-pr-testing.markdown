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


# Aurora operations

The following documentation applies to the pull request checkers deployed on Aurora. Checkers on
macOS do not use Aurora, instead they are run directly as scripts.

## Deploying the builders

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

In some cases, builders need restart. The Aurora command for restarting a builder does not require
any `.aurora` file as an option, and the builder will be restarted as it was deployed.

All our production builders are under the tree `build/mesosci/devel`, so if you want to restart the
builder called `build_AliRoot_el6native` we would do:

    aurora job restart build/mesosci/devel/build_AliRoot_el6native


# Monitoring the builders

Builders are monitored in [Monalisa](http://alimonitor.cern.ch/display?page=github/combined).
In particular you can use aliendb9 and look at the `github-pr-checker/github.com/github-api-calls`
metric to know how many API calls are being done by the system.


# Currently active builders

This is the list of builders that should be up and running.

## Production

| Repository           | GitHub check name        | Aurora job               | # |
|----------------------|--------------------------|--------------------------|---|
| alisw/AliRoot        | build/AliRoot/el6native  | build_AliRoot_el6native  | 1 |
| alisw/AliRoot        | build/AliRoot/macos      | _not on Aurora_          | 1 |
| alisw/AliRoot        | build/AliRoot/release    | build_AliRoot_release    | 1 |
| alisw/AliPhysics     | build/AliPhysics/release | build_AliPhysics_release | 4 |
| AliceO2Group/AliceO2 | build/O2/o2              | build_O2_o2              | 1 |
| AliceO2Group/AliceO2 | build/O2/o2-dev-fairroot | build_O2_o2-dev-fairroot | 1 |
| AliceO2Group/AliceO2 | build/O2/macos           | _not on Aurora_          | 1 |
| AliceO2Group/AliceO2 | build/o2checkcode/o2     | build_o2checkcode_o2     | 1 |

Notes:

* AliRoot "el6native" check does not run on SLC6, but it runs in C++98-compatible mode. This is
  still required until the end of Run 2 for compatibility with Point-2 nodes.
* AliRoot builds on macOS use ROOT 6.
* The AliRoot "release" check runs the guntest as well. In some cases, tests might fail if the
  reference OCDB hasn't been updated, and this is correct.
* AliPhysics checks use the latest AliRoot tag as reference (not master), as configured in the
  defaults `prod-latest`.

## Builders for macOS

There is a dedicated process to run validation of pullrequests for macos.The routines therein are implemented within a [runscript](https://github.com/alisw/ali-bot/blob/master/ci/run-continuous-builder.sh) inside [ali-bot](https://github.com/alisw/ali-bot). This script currently needs to be started manually using a SREEN session to ensure it stays open. A launchd version is wip. 

```bash
SCREEN -dmS <session_name> /build/ali-bot/ci/run-continuous-builder.sh <config_file_argument>
```
There are four dedicated worker nodes:
* `alimacwn` `alimacwn2` process pull requests for o2 and are launched by providing argument [`o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-0.sh) resp. [`o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-1.sh)
* `alimacwn3` `alimacwn4` process pull requests for alidist and are launched by providing argument [`alidist_o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-0.sh) and [`alidist_o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-1.sh)
