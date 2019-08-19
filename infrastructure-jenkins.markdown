---
title: Creating the Jenkins Master
layout: main
categories: infrastructure
---

In order to drive the Continuos Integration process the ALICE build infrastructure uses Jenkins. 

Jenkin master belong to the  `alibuild/jenkins` hostgroup in CERN puppet, while slaves
are in general dynamically provisioned using Apache Mesos using the Jenkins
[mesos-plugin](https://github.com/jenkinsci/mesos-plugin). The main advantage
of this setup is that the plugin can spawn slaves inside a docker container so
that it can run, for example SLC5 on CC7 or similar setups.

- [/code/manifests/alibuild/jenkins.pp](https://gitlab.cern.ch/ai/it-puppet-hostgroup-alibuild/blob/master/code/manifests/jenkins.pp)  for the master.

# Essential Operation Guides:

* [Create the Jenkins master](#create-jenkins)
* [Adding a new Mesos cloud](#add-mesos-cloud)
* [Killing a stuck job](#kill-stuck-job)
* [Triggering builds programmatically](#trigger-jobs)
* [Gotchas and issues](#gotchas)

### Create the Jenkins master (only in case of disaster recovery!!!!)
{: #create-jenkins}

The jenkins master is created on top of the usual OpenStack / Foreman
infrastructure at CERN. This is because we want to be able to run Jenkins even
if every other bit of the Mesos Build infrastructure fails.

Creation of the jenkins masters in CERN Foreman setup is described at
<http://cern.ch/config/nodes/createnode.html>. The short recipe to create the machine,
**to do only in case of disaster** is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "ALICE Release Testing")
- To spawn a machine you need to use the `ai-bs` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

      MACHINE_NAME=<alijenkinsXX>

      ai-bs -g alibuild/jenkins                              \
            --cc7                                            \
            --nova-sshkey {{site.builduser}}                 \
            --nova-flavor {{site.openstack_master_flavor}}   \
            --landb-mainuser alice-agile-admin               \
            --landb-responsible alice-agile-admin            \
            $MACHINE_NAME

### Adding a new Mesos cloud
{: #add-mesos}

In order to provision jenkins slaves we use the Jenkins Mesos plugin. The
configuration of such a plugin is found at the bottom of the "Jenkins >
Configuration" page. In order to create a new one:

- Click on "Add Slave Info" you will get a new entry.
- Modify the various entries as it follows:

    - Label String: a mnemonic string with the architecture and the size of the queue,
      e.g. `slc5_x86-64-large`. The size should be either small, medium, large.
    - Jenkins Slave CPUs: 0.1
    - Jenkins Slave Memory: 512 
    - Maximum number of Executor per slave: 1
    - Jenkins Executor CPUs: either 1 for small, 4 for medium, 20 for large.
    - Jenkins Executor Memory in MB: either 2000 for small, 8000 for medium, 40000 for large.
    - Remove FS Root: jenkins
    - Idle termination minutes: 3
    - Mesos Offer Selection Attributes: empty.
    - Additional Jenkins Slave JVM arguments: `-Xms16m -XX:+UseConcMarkSweepGC -Djava.net.preferIPv4Stack=true`
    - Additional Jenkins Slave Agent JNLP arguments: `-noCertificateCheck -noReconnect`

- Click on advanced and modify as follow:

    - Use Docker Containerizer: checked.
    - Docker image: `alisw/<os>-builder` where `<os>` is the operating system
      to use, e.g. `slc5` for Scientific Linux 5. 

- Click on save.

### Killing a stuck job
{: #kill-stuck-job}

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
{: #trigger-jobs}

It's now possible to start new builds in a programmatic way. In order to do so you must
have a valid kerberos token, and you must be able to execute the build from the web GUI.
The step by step guide is:

* Get the token with `kinit`, use `klist` to verify you have one.
* Given the name of the job in the GUI, `<job>`, you can trigger a build with:
      
      curl -k -X POST --negotiate -u : https://alijenkins.cern.ch/kerberos/job/<job>/build --data-urlencode json='{}'

  the urlencoded data is a dictionary with the parameters of the job.
  
## Gotchas and issues:
{: #gotchas}

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
