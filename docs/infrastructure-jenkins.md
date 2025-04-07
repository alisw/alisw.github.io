---
title: Creating the Jenkins Master
layout: main
categories: infrastructure
---

In order to drive the Continuous Integration process the ALICE build infrastructure uses Jenkins. 

The Jenkins master is started by Nomad, on a machine which belongs to the `alibuild/mesos/slave/jenkins` hostgroup in CERN puppet, while slaves are declared using [Nomad](infrastructure-nomad.md) under `ci-jobs/jenkins-builder/`.
The main advantage of this setup is that we can run slaves inside a Docker container, so that it can run e.g. CentOS 7 builds on Alma 9 hosts.

Master nodes are configured through Puppet in the file:

- [/code/manifests/alibuild/mesos/slave/jenkins.pp](https://gitlab.cern.ch/ai/it-puppet-hostgroup-alibuild/blob/master/code/manifests/mesos/slave/jenkins.pp)

## Essential Operation Guides:

* [Create the Jenkins](#create-the-jenkins-master-only-in-case-of-disaster-recovery)
* [Starting Jenkins](#starting-jenkins)
* [Killing a stuck job](#killing-a-stuck-job)
* [Triggering builds programmatically](#triggering-builds-programmatically)
* [Gotchas and issues](#gotchas-and-issues)

### Create the Jenkins master (only in case of disaster recovery!!!!)

The jenkins master is created on top of the usual OpenStack / Foreman
infrastructure at CERN. This is because we want to be able to run Jenkins even
if every other bit of the build infrastructure fails.

Creation of the jenkins masters in CERN Foreman setup is described at
<http://cern.ch/config/nodes/createnode.html>. The short recipe to create the machine,
**to do only in case of disaster** is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:
  ```bash
  eval $(ai-rc "ALICE Release Testing")
  ```
- To spawn a machine you need to use the `ai-bs` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:
  ```bash
  MACHINE_NAME=<alijenkinsXX>

  ai-bs -g alibuild/mesos/slave/jenkins       \
        --alma9                               \
        --nova-sshkey alibuild                \
        --nova-flavor m2.large                \
        --landb-mainuser alice-agile-admin    \
        --landb-responsible alice-agile-admin \
        $MACHINE_NAME
  ```

### Starting Jenkins

While the machine on which Jenkins is run is provisioned by OpenStack, the actual Jenkins instance is managed by Nomad.
The instance can be started with:

```bash
cd ci-jobs
nomad job plan jenkins-master.nomad  # sanity check
nomad job run jenkins-master.nomad   # actually deploy the job
```

You can then look at the logs in the Nomad GUI.

### Killing a stuck job

Sometimes Jenkins jobs (especially pipelines) remain stuck in some weird state
and refuse to be killed by the GUI. When this happens, the last resort is to do the following:

* Go to "Manage Jenkins"
* Go to "Script Console"
* Adapt the following script to your needs:

      def jobId = XYZ
      def jobName = "daily-builds/daily-aliphysics-test"
      def job = Jenkins.instance.getItemByFullName(jobName)
      def task = job.getBuildByNumber(jobId)
      task.doKill()

### Triggering builds programmatically

It's now possible to start new builds in a programmatic way. In order to do so you must
have a valid kerberos token, and you must be able to execute the build from the web GUI.
The step by step guide is:

* Get the token with `kinit`, use `klist` to verify you have one.
* Given the name of the job in the GUI, `<job>`, you can trigger a build with:
      
      curl -k -X POST --negotiate -u : https://alijenkins.cern.ch/kerberos/job/<job>/build --data-urlencode json='{}'

* The parametrized job can be started with:

      curl -X POST -k --negotiate -u : https://alijenkins.cern.ch/kerberos/job/<job>/buildWithParameters -H 'Content-Type: application/x-www-form-urlencoded' -d '<parameters>'

  The `<parameters>` are formatted as in a URL: `<name>=<value>&<name2>=<value2>`.

### Creating Jenkins agents with guaranteed resources

This is the main way we deploy Jenkins builders.
The advantage of fixed builders is that we are never in a situation where there is not enough space on the cluster by accident to run a Jenkins build.

You can create dedicated Jenkins builders using Nomad.

[Similar to the setup for CI builders](infrastructure-nomad.md#complex-templated-job-declarations-eg-ci),
Jenkins builders are defined as YAML files under `ci-jobs/jenkins-builder/`.

They are normally configured using three keys:

- `name`: a user-visible name, to keep builders apart. It's a good idea to mention the OS that the builder is running, e.g. `slc9-builder-1`.
  Make this the same as the file name, sans the `.yaml` extension.
- `docker_image`: the image to run the builder inside of.
  Builds will run for this platform.
  For example: `registry.cern.ch/alisw/slc9-builder:latest`.
- `is_light`: a boolean; if true, enough resources are allocated to actually run a compilation.
  "Light" builders are useful to have for non-compilation tasks; for instance, some Jenkins builds wait for approved PRs to be merged before starting the build; this only takes up a slot on a "light" builder to avoid wasting resources.

Jenkins builders also need to be declared to Jenkins.
To do this, go to <https://alijenkins.cern.ch/computer> > "New Node" and create a new node.
Assuming the architecture is `<arch>`, the node needs to be called (e.g. `<arch>-builder-<X>`) where `<X>` is incremental number.
Valid `<arch>` values are e.g. `slc7`, `slc8`, `ubuntu2004`, etc. These mirror `aliBuild`'s idea of architectures.

Click on the newly create agent and note down the associated secret.
This needs to be stored in [the `jenkins-builder` Vault secret](https://alivault.cern.ch/ui/vault/secrets/kv/kv/jenkins-builder/details).
Click "create a new version", then add a new key at the bottom, named `<name>_node_secret` (where `<name>` is the one you gave it in the YAML file earlier).
For the value, paste in the secret hexadecimal token you copied from Jenkins earlier.
Save the updated secret.

Finally, you can create the new agent with:

```bash
cd ci-jobs/jenkins-builder/
levant render -var-file <name>.yaml | nomad job validate -  # check syntax
levant render -var-file <name>.yaml | nomad job plan -      # make sure job can be scheduled
levant render -var-file <name>.yaml | nomad job run -       # actually run job
```

### Gotchas and issues:

* On some systems, the CERN CA is not available by default. You can overcome this by either:
  * Go to <https://ca.cern.ch> and install all the required CA certificates. In general this is what is needed on macOS.
  * Installing the `CERN-CA-certs` package.

* On some systems, kerberos gives a token for the actual backend name, rather than `aliaurora`. You can check that by doing klist and you will see `HTTP/alibuild-frontend01.cern.ch@CERN.CH`:

```bash
Credentials cache: API:B7FC3DD4-738F-417E-B2FA-92B2CCA9590C
        Principal: eulisse@CERN.CH

  Issued                Expires               Principal
Aug 14 15:50:42 2019  Aug 15 01:50:42 2019  krbtgt/CERN.CH@CERN.CH
Aug 14 15:50:46 2019  Aug 15 01:50:42 2019  HTTP/alibuild-frontend01.cern.ch@CERN.CH
```

In order to fix this you will have to change your kerberos configuration, usually found in `/etc/krb5.conf`, and add `rdns = false` in the `[libdefaults]` stanza.
