---
title: Creating the Jenkins Master
layout: page
categories: infrastructure
---

**WORK IN PROGRESS -- do not use**

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
- Set up your OpenStack environment (once) and source the
  `~/private/{{site.builduser}}-openrc.sh` file, entering the password when
  prompted.
- To spawn a machine you need to use the `ai-bs-vm` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

      MACHINE_NAME=<{{site.exp_prefix}}jenkinsXX>

      ai-bs-vm -g {{site.jenkins_hostgroup}} \
               -i "{{site.openstack_image}}" \
               --nova-sshkey {{site.builduser}} \
               --nova-flavor {{site.openstack_master_flavor}} \
               $MACHINE_NAME
