---
title: Creating the Jenkins Master
layout: page
categories: infrastructure
---

# Jenkins installation

In order to drive the Continuos Integration process the `{{site.experiment}}`
build infrastructure uses Jenkins. 

Jenkin master belong to the hostgroup `{{site.jenkins_hostgroup}}` while slaves
are in general dynamically provisioned using Apache Mesos using the Jenkins
[mesos-plugin](https://github.com/jenkinsci/mesos-plugin). The main advantage
of this setup is that the plugin can spawn slaves inside a docker container so
that it can run, for example SLC5 on CC7 or similar setups.

- [/code/manifests/{{site.jenkins_hostgroup}}.pp](https://git.cern.ch/web/it-puppet-hostgroup-{{site.builduser}}.git/blob/HEAD:/code/manifests/{{site.jenkins_hostgroup}}.pp)  for the master.

# Useful recipes:

### Start the Jenkins master

The jenkins master is created on top of the usual OpenStack / Foreman
infrastructure at CERN. This is because we want to be able to run Jenkins even
if every other bit of the Mesos Build infrastructure fails.

Creation of the jenkins masters in CERN Foreman setup is described at
<http://cern.ch/config/nodes/createnode.html>. The short recipe for build
machine is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")
- To spawn a machine you need to use the `ai-bs-vm` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

      MACHINE_NAME=<{{site.exp_prefix}}jenkinsXX>

      ai-bs-vm -g {{site.jenkins_hostgroup}} \
               -i "{{site.openstack_image}}" \
               --nova-sshkey {{site.builduser}} \
               --nova-flavor {{site.openstack_master_flavor}} \
               --landb-mainuser alice-agile-admin \
               --landb-responsible alice-agile-admin \
               $MACHINE_NAME

### Adding a new Mesos cloud

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

