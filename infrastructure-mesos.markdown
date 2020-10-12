---
title: Creating the build infrastructure
layout: main
categories: infrastructure
---

# Cluster architecture description

The ALICE build infrastructure consists of two kinds of nodes: masters,
responsible for scheduling jobs, and agents, responsible for executing jobs and
services. They are in general provisioned using [CERN Openstack
Infrastructure](http://openstack.cern.ch) and configured using the [CERN Puppet /
Foreman setup](http://cern.ch/config).

Masters belong to the Puppet hostgroup `alibuild/mesos/master` while agents
belong to `alibuild/mesos/slave`. The configuration of those hostgroups
can be found in the `master` branch of the
[it-puppet-hostgroup-alibuild](https://gitlab.cern.ch/ai/it-puppet-hostgroup-alibuild)
git repository, in particular in:

- [/code/manifests/mesos/master.pp](https://gitlab.cern.ch/ai/it-puppet-hostgroup-alibuild/-/blob/master/code/manifests/mesos/master.pp) for the master.
- [/code/manifests/mesos/slave.pp](https://gitlab.cern.ch/ai/it-puppet-hostgroup-alibuild/-/blob/master/code/manifests/mesos/slave.pp) for the slaves.

Notice that in order to be able to modify such configuration and create / deploy machines in those hostgroups you will have to be part of the `alice-puppet` hostgroup.

We have in particular three masters, each running on a separate OpenStack
availability zone which work in an High Availability (HA) mode which allows
the ensemble to continue working correctly and scheduling jobs even in the eventuality that one
of the machines goes down. A diagram for the services running on the masters can be find below:

![build-infrastructure-diagram](images/build-infrastructure-diagram.png)

The masters run the following services:

- The [**Mesos Master**](http://mesos.apache.org) service: Mesos is used to
  schedule some of the Jenkins jobs automatically on the cluster and to automate
  deployment of some of the services, in particular using the Marathon setup.

- The [**ZooKeeper**](https://zookeeper.apache.org): the backend which keeps
  track of Mesos distributed state, actually providing the HA setup.

- The [**Marathon**](https://mesosphere.github.io/marathon/) service: a simple
  Platform as a Service (PaaS) implemented as a Mesos framework which allows
  to define, launch and monitor long running services on the slaves. It relies
  on Mesos to do the resource management.

# Essential Operation Guides

* [Getting access to the OpenStack / Puppet infrastructure](#setup)
* [Creating a master](#create-master)
* [Rebuild a master](#rebuild-master)
* [Creating an agent](#create-agent)
* [Reboot an agent](#reboot-agent)
* [Delete an agent](#delete-agent)

## Getting access to the OpenStack / Puppet infrastructure
{: #setup}

First of all make sure you have all the rights to create machines in OpenStack and to administer them via Puppet. 

To get the OpenStack access rights, you should ask to become member of the `alice-vm-admin` egroup. To get the Puppet rights, you should ask to become member of the `alice-agile-admin` egroup. This can be done using the usual [egroups interface](https://egroups.cern.ch).

Once you have those rights to use OpenStack you need to ssh into the CERN
OpenStack administration machines (`aiadm.cern.ch`) and obtain the correct
OpenStack credentials by doing:

```bash
eval $(ai-rc "{{site.openstack_project}}")
```

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

Creation of masters in CERN Foreman setup is described in the [Configuration
Management User Guide](https://configdocs.web.cern.ch/nodes/create/index.html).
The short recipe for a build machine is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

  ```bash
  eval $(ai-rc "{{site.experiment}} Release Testing")
  ```
- To spawn a machine you need to use the `ai-bs` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

  ```bash
  MACHINE_NAME=<{{site.exp_prefix}}mesosXX>
  ZONE=cern-geneva-<X> # <X> needs to be in {a,b,c}
                       # make sure you use different zones
                       # to improve availability.

  ai-bs -g {{site.master_hostgroup}}                   \
        --foreman-environment alibuild_devel           \
        --{{site.openstack_image | downcase}}          \
        --nova-sshkey {{site.builduser}}               \
        --nova-availabilityzone $ZONE                  \
        --nova-flavor {{site.openstack_master_flavor}} \
        --landb-mainuser alice-agile-admin             \
        --landb-responsible alice-agile-admin          \
        --nova-attach-new-volume vdb=200GB             \
        $MACHINE_NAME
  ```

## Creating a agent
{: #create-agent}

Creation of mesos agents in CERN Foreman setup is described in the
[Configuration Management User Guide](https://configdocs.web.cern.ch/nodes/create/index.html).
The short recipe for a build machine is:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing (for build machines):

  ```bash
  eval $(ai-rc "ALICE Release Testing")
  ```

  or (for the release validation machines):
  
  ```bash
  eval $(ai-rc "ALICE Cloud Tests")
  ```

- In order to create new machines, you will also need the public SSH key to be
  added on the `alibuild` machines. Execute (assuming you have saved the public
  SSH key in `alibuild.pub`):

  ```bash
  openstack keypair create --public-key alibuild.pub alibuild
  ```

- Specify a few parameters for the machine you want to spawn:

  ```bash
  MACHINE_NAME=<alibuildXX>
  ```

- To spawn a machine you need to use the `ai-bs` wrapper, which will take
  care of provisioning the machine and putting it in Foreman, so that it will
  receive from it the Puppet configuration:

  ```bash
  ai-bs -g alibuild/mesos/slave                     \
        --foreman-environment production            \
        --cc7                                       \
        --nova-sshkey alibuild                      \
        --nova-flavor m2.2xlarge                    \
        --landb-mainuser alice-agile-admin          \
        --landb-responsible alice-agile-admin       \
        --nova-attach-new-volume vdb=500GB:type=io1 \
        $MACHINE_NAME
  ```

  or for release validation machines, the same command but without the
  `--nova-attach-new-volume vdb=500GB:type=io1` option:

  ```bash
  ai-bs -g alibuild/mesos/slave                     \
        --foreman-environment production            \
        --cc7                                       \
        --nova-sshkey alibuild                      \
        --nova-flavor m2.2xlarge                    \
        --landb-mainuser alice-agile-admin          \
        --landb-responsible alice-agile-admin       \
        $MACHINE_NAME
  ```

This will spawn a new machine. You can check the boot status either in the
OpenStack GUI or via `openstack server list`. Of course you should change the name
of the machine (`<alibuildXX>` in the example).
- In order to make sure that the machine is correctly up and running, you should:
  - ping it
  - ssh to it
  - run `puppet agent -t -v` until no errors are reported
  - execute `docker pull alisw/slc7-builder` to force pull the builder image.

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

  ```bash
  eval $(ai-rc "{{site.experiment}} Release Testing")
  ```

- Actually rebuild the machine

  ```bash
  ai-rebuild-vm --cc7 {{site.exp_prefix}}mesosXX
  ```

It can take up to one hour for the process to complete.

- In order to make sure that the machine is correctly up and running, you should:
  - ping it
  - ssh to it
  - run `puppet agent -t -v` until no errors are reported

## Rebuilding an agent
{: #rebuild-master}

**YOU SHOULD NEVER REBUILD ALIBUILD03 and ALIBUILD09**

Rebuilding an agent is potentially a problem, since the Mesos machine might be doing something,
e.g. building a release, which should not be in general interrupted. Therefore you need to:

* Discuss with your collegueas wether that's a good idea.
* Verify that the machine is not running any particularly important task, by looking at the report
  in the Mesos GUI. If in doubt, ask.

In order to perform the rebuild you need to do:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

  ```bash
  eval $(ai-rc "ALICE Release Testing")
  ```
      
  or 

  ```bash
  eval $(ai-rc "ALICE Cloud Tests")
  ```

  depending on which project the machine belongs to.

- Actually rebuild the machine

  ```bash
  ai-rebuild-vm --cc7 alibuildXX
  ```
      
- In order to make sure that the machine is correctly up and running, you should:
  - ping it
  - ssh to it
  - run `puppet agent -t -v` until no errors are reported. If you keep having errors after a few
    runs, report them.

## Deleting a build infrastructure VM
{: #delete-agent}

**WHATEVER YOU DO, NEVER DELETE ALIBUILD03 OR ITS ATTACHED VOLUME** 

Documentation to delete a VM is found in the [Configuration Management User
Guide](http://configdocs.web.cern.ch/configdocs/nodes/deletenode.html).

The recipe for destoying agents is:

- Ask yourself why you are deleting the VM. **Do it only if you want to get rid of it for good**. 
  If you want to recreate it immediately after, e.g. to handle some 
  irreversible fault on the installation, you might first want to
  try rebuilding it, since it will be much faster. 
- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

  ```bash
  eval $(ai-rc "{{site.experiment}} Release Testing")
  ```
- Delete the VM with `ai-kill <{{site.builduser}}XX>`
- Delete the previously attached volumes.

## Rebooting a Mesos server
{: #reboot-server}

In case there is an issue with any of the agents, a hard reboot can be
attempted to bring it back to a working state. This can be done via the
OpenStack GUI, in the Instances tab, or doing:

- Login to `aiadm.cern.ch`.
- Set up your OpenStack environment by doing:

  ```bash
  eval $(ai-rc "{{site.experiment}} Release Testing")
  ```
- Actually reboot `<server name>`

  ```bash
  openstack server reboot --hard <server name>
  ```

in case the GUI is not functional.

## Master backups

Backup of the Mesos Master is done via CERN TSM. This needs to be renewed regularly and CERN IT will ping by email about it. Agents are not backed up.
