---
title: Creating the build infrastructure
layout: page
categories: infrastructure
---

**WORK IN PROGRESS -- do not use**

# Cluster architecture description:

The {{site.experiment}} build infrastructure consists of two kinds of nodes, masters,
responsible for scheduling jobs, and slaves, responsible to execute jobs and
services. They are in general provisioned using [CERN Openstack
Infrastructure](http://openstack.cern.ch) and configured using [CERN Puppet /
Foreman setup](http://cern.ch/config).

Masters belong to the hostgroup `{{site.master_hostgroup}}` while slaves
belong to `{{site.slave_hostgroup}}`. The configuration of those hostgroups
can be found in the GIT repository
<https://git.cern.ch/web/it-puppet-hostgroup-{{site.builduser}}.git>, in particular in:

- [/code/manifests/{{site.master_hostgroup}}.pp](https://git.cern.ch/web/it-puppet-hostgroup-{{site.builduser}}.git/blob/HEAD:/code/manifests/{{site.master_hostgroup}}.pp)  for the master.
- [/code/manifests/{{site.slave_hostgroup}}.pp](https://git.cern.ch/web/it-puppet-hostgroup-{{site.builduser}}.git/blob/HEAD:/code/manifests/{{site.slave_hostgroup}}.pp)  for the slaves.


We have in particular three masters, each running on a separate OpenStack
availability zone which work in an High Availability (HA) mode which allows
the ensamble to continue working correctly and scheduling jobs. In particular
the masters run the following services:

- The [**Mesos Master**](http://mesos.apache.org) service: Mesos is used to
  schedule some of the Jenkins jobs automatically on the cluster and to automate
  deployment of some of the services, in particular using the Marathon setup.

- The [**ZooKeeper**](https://zookeeper.apache.org): the backend which keeps
  track of Mesos distributed state, actually providing the HA setup.

- The [**Marathon**](https://mesosphere.github.io/marathon/) service: a simple
  Platform as a Service (PaaS) implemented as a Mesos framework which allows
to define, launch and monitor long running services on the slaves. It relays
on Mesos to do the resource management.

# Useful recipes:

### Setting up the OpenStack environment:

First of all make sure you have all the rights to create machines in OpenStack
and to administer them via Puppet. In particular you'll have to have rights
for the "{{site.openstack_project}}" OpenStack project. In order to use openstack
you need to go to CERN OpenStack administration machines: `aiadm.cern.ch` and 
obtain the OpenStack credentials by doing:

    eval $({{site.experiment Release Testing}})

You can now execute the various OpenStack commands, using the CLI tool called
`openstack`, while an exhaustive list of all the available options can be
optained via `openstack help -h`, for the process of spawning new machines you probably
only care about:

- `openstack server list`: list the machines in the {{site.openstack_project}} 
- `openstack image list`: list of OS images you can use. In
  particular the build nodes should use the latest `{{site.openstack_image}}`
  ones.
- `openstack flavor list`: list available flavors of virtual machines (i.e. how
  many CPUs, RAM).

Before you can continue to create a slave, make also sure you import the SSH
key required by build machines into your openstack configuration (use the
"Access & Security" tab and use "Import key") and that you call it `{{site.builduser}}`.

### Creating a master

Creation of masters in CERN Foreman setup is described at
<http://cern.ch/config/nodes/createnode.html>. The short recipe for build
machine is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")
- To spawn a machine you need to use the `ai-bs-vm` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

      MACHINE_NAME=<{{site.exp_prefix}}mesosXX>
      ZONE=cern-geneva-<X> # <X> needs to be in {a,b,c}
                           # make sure you use different zones
                           # to improve availability.

      ai-bs-vm -g {{site.master_hostgroup}} \
               -i "{{site.openstack_image}}" \
               --nova-sshkey {{site.builduser}} \
               --nova-availabilityzone $ZONE \
               --nova-flavor {{site.openstack_master_flavor}} \
               --landb-mainuser alice-agile-admin \
               --landb-responsible alice-agile-admin \
               $MACHINE_NAME

### Creating a slave  

Creation of slaves in CERN Foreman setup is described at
<http://cern.ch/config/nodes/createnode.html>. The short recipe for build
machine is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")

- Specify a few parameters for the machine you want to spawn:

      MACHINE_NAME=<{{site.builduser}}XX>

- Create a data volume which will be used as build scratch space and
  for build artifacts:

      openstack volume create --type io1 --size 1000 $MACHINE_NAME-store

- To spawn a machine you need to use the `ai-bs-vm` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

      ai-bs-vm -g {{site.slave_hostgroup}} \
               -i "{{site.openstack_image}}" \
               --nova-sshkey {{site.builduser}} \
               --nova-flavor {{site.openstack_flavor}}\
               --landb-mainuser alice-agile-admin \
               --landb-responsible alice-agile-admin \
               --nova-attach-new-volume vdc=1TB \
               $MACHINE_NAME

This will spawn a new machine. You can check the boot status either in the
OpenStack GUI or via `openstack server list`. The `{{site.builduser}}` key used is the ssh
key available from the {{site.builduser}} user AFS account. Of course you
should change the name of the machine (`<{{site.builduser}}XX>` in the
example) and use a current image and flavor. If you have issues about the ssh
key, make sure you imported it in your account (see the Setting up the
OpenStack environment) part.

### Deleting a slave

Similarly the documentation to delete a slave is found at:

<http://configdocs.web.cern.ch/configdocs/nodes/deletenode.html>

the recipe for destoying slaves is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")
- Delete the machine with `ai-kill-vm <{{site.builduser}}XX>`
- Delete the previously attached volumes.

### Rebooting a slave

In case there is an issue with any of the slaves, a hard reboot can be
attempted to bring it back to a working state. This can be done via the
OpenStack GUI, in the Instances tab, or by issuing:

      eval $(ai-rc "{{site.experiment}} Release Testing")
      openstack server reboot --hard <slave name>

in case the GUI is not functional.
