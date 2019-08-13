---
title: Using Apache Aurora
layout: main
categories: infrastructure
---

# Setting up your environment

It's now possible to use [Apache Aurora](https://aurora.apache.org) to
schedule long running jobs, cron jobs, and ad hoc job on the ALICE Mesos
infrastructure.

ALICE Aurora instance can be found at:

https://aliaurora.cern.ch

access is allowed to ALICE members who are part of the
`alice-aurora-users` egroups. If you are denied access, please ask
amins. FIXME: not true, only Giulio and Michael for now.

## Getting the client

You can get a binary distribution of the Aurora client at:

<https://github.com/alisw/aurora/releases/tag/0.16.0-alice2>

Alternatively you can download the sources and build it with:

```bash
./pants binary src/main/python/apache/aurora/kerberos:kaurora
cp dist/kaurora aurora
```

## Getting the authentication token

Once you have the tools in place, you can get authentication token
(similar to alien one, the two will probably need to be unified at
some point) using `cern-get-sso-cookie`. Notice the token has a 1 day
lifetime, after which you need to renew it. The token will also know the
IP address you are using to authenticate, so switching from one network
to the other or changing machine will require a new one.

If you are on Mac you can do:

    cern-get-sso-cookie -r -u https://aliaurora.cern.ch -o ~/.aurora/auth-token --cert ~/Certificates.p12 --password ""

where `Certificates.p12` is your grid certificate, exported from your
KeyChain. If you are on linux, you can either use your grid certificate
in `~/.globus/usercert.pem`:

    cern-get-sso-cookie -r -u https://aliaurora.cern.ch  \
                        -o ~/.aurora/auth-token          \
                        --cert ~/.globus/usercert.pem    \
                        --key ~/.globus/userkey.pem

or install the kerberos client and authenticate with CERN Kerberos realm
via username and password:

    kinit <cern-user-name>@CERN.CH
    cern-get-sso-cookie -r -u https://aliaurora.cern.ch  \
                        -o ~/.aurora/auth-token
                        
Whatever solution you chose, you should end up with an authentication
token in `~/.aurora/auth-token` which you should guard safely as if it was
your own password.

## Configure your Aurora client

In order to reach the Aurora cluster, you need to configure how to
access it. This can be done by creating a file `~/.aurora/clusters.json`:

    [{
      "name": "build",
      "scheduler_uri": "https://aliaurora.cern.ch",
      "auth_mechanism": "KERBEROS",
      "slave_run_directory": "latest",
      "slave_root": "/build/mesos"
    }]

# Test application

## A simple Aurora example

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

## Gotchas

In order to access the logs, you need to connect directly to the client machines, therefore you need to open a SOCKS tunnel inside the cluster and configure your browser to use it.

On Costin machines, for security reasons the log provider are not running, so you need to directly ssh inside them and lock at the filesystem.

