---
title: Using Apache Aurora
layout: page
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
amins.

## Getting the client

In order to work with CERN SSO Authentication, you will need to use a
slightly modified client for both aurora and cern-get-sso-cookie.

You can build them with [aliBuild]():

```bash
aliBuild build aurora
aliBuild build cern-get-sso-cookie/latest
alienv enter aurora/latest cern-get-sso-cookie/latest
```

## Getting the authentication token

Once you have the tools in place, you can get authentication token
(similar to alien one, the two will probably need to be unified at
some point) using `cern-get-sso-cookie`. Notice the token has a 1 day
lifetime, after which you need to renew it. The token will also know the
IP address you are using to authenticate, so switching from one network
to the other or changing machine will require a new one.

If you are on Mac you can do:

    cern-get-sso-cookie -r -u https://aliaurora.cern.ch -o ~/.aurora-token --cert ~/Certificates.p12 --password ""

where `Certificates.p12` is your grid certificate, exported from your
KeyChain. If you are on linux, you can either use your grid certificate
in `~/.globus/usercert.pem`:

    cern-get-sso-cookie -r -u https://aliaurora.cern.ch  \
                        -o ~/.aurora-token               \
                        --cert ~/.globus/usercert.pem    \
                        --key ~/.globus/userkey.pem

or install the kerberos client and authenticate with CERN Kerberos realm
via username and password:

    kinit <cern-user-name>@CERN.CH
    cern-get-sso-cookie -r -u https://aliaurora.cern.ch  \
                        -o ~/.aurora/auth-token
                        
Whatever solution you chose, you should end up with an authentication
token in `~/.aurora-token` which you should guard safely as if it was
your own password.

## Configure your Aurora client

In order to reach the Aurora cluster, you need to configure how to
access it. This can be done by creating a file `~/.aurora/config.json`:

    [{
      "name": "build",
      "scheduler_uri": "https://aliaurora.cern.ch",
      "auth_mechanism": "COOKIE",
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
