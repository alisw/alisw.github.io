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

## User guide

### Getting the client
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

### Configuring your environment
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
aurora job list build
```

### Run a simple application
{: simple-app}

We keep Aurora configuration files in:

<https://gitlab.cern.ch/ALICEDevOps/ali-marathon>

in the `aurora` folder. You can for example look at the "Hello world"
example:

    ali-marathon/aurora/hello.aurora

You can start it with:

    $ aurora job create build/mesostest/devel/hello hello.aurora
      INFO] Creating job hello
      INFO] Checking status of build/root/devel/hello
     Job create succeeded: job url=https://aliaurora.cern.ch/scheduler/root/devel/hello

then you can open the provided web page to look at the produced logs.

### Gotchas

In order to access the logs, you need to connect directly to the client machines, therefore you need to open a SOCKS tunnel inside the cluster and configure your browser to use it.

On Costin machines, for security reasons the log provider are not running, so you need to directly ssh inside them and lock at the filesystem.
