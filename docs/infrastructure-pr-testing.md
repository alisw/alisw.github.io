---
title: Pull Request Builders
layout: main
categories: infrastructure
---

The ALICE Github PR builder is implemented as a set of independent agents, each of which looks at
the full list of pull requests available for testing and *claims* the work it is about to do. A
claim is taken on the pair (check, commit), and it is held cluster-wide, so two agents never build
the same thing and no centralised scheduling service is needed. Agents therefore have no identity
and need to know nothing about each other: adding one simply drains the queue faster, and when one
disappears its claim lapses and the work becomes available again.

Moreover, since the process of
checking a PR is not done once a given PR is built, but only when it's
merged, builders keep retrying to build even PR which were previously
successfully built. This is because the boundary condition of the PR
checking could be different (e.g. merge conflict might happen a
previously unresponsive service is back to production) and therefore the
test of a given PR is considered complete only when the PR is merged.

By default the builders will behave in the following manner:

- Refresh the check definitions from `ali-bot`'s master branch.
- Ask for the list of every buildable pull request, in the order the lister recommends (untested
  ones first, then previously failed ones, then the rest).
- Walk that list and build the first pull request whose claim they can take.
- Only if the whole list was already claimed or there was nothing to do, wait for a configurable
  number of seconds before starting over.

We use [Nomad](infrastructure-nomad.md) to deploy the builders on Linux and MacOS.
  
## Essential operations guide

See also: [the essential CI operations guide](infrastructure-nomad.md#essential-ci-operations-guide) for the ALICE Nomad deployment.

* [Adding a package to be tested](#adding-a-package-to-be-tested)
* [Setup your environment](#setup-your-environment)
* [Listing active PR checker](#listing-active-pr-checkers)
* [Scaling the number of checkers](#scaling-the-number-of-checkers)
* [Creating a new checker](#creating-a-new-checker)
* [Updating the PR checker](#updating-the-pr-checker-inner-loop)
* [Restarting a checker](#restarting-a-checker)
* [Inspecting the checkers](#inspecting-the-checkers)
* [Monitoring the checkers](#monitoring-the-checkers)

### Adding a package to be tested

Each checker deployed on Nomad can test multiple packages.
These are declared as `*.env` files [in a subdirectory of ali-bot](https://github.com/alisw/ali-bot/tree/master/ci/repo-config), following the pattern:

```
ali-bot/ci/repo-config/<mesos role>/<container>/<short name>.env
```

The `DEFAULTS.env` files in the directory tree are special -- they are sourced before the applicable `*.env` file, and can provide default values for various variables.

The most relevant variables you can set through the `*.env` file are:

- `ALIBUILD_DEFAULTS`: what to pass to alibuild using its `--defaults` flag
- `ALIBUILD_O2_TESTS`, `ALIBUILD_O2PHYSICS_TESTS`, `ALIBUILD_XJALIENFS_TESTS`: set these if O2, O2Physics and/or xjalienfs unit tests should be run as part of the check
- `CHECK_NAME`: the display name for this check, shown on GitHub
- `CI_NAME`: an internal name for this check
- `DEVEL_PKGS`: a newline-separated list of repositories to install as "development packages" for the build
- `DOCKER_EXTRA_ARGS`: if running builds in Docker, any additional args to pass to the container; e.g. use this to mount a `tmpfs` if needed
- `DONT_USE_COMMENTS`: only update GitHub statuses after the check completes, instead of commenting on each pull request as the alibuild user
- `INSTALL_ALIBOT`, `INSTALL_ALIBUILD`: a slug (`<org>/<repo>@<tag>#egg=<package>`) specifying the ali-bot and alibuild versions to install for this check
- `JOBS`: how many CPU cores to use for building
- `NO_ASSUME_CONSISTENT_EXTERNALS`: auto-generate a build identifier to keep builds of different pull requests apart. This uses more disk space overall, but can speed up rebuilds, since build artifacts are kept for each pull request.
- `PACKAGE`: the package (as declared in alidist) to build for the check
- `PR_BRANCH`: look for pull requests against this branch
- `PR_REPO`: look for pull requests in this repository
- `PR_REPO_CHECKOUT`: check out the repository to be tested into this directory
- `REMOTE_STORE`: use this as alibuild's `--remote-store`, for caching
- `TRUST_COLLABORATORS`: automatically test PRs from people who have committed to the same repo before, without waiting for approval from someone else
- `TRUSTED_USERS`, `TRUSTED_TEAM`: trust pull requests from these users (i.e. test them without requiring approval)
- `ONLY_RUN_WHEN_CHANGED`: a space-separated list of paths. The check will only be run if the given pull request contains any changes under any of these paths. If not, the check will be skipped and marked as successful with an appropriate message.

In order to add a new pull request check for a repository, and a checker for the required platform/container exists already, just add a `*.env` file in the appropriate directory in ali-bot.

If you add a new check, make sure to update the appropriate repository's GitHub action that cleans up broken statuses to include your new check. For example, in alidist, update [this file](https://github.com/alisw/alidist/blob/master/.github/workflows/clean-pr-checks.yml), or [this file](https://github.com/AliceO2Group/AliceO2/blob/dev/.github/workflows/clean-test.yml) for O2.

### Setup your environment

Set up your Nomad environment as described [here](infrastructure-nomad.md#setting-up-your-local-environment).

Job declarations for the pull request checkers are plain HCL files named `ci-*.nomad`, in the root
of the [ci-jobs](https://github.com/alisw/ci-jobs) repository.
See [this section](infrastructure-nomad.md#single-instance-jobs-eg-rsync-server) of the ALICE Nomad
docs for instructions on deploying them.

### Listing active PR checkers

In order to see the list of the running prs you can do:

```bash
nomad job status ci-
```

There is one job per platform family, e.g. `ci-linux-x86`, `ci-linux-arm`, `ci-linux-x86-gpu`,
`ci-linux-x86-release` and `ci-macos-arm`.

Which checks a job serves is not part of its name: each job sets `MESOS_ROLE` and a list of
containers in `CUR_CONTAINERS`, and reads its `.env` files from
`ali-bot/ci/repo-config/<role>/<container>/` for each of them. One job can therefore serve the
checks of several platforms at once, and moving a set of checks from one job to another is a
configuration change rather than a new job.

Each job will have one or more instances with the same configuration.
You can see these by running, e.g.:

```bash
nomad job status ci-linux-x86
```

You can check what each instance is doing by getting its allocation ID, then running:

```bash
nomad alloc logs -stderr -tail -f <alloc ID>
```

### Scaling the number of checkers

[See here for the procedure.](infrastructure-nomad.md#scaling-a-ci-job)

Running builders should not be affected by this operation.
This will only add builders if you increased the count, or delete some if you decreased it.

Because builders claim their work rather than owning a share of it, the remaining builders need no
notification that the count changed, and nothing has to be restarted.

### Creating a new checker

Most of the time there is no new Nomad job to create: adding a container to the `CUR_CONTAINERS`
list of an existing `ci-*.nomad` job is enough, and the new checks are then served by the builders
that job already runs.

A new job is only needed when the checks require something the existing jobs cannot offer, such as
a different CPU architecture, a GPU, or dedicated capacity. In that case, copy the closest existing
`ci-*.nomad` file, give it a new job name, and adjust its constraints, its `count` and its
`MESOS_ROLE`/`CUR_CONTAINERS` settings. Because a claim is cluster-wide and carries no job name, a
new job can be registered alongside an existing one and the two will serve the same checks without
building anything twice, which is the recommended way to roll a change out gradually.

Either way, you need to tell the checker what it should build and where it should look for incoming
pull requests.
Do this by creating configuration files in `ali-bot/ci/repo-config/<role>/<container>/`, e.g. in
`ali-bot/ci/repo-config/mesosci/slc10/`.

The easiest thing to do is to copy an existing `.env` file that you want to mimic and change the `CI_NAME` and `CHECK_NAME` variables.

For example, if you want to test PRs in the <https://github.com/AliceO2Group/AliceO2> repository by compiling O2Suite and running O2's and O2Physics' unit tests, adapt the following:

```bash
CI_NAME=build_O2_o2
CHECK_NAME=build/O2/o2
PACKAGE=O2Suite
ALIBUILD_DEFAULTS=o2
PR_REPO=AliceO2Group/AliceO2
PR_REPO_CHECKOUT=O2
PR_BRANCH=dev
TRUST_COLLABORATORS=true
DONT_USE_COMMENTS=1
DEVEL_PKGS="$PR_REPO $PR_BRANCH $PR_REPO_CHECKOUT
AliceO2Group/O2Physics master
alisw/alidist master"
ALIBUILD_O2_TESTS=1
ALIBUILD_O2PHYSICS_TESTS=1
```

The `.env` files are in shell syntax.
They are sourced by the CI builders, but some utilities (`list-branch-pr` and `ci-status-overview`) also parse them using Python's `shlex` library, so try not to use overly complex Bash features.
Try to use only literal strings for the `PR_REPO`, `PR_BRANCH`, `CHECK_NAME`, `TRUST_COLLABORATORS`, `TRUSTED_USERS` and `TRUSTED_TEAM` variables, since the Python tools parse these.
For other variables, feel free to use variable substitution with the usual shell syntax.

### Updating the PR checker inner loop

Sometimes one might need to update the inner loop of the PR checking, without having to restart the checker itself.
This is done automatically for any updates of the `ali-bot/ci/build-loop.sh` and `ali-bot/ci/build-helpers.sh` files, as well as of the `.env` files under `ali-bot/ci/repo-config/`.

Upgrades will be picked up at the next iteration of the PR builder, so you might still have to wait for the current one to finish before you can see your updates deployed.

Changes to the outer loop itself (`ali-bot/ci/claim-builder.sh` and `ali-bot/ci/claims.sh`) require
the job to be restarted.

### Restarting a checker

In some cases, builders need to be restarted.

[See here for the procedure.](infrastructure-nomad.md#stopping-and-restarting-ci-jobs)

This will redeploy the same configuration, but the `ali-bot` scripts will be taken from the master branch and `continuous-builder.sh` will be run again.

### Inspecting the checkers

You can stream the logs of any CI builder instance easily -- [see here how to do this](infrastructure-nomad.md#where-to-find-logs).

If you want to interactively log in to any instance, run:

```bash
nomad alloc exec <alloc ID> sh -c 'export TERM=xterm-256color HOME=$NOMAD_TASK_DIR PS1="\\u@\\h \\w \\\$ "; cd; exec bash -i'
```

This will give you a shell in the builder's home directory, where it stores its workarea.

Log files are in the `$NOMAD_ALLOC_DIR/logs` directory.

The checkers react to some special files, which can be created in their home directory.
In particular:

* `config/silent` with non empty content means that the checker will not report issues to the github PR page associated with the checks being done.
* `config/debug` will force additional debugging output.
* `config/workers-pool-size` will force the number of checkers currently active.
  This is normally created and updated automatically by Nomad.
* `config/worker-index` will force the index of the current instance of the checker.

### Monitoring the checkers

The CI system is monitored using a dedicated [Grafana dashboard](https://monit-grafana.cern.ch/d/7_5s-W27k/ci-overview?orgId=65).

Builders are monitored in [Monalisa](http://alimonitor.cern.ch/display?page=github/combined).
In particular you can use aliendb9 and look at the `github-pr-checker/github.com/github-api-calls`
metric to know how many API calls are being done by the system.

You can also get a detailed view the activity of the builders in our [Build Infrastructure Cockpit](https://datastudio.google.com/embed/reporting/f41f8c21-c617-4e7e-b14f-0f760c228be4/page/5FCOB). If you are interested in extending the reports, please contact us.

Some of the CI machines are also included in the [IT Grafana dashboard](https://monit-grafana.cern.ch/d/8KnQO4HMk/openstack-vms?orgId=1&var-project_name=ALICE%20Release%20Testing). Notice this includes also machines which do not run CI builds.

There is also a [dashboard that shows the status of each check and pull request](https://alice-ci-overview.cern.ch/).
In case of outages for larger problems, this can be useful to find failed PRs, or to find out when the problems started (since PRs are sorted newest-first).

## Troubleshooting
  
### Empty logs

Empty logs can happen in the case the build fails in some pre-alibuild steps and the harvesting script is not able or not allowed to fetch logs. In particular, due to the criticality of some operations, logs which might contain sensitive information are not retrieved, to avoid exposing them to unprivileged ALICE users. Maintainers of the build infrastructure can follow the instructions in the report to retrieve them.

More generally, build logs must never contain credentials in the first place: being able to read a
job's logs is a much more widely held permission than being able to read the credentials that job
uses. Note in particular that a shell running under `set -x` echoes every assignment it makes, so a
traced shell that sources a file of secrets prints all of them into its log; disable tracing around
any such step.
  
That said, the vast majority of the "missing logs issues" are due to the following two items:
  
* Wrong tag / commit / url when checking out the code. Make sure you review any tag and commit you changed in your PR to make sure they are actually existing. Possibly check by hand all the modified tags with `git clone -b <tag> <url>`.
* Issues with the infrastructure backend not directly under our control, in particular gitlab and github. You can review if there was any known issues by going to either the [github status page](https://www.githubstatus.com) or the [CERN Service Status Board](https://cern.service-now.com/service-portal?id=service_status_board).
  
Failing those (or if you are unsure) feel free to contact us directly either on mattermost or by adding a comment to your PR.
