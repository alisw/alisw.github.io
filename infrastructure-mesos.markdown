---
title: Creating the build infrastructure
layout: main
categories: infrastructure
---

# Cluster architecture description:

The {{site.experiment}} build infrastructure consists of two kinds of nodes, masters,
responsible for scheduling jobs, and agents, responsible to execute jobs and
services. They are in general provisioned using [CERN Openstack
Infrastructure](http://openstack.cern.ch) and configured using [CERN Puppet /
Foreman setup](http://cern.ch/config).

Masters belong to the Puppet hostgroup `{{site.master_hostgroup}}` while agents
belong to `{{site.slave_hostgroup}}`. The configuration of those hostgroups
can be found in the GIT repository for [it-puppet-hostgroup-alibuild](
https://gitlab.cern.ch/ai/it-puppet-hostgroup-alibuild), in particular in:

- [/code/manifests/{{site.master_hostgroup}}.pp](https://gitlab.cern.ch/ai/it-puppet-hostgroup-{{site.builduser}}.git/blob/HEAD:/code/manifests/{{site.master_hostgroup}}.pp)  for the master.
- [/code/manifests/{{site.slave_hostgroup}}.pp](https://gitlab.cern.ch/ai/it-puppet-hostgroup-{{site.builduser}}.git/blob/HEAD:/code/manifests/{{site.slave_hostgroup}}.pp)  for the slaves.

We have in particular three masters, each running on a separate OpenStack
availability zone which work in an High Availability (HA) mode which allows
the ensamble to continue working correctly and scheduling jobs even in eventuality that one
of the machines goes down. A diagram for the services running on the masters can be find below:

![build-infrastructure-diagram](images/build-infrastructure-diagram.png)

In particular the masters run the following services:

- The [**Mesos Master**](http://mesos.apache.org) service: Mesos is used to
  schedule some of the Jenkins jobs automatically on the cluster and to automate
  deployment of some of the services, in particular using the Marathon setup.

- The [**ZooKeeper**](https://zookeeper.apache.org): the backend which keeps
  track of Mesos distributed state, actually providing the HA setup.

- The [**Marathon**](https://mesosphere.github.io/marathon/) service: a simple
  Platform as a Service (PaaS) implemented as a Mesos framework which allows
to define, launch and monitor long running services on the slaves. It relays
on Mesos to do the resource management.

# Essential Operation Guides:

* [Getting access to the OpenStack / Puppet infrastructure](#setup)
* [Creating a master](#create-master)
* [Rebuild a master](#rebuild-master)
* [Creating an agent](#create-agent)
* [Reboot an agent](#reboot-agent)
* [Delete an agent](#delete-agent)

## Getting access to the OpenStack / Puppet infrastructure:
{: #setup}

First of all make sure you have all the rights to create machines in OpenStack and to administer them via Puppet. 

To get the OpenStack access rights, you should ask to become member of the `alice-vm-admin` egroup. To get the Puppet rights, you should ask to become member of the `alice-agile-admin` egroup. This can be done using the usual [egroups interface](https://egroups.cern.ch).

Once you have those rights to use OpenStack you need to go to CERN OpenStack administration 
machines: `aiadm.cern.ch` and obtain the OpenStack credentials by doing:

    eval $(ai-rc "{{site.openstack_project}}")

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

Further information on how CERN OpenStack cloud works can be found [here](https://clouddocs.web.cern.ch/clouddocs/).

Note that you will have to login as `root` to all the machines.

## Checklist to verify the status of a master

In case there are issues with one of the masters you should follow the following checklist:

* Check on the [Openstack Dashboard](https://openstack.cern.ch) if the machine is up and running.
* Check in [Foreman](https://foreman.cern.ch) if there are any puppet errors.
* Ping the machine.
* SSH into the machine.
* Check if docker is running and if it has at least the following containers: `mesos-master`, `zookeeper`, `marathon`, `aurora-scheduler`, `mesos-dns`.

## Creating a master
{: #create-master}

Creation of masters in CERN Foreman setup is described at
<http://cern.ch/config/nodes/createnode.html>. The short recipe for build
machine is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")
- To spawn a machine you need to use the `ai-bs` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

      MACHINE_NAME=<{{site.exp_prefix}}mesosXX>
      ZONE=cern-geneva-<X> # <X> needs to be in {a,b,c}
                           # make sure you use different zones
                           # to improve availability.

      ai-bs -g {{site.master_hostgroup}}                      \
            --foreman-environment alibuild_devel        \
            --{{site.openstack_image | downcase}}                 \
            --nova-sshkey {{site.builduser}}                  \
            --nova-availabilityzone $ZONE               \
            --nova-flavor {{site.openstack_master_flavor}} \
            --landb-mainuser alice-agile-admin          \
            --landb-responsible alice-agile-admin       \
            --nova-attach-new-volume vdb=200GB          \
            $MACHINE_NAME

## Creating a agent  
{: #create-agent}

Creation of mesos agents in CERN Foreman setup is described at
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

      ai-bs -g {{site.slave_hostgroup}}               \
            --foreman-environment alibuild_devel      \
            --{{site.openstack_image | downcase}}     \
            --nova-sshkey {{site.builduser}}          \
            --nova-flavor {{site.openstack_flavor}}   \
            --landb-mainuser alice-agile-admin        \
            --landb-responsible alice-agile-admin     \
            --nova-attach-new-volume vdc=1TB:type=io1 \
            $MACHINE_NAME

This will spawn a new machine. You can check the boot status either in the
OpenStack GUI or via `openstack server list`. The `{{site.builduser}}` key used is the ssh
key available from the {{site.builduser}} user AFS account. Of course you
should change the name of the machine (`<{{site.builduser}}XX>` in the
example) and use a current image and flavor. If you have issues about the ssh
key, make sure you imported it in your account (see the Setting up the
OpenStack environment) part.

## Rebuilding a master
{: #rebuild-master}

Rebuilding a master is a potentially disruptive operation, since our mesos setup requires at least 2
masters to be up and running in order to schedule new jobs. Therefore before you actually decide to
rebuild one you should:

* Discuss with your collegueas wether that's a good idea.
* Make sure that the other two masters are properly functioning.
* If the master is the currently leading master, force a leadership transition to one of the other
  two machines before the rebuild (Optional, since failower will take care of that).  
  
In order to perform the rebuild you need to do:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")

- Actually rebuild the machine

      ai-rebuild-vm --cc7 {{site.exp_prefix}}mesosXX

it can take up to one hour for the process to complete.
- In order to make sure that the machine is correctly up and running, you should:
     - ping it
     - ssh to it
     - run `puppet agent -t -v` until no errors are reported

## Deleting build infrastructure VM
{: #delete-agent}

**WHATEVER YOU DO, NEVER DELETE ALIBUILD03 OR ITS ATTACHED VOLUME** 

Documentation to delete a VM is found at:

<http://configdocs.web.cern.ch/configdocs/nodes/deletenode.html>

the recipe for destoying agents is:

- Ask yourself why you are deleting the VM. **Do it only if you want to get rid of it for good**. 
  If you want to recreate it immediately after, e.g. to handle some 
  irreversible fault on the installation, you might first want to
  try rebuilding it, since it will be much faster. 
- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")
- Delete the VM with `ai-kill <{{site.builduser}}XX>`
- Delete the previously attached volumes.

## Rebooting a Mesos server
{: #reboot-server}

In case there is an issue with any of the agents, a hard reboot can be
attempted to bring it back to a working state. This can be done via the
OpenStack GUI, in the Instances tab, or doing:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

      eval $(ai-rc "{{site.experiment}} Release Testing")
- Actually reboot `<server name>`

      openstack server reboot --hard <server name>

in case the GUI is not functional.
