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

* [Setup your environment](#setup)
* [Listing active PR checker](#list-checkers)
* [Deploying a PR checker](#deploy-checker)
* [Scaling up the number of checkers](#scale-up-checkers)
* [Scaling down the number of checkers](#scale-down-checkers)
* [Restart a PR checker](#restart-checker)
* [Removing a PR checker](#remove-checker)
* [Inspecting the status of the checker](#inspect-checker)
* [Monitoring the checkers](#monitor-checkers)
* [macOS checkers](#mac-checkers)

## Setup your environment
{:setup}

Besides setting up your ALICE Aurora environment as described [here](https://alisw.github.io/infrastructure-aurora), you must be part of the `alice-aurora-mesosci` [egroup](https://egroups.cern.ch). Moreover, you will need to download
the set of recipes describing the jobs from the `ali-marathon` (bad name...) repository:

```bash
git clone https://gitlab.cern.ch/ALICEDevOps/ali-marathon
```

unless otherwise specified all the instructions of this page assume that you use `ali-marathon` as your working directory.

## Listing active PR checkers
{:list-checkers}

In order to see the list of the running prs you can do:

    $ aurora job list build/mesosci

where the resulting job names will follow the convention:

    build_<Package>_<alibuild-default>

## Scaling up the number of checkers
{:scale-up-checkers}

A simpler update operation is scaling up (or down) the number of checkers. This is also done with the `aurora add`
command which will add more instances, duplicating the configuration of one of the running ones.

* First of all you need to add more instances by doing:

```bash
aurora job add <ID>/0 <N>
```

where `<ID>/0` is your template configuration while `<N>` is the number of instances you want to add.

* Once they are warm and building pull requests correctly (use `aurora task ssh` to check), you can increase the worker pool size by setting `config/workers-pool-size` for each of them, you assign each a partition, and make all reporting their results.

```bash
aurora task ssh -l root <ID> "echo 8 > config/workers-pool-size"
aurora task ssh -l root <ID> 'echo {{ "{{mesos.instance" }}}} > config/worker-index'
```

* Finally you mark all the new instances as non-silent, so that they can start reporting results of the check:

```bash
aurora task ssh -l root <ID> "rm config/silent"
```

## Scaling down the number of checkers
{:scale-down-checkers}

Scaling down the number of checker might be useful to claim back resources in low activity periods. The logic is similar to the scaling up:

* Kill the checkers you do not need.

```bash
aurora job kill <ID>/4-7
```

* Resize the workers pool accordingly:

```bash
aurora task ssh -l root <ID>/0-3 "echo 4 > config/workers-pool-size"
aurora task ssh -l root <ID>/0-3 'echo {{ "{{mesos.instance" }}}} > config/worker-index'
```

## Updating a PR checker
{:deploy-checker}

Sometimes you want to update the cluster to a new version of the build script, e.g. to address some bug
or to provide a new feature. This can be done with no interruption of service by using the following recipe.

* Make sure that the number of instances you have in `aurora/continuous-builder.sh` matches the final number of instances you want to have.

* Check what are the differences between the currently running configuration and the one you have locally in `ali-marathon`:

```bash
aurora job diff <ID> aurora/continuous-integration.aurora
```

You should see only changes in the either ownership or resources.

* If you do not have more than one instance, make sure you scale to at least two instances and wait for the second one to be ready.

* Make sure the second half of the cluster can do the job of the first half, by halving the value in `config/workers-pool-size` and remapping the indices. E.g., if you have 8 instances:

```bash
# assuming 8 workers in total.
aurora task ssh -l root <ID>/0-3 "echo 1 > config/silent"
aurora task ssh -l root <ID>/4-7 "echo 4 > config/workers-pool-size"
aurora task ssh -l root <ID>/4-7 'echo $(({{ "{{mesos.instance" }}}} - 4)) > config/worker-index'
```

* Update the first half of the cluster and set it in silent mode:

```bash
# assuming 8 workers in total.
aurora update start <ID>/0-3 aurora/continuous-integration.aurora
aurora task run -l root build/mesosci/devel/build_O2_o2/0-3 "echo 1 > config/silent"
```

* Wait for it to be up and running by looking at the `.logs/*-continuos_integration/0/stderr` file.

```bash
aurora task run -l root build/mesosci/devel/build_O2_o2/0-3 "tail .logs/*continuos_integration/0/stderr"
```

If it's there, then the new builders are processing PRs. Mark them as non-silent and make the first half do all the work:

```bash
aurora task ssh -l root <ID>/0-3 "rm config/silent"
aurora task ssh -l root <ID>/0-3 "echo 4 > config/workers-pool-size"
aurora task ssh -l root <ID>/0-3 'echo {{ "{{mesos.instance" }}}} > config/worker-index'
```

* Now you can safely update the second half.

```bash
# assuming 8 workers in total.
aurora update start <ID>/4-7 aurora/continuous-integration.aurora
```

* Wait until all the new nodes are working correctly:

```
aurora task run -l root build/mesosci/devel/build_O2_o2/4-7 "tail .logs/*continuos_integration/0/stderr"
```

* Rebalance the workload on the whole pool:

```bash
aurora task ssh -l root <ID>/0-3 "rm config/silent"
aurora task ssh -l root <ID>/0-3 "echo 8 > config/workers-pool-size"
aurora task ssh -l root <ID>/0-3 'echo {{ "{{mesos.instance" }}}} > config/worker-index'
```

## Restarting a checker
{:restart-checker }

In some cases, builders need to be restarted. This will redeploy the same aurora configuration,
but the `ali-bot` scripts will be taken from HEAD and `continuous-builder.sh` will be run again.
Because each builder has ~30 minutes of warm up periods, you should follow the following procedure
to make it transparent to the user.

* Take the builder to be restarted out of the workers pool by changing the worker pool size to the old size -1. E.g. if
you have eight builders and you want to restart 4

```bash
aurora task ssh -l root <ID>/0-7 "echo 7 > config/workers-pool-size"
aurora task ssh -l root <ID>/0-2 'echo {{ "{{mesos.instance" }}}} > config/worker-index'
aurora task ssh -l root <ID>/4-7 'echo $(({{ "{{mesos.instance" }}}} - 1)) > config/worker-index'
aurora job restart <ID>/3'
```

* Wait for the new builder to warm up (use `aurora task ssh` to check), and then scale up the whole cluster back to the worker pool size:

```bash
aurora task ssh -l root <ID>/0-7 "echo 8 > config/workers-pool-size"
aurora task ssh -l root <ID>/0-7 'echo {{ "{{mesos.instance" }}}} > config/worker-index'
aurora task ssh -l root <ID>/0-7 "rm config/silent"
```

## Inspecting the checkers
{:inspect-checker}

Toi inspect checkers, you should probably get familiar with the command line client of aurora [documentation](http://aurora.apache.org/documentation/latest/reference/client-commands/).

Inspecting the checkers can be done interactively, using the `aurora task ssh <ID>/<instance>` command. 

E.g. use:

```bash
aurora task ssh -l root build/mesosci/devel/build_O2_o2/0
```

to ssh on the first instance where the job is running.

You can also run commands programmatically, using the `aurora task run <ID>/<instance-range>` command, e.g.:

```
aurora task run -l root build/mesosci/devel/build_O2_o2/0-1 ls
```

to run `ls` on the instances 0 and 1.

The checkers react to some special files which can be created in the sandbox. In particular:

* `config/silent` with non empty content means that the checker will not report issues to the github PR page associated with the checks being done.
* `config/debug` will force alibuild execution with `--debug` option.
* `config/workers-pool-size` will force the number of checkers currently active.
* `config/worker-index` will force the index of the current instance of the checker.

Moreover there are a few files which give you information about the current status of the system:

* `.logs/<step>/0/stderr`: stderr logs for <step> step in the job.
* `.logs/<step>/0/stderr`: stderr logs for <step> step in the job.
* `state/ready`: if present, the builder went throught the cache warm up phase

A few useful commands:

* To see the current pool configuration:

```bash
aurora task run -l root <ID> 'grep list .logs/*cont*/0/stderr | tail -n1'
```

* To update `ali-bot` on each node:

```bash
aurora task run -l root <ID> "cd ali-bot ; git pull --rebase"
```

* To check what it's happening inside the O2 build:

```bash
aurora task run -l root <ID> "tail -n1 sw/BUILD/O2-latest/log"
```

## Creating a new checker

Checkers configuration is in `ali-marathon/aurora/continuous-integration.aurora`. I order to add a new checker you need
to add a properly configured `CIConfig` instance to the `specs` list and start it with:

```bash
aurora job create build/mesosci/devel/<ci-name>
```

where `<ci-name>` is the value of the `ci_name` field in the associated `CIConfig` configuration object. Notice that in
order to post status updates for pull requests, the user `alibuild` must be a collaborator of the repository with write access.

## Monitoring the checkers
{:monitor-checkers}

Builders are monitored in [Monalisa](http://alimonitor.cern.ch/display?page=github/combined).
In particular you can use aliendb9 and look at the `github-pr-checker/github.com/github-api-calls`
metric to know how many API calls are being done by the system.

You can also get a detailed view the activity of the builders in our [Build Infrastructure Cockpit](https://alisw.cern.ch/cockpit).

## Builders for macOS
{:mac-checkers}

There is a dedicated process to run validation of pullrequests for macos.The routines therein are implemented within a [runscript](https://github.com/alisw/ali-bot/blob/master/ci/run-continuous-builder.sh) inside [ali-bot](https://github.com/alisw/ali-bot). This script currently needs to be started manually using a SREEN session to ensure it stays open. A launchd version is wip. 

```bash
SCREEN -dmS <session_name> /build/ali-bot/ci/run-continuous-builder.sh <config_file_argument>
```
There are four dedicated worker nodes:
* `alibuildmac01` `alibuildmac02` process pull requests for o2 and are launched by providing argument [`o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-0.sh) resp. [`o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-1.sh)
* `alibuildmac03` `alibuildmac04` process pull requests for alidist and are launched by providing argument [`alidist_o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-0.sh) and [`alidist_o2-1`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-1.sh)
* the actual builds are taking place in `/build/<config_file_argument>` 
