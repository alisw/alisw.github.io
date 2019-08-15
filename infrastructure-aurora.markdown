---
title: Using Apache Aurora
layout: main
categories: infrastructure
---

It's now possible to use [Apache Aurora](https://aurora.apache.org) to
schedule long running jobs, cron jobs, and ad hoc job on the ALICE Mesos
infrastructure.

ALICE Aurora instance can be found at:

https://aliaurora.cern.ch

It is used for a number of jobs an in particular for the continuous builders
of the pull request.

Access is allowed to ALICE members who are part of the
`alice-aurora-users` egroup. You can subscribe to it by going to
the usual [egroup page](https://egroup.cern.ch).

# User guide

* [Getting the client](#get-the-client)
* [Configuring your environment](#configuring)
* [Run a simple application](#simple-app)

## Getting the client
{:get-the-client}

The GUI is only a read-only view on the state of the jobs
running on the cluster. If you want to interact with Aurora
itself you will need the aurora client.

You can get a binary distribution of the Aurora client at:

<https://github.com/alisw/aurora/releases/tag/0.16.0-alice2>

Alternatively you can download the sources and build it with:

```bash
./pants binary src/main/python/apache/aurora/kerberos:kaurora
cp dist/kaurora aurora
```

If you use homebrew, you can also do:

```bash
brew install ktf/system-deps/alice-aurora
```

## Configuring your environment
{:configuring}

Access is allowed to ALICE members who are part of the
`alice-aurora-users` egroup. You can subscribe to it by going to
the usual [egroup page](https://egroup.cern.ch).

The authentication mechanism uses kerberos, so you should make sure
you have a valid kerberos ticket in the CERN.CH realm. You can verify that 
by doing:

```bash
$ klist
```

which should result in something like:

```
Credentials cache: API:5BD2DB44-B9A8-48BD-9CD1-47078B7D00A9
        Principal: <your-cern-user-name>@CERN.CH

  Issued                Expires               Principal
Aug 13 14:26:57 2019  Aug 14 00:26:57 2019  krbtgt/CERN.CH@CERN.CH
```

In order to reach the Aurora cluster, you need to configure how to
access it. This is done by creating a file `~/.aurora/clusters.json`:

    [{
      "name": "build",
      "scheduler_uri": "https://aliaurora.cern.ch",
      "auth_mechanism": "KERBEROS",
      "slave_run_directory": "latest",
      "slave_root": "/build/mesos"
    }]

If everything is setup as expected you should be able to get a list of
jobs by doing:

```
$ aurora job list build
```

If you are an admin, you should also verify that `aurora_admin` also works.

```
$ aurora_admin get_cluster_config aliaurora
{"auth_mechanism": "KERBEROS", "name": "aliaurora", "scheduler_uri": "https://aliaurora.cern.ch"}%
```
## Run a simple application
{: simple-app}

We keep Aurora configuration files in:

<https://gitlab.cern.ch/ALICEDevOps/ali-marathon>

in the `aurora` folder. You can for example look at the "Hello world"
example:

    ali-marathon/aurora/hello.aurora

You can start it with:

    $ aurora job create build/mesostest/devel/hello hello.aurora
      INFO] Creating job hello
      INFO] Checking status of build/mesostest/devel/hello
     Job create succeeded: job url=https://aliaurora.cern.ch/scheduler/mesostest/devel/hello

This will start on the cluster a (somewhat) long running job. You can open the provided web page to look at the workarea.
If you want to interact with the job in an ad-hoc manner, e.g. to debug what it is doing or force some action to it, you can SSH to the machine running the job by doing:

    $ aurora task ssh build/mesostest/devel/hello/0
    
which will ssh for you in the sandbox for the job on the machine it is running. Notice that you might still have to `docker exec` yourself into the container to get the correct environment. Alternatively you can execute a one off job by doing:

    $ aurora task run build/mesostest/devel/hello/0 "hostname > foo.txt"
    
You can find more information about the available commands in the [official Aurora documentation]( http://aurora.apache.org/documentation/0.16.0/reference/client-commands/).
    
## Managing build infrastructure pull request checkers
{: pr-checkers}

Pull request checkers run as long running jobs. A given repository for which we test the PRs under certain configurations is associated with a given aurora job and has one or more workers. Each worker takes a fraction of the hash phase space for git commits and keeps rebuilding all the pull requests which have the tip of their feature branch as part of that fraction. E.g. if you have two builders, a pull request whose tip commit is "08312648716746174678326478174" will be handled by worker 0, while "8487897843894827483274329874832" will be handled by worker 1. This allows us to avoid a central scheduling server and to make sure we keep testings pull requests as other ones get merged or external conditions change.

### Listing PR checkers

In order to see the list of the running prs you can do:

    $ aurora job list build/mesosci

where the resulting job names will follow the convention:

### Removing a PR checker

First of all make sure the pr checker you want to kill uses the same job description as the one you have in ali-marathon. This can be done with `aurora job diff <ID> aurora/continuos-integration.aurora`. E.g.:

```bash
aurora job diff build/mesosci/devel/build_O2_o2-dev-fairroot aurora/continuos-integration.aurora
```

If the differences are only in the reported `Resources` or in the `Owner` you can proceed with the killing by doing:

```bash
aurora job killall <ID>
```

## Gotchas & issues

* On Costin machines, for security reasons the log provider are not running, so you need to directly ssh inside them and look at the filesystem.

* On some systems, the CERN CA is not available by default. You can overcome this by either:
  * Go to <https://ca.cern.ch> and install all the required CA certificates. In general this is what is needed on macOS.
  * Obtain it via `scp lxplus.cern.ch:/etc/ssl/certs/ca-bundle.crt ca-bundle.crt` and doing `export REQUESTS_CA_BUNDLE=$PWD/ca-bundle.crt`.
  * Installing the `CERN-CA-certs` package and doing `export REQUESTS_CA_BUNDLE=$PWD/ca-bundle.crt`.

* On some systems, kerberos gives a token for the actual backend name, rather than aliaurora. You can check that by doing klist and you will see `HTTP/alibuild-frontend01.cern.ch@CERN.CH`:

```bash
Credentials cache: API:B7FC3DD4-738F-417E-B2FA-92B2CCA9590C
        Principal: eulisse@CERN.CH

  Issued                Expires               Principal
Aug 14 15:50:42 2019  Aug 15 01:50:42 2019  krbtgt/CERN.CH@CERN.CH
Aug 14 15:50:46 2019  Aug 15 01:50:42 2019  HTTP/alibuild-frontend01.cern.ch@CERN.CH
```

In order to fix this you will have to change your kerberos configuration, usually found in `/etc/krb5.conf`, and add `rdns = false` in the `[libdefaults]` stanza. 

* On mac the most reliable way to operate is:
  * First do `kdestroy`
  * Then do `kinit`
  * Finally `aurora job list`
  * The three steps above should guarantee that Firefox has the correct token.
  
* SSH / running ad-hoc tasks on some of the machines requires extra work, most notably on Costin's `alientest` machines which do not use kerberos. In order to be able to login, you need to ask the admin of such machines to add your SSH key to 
the `.authorised_keys` of the user which has the same name as the role for the job (e.g. `mesostest` for `build/mesostest/devel/hello`).
