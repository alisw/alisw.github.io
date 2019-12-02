# ALICE Benchmarking Infrastrure (AliBI)

The main goal of AliBI is to provide a service for benchmarking CPU and GPU code in a consistent, reproducable environment. This allows monitoring of performance of the code base over time, especially for simulation and reconstruction tasks.

The system is conceived to run two types of jobs:
* Nightly performance regression tests for simulation and reconstruction tasks
* Interactive and batch access for registerd users during CERN extended working hours.

To ensure that results are always reproducable, the machine setup is enforced and controlled using Puppet and access to the compute resources is serialized using the SLURM cluster manager. This means that users do not login onto the nodes directly where the computation is being carried out. Instead, users log in into a _head node_ and ask for access to compute resources. If they can be granted, an interactive bash session on a _compute node_ is opened. In this session the following guarantees can be made:
* The user is the only active user on the underlying hardware eliminating system load that might have otherwise been caused by other users. 
* The system state corresponds to the one described in the systems initial puppet manifest. This ensures that no processes or containers from previous users are still running on the hardware as well as a consistent software stack.

# Installing the AliBI system

The AliBI system relies on a CERN OpenStack VM for the _head node_ (`alibilogin01.cern.ch`) and a bare metal server as _compute node_ (`alibicompute01.cern.ch`). The software stack and machine state is formalized using puppet manifests and is fully integrated in the CERN configuration management ecosystem. The setup process is fully described below.

## AliBI head node

* On `aiadm.cern.ch` enter the OpenStack _Release Testing_ environment by running
```bash
eval $(ai-rc "{{site.experiment}} Release Testing")
```
* Spawn the head node by
```bash
ai-bs -g alibuild/alibi/login --foreman-environment alibuild_alibi --cc7 --nova-sshkey alibuild --nova-flavor m2.xlarge --landb-mainuser alice-agile-admin --landb-responsible alice-agile-admin alibilogin01
```
* add alias to `alibi`:
```bash
openstack server set --property landb-alias=alibi alibilogin01
```

## AliBI compute node

The compute node is a physical machine outside the CERN datacenter, which makes provisioning a bit more complicated.

### Registrations (only for first time set up)
* Register the machine in CERN [LANDB](https://network.cern.ch)
* Create an entry for the machine in [Foreman](https://judy.cern.ch/):
  * In Foreman, select `Hosts>New Host`. 
  * In _Host_ enter:
    * Name: `alibicompute01`
    * Host Group: `alibuild/alibi/compute`
    * Environment: `production`
    * Rest is blank
  * Puppet classes: blank
  * Interfaces: en1 - crosscheck with LANDB!
    * Type: `Interface`
    * Mac address: `08:f1:ea:f0:1f:3c`
    * Device identifier: `eno1`
    * DNS name: `alibicompute01`
    * Domain: `cern.ch`
    * Subnet: `CERN GPN 2 (188.184.0.0/15)`
    * IP address: `188.184.2.54`
    * Managed: `YES`
    * Primary: `YES`
    * Provision: `YES`
  * Operating System:
    * Architecture: `x86_64`
    * Operating System: `CentOS 7.7`
    * Media: `CentOS mirror`
    * Root password required.
  * Parameters: no changes.
  * Additional Information:
    * Owned by: `alice-agile-admin`
    * Enabled: `YES`
    * Hardware Model: `ProLiant DL380 Gen10`

### Prepare installation
* Based on the Foreman entry, a provisioning template in form of a _kickstart file_ is generated and is updated every time the configuration in Foreman is changed.
* Since the compute node is outside of the CERN datacenter it does not have direct access to this file, so it needs to be downloaded and self hosted for the duration of the installation.
  * Download kikstart file from `Templates>provision Template>Review`
  * Inside the file:
    * Set install to `graphical`
    * Remove content about:
      *  bootloader (mbr)
      *  partitioning
  * Host the file on a webserver. The simplest way is to use python2.7 `python -m SimpleHTTPServer` in the directory where your kickstart file is located.
* Stage certificate on `aiadm.cern.ch`:
```
certmgr-stage --host alibicompute01.cern.ch
```
* Set Foreman environment to `alibuild/alibi`.

### Installation
* Get IPMI/ILO access to the physical server
* Boot machine in network boot (PXE)
* Select OS and press E to edit.
* Modify the `linuxefi ...` line by deleting everything that comes after `ip=dhcp`and append the location of your kickstart script such that it reads
```bash
linuxefi (http)/aims/boot/<YOUR_CC_VERSION_HERE>/vmlinuz ip=dhcp ks=http://<PATH_TO_KICKSTART_FILE>
```
* `CTRL+x` to start
* A graphical installer will start.
    * All options but the partitioning are already preset, but can be changed manually.
    * Perform partitioning:

| Mountpoint | Space (GB)  |
| ---------- | ----------: |
| /boot      | 1.0         |
| /boot/efi  | 0.2         |
| /          | 500.0       |
| /docker    | 1000.0      |
| /home      | 7000.0      |
| /tmp       | 14000.0     |

* Start the installation and let it finish. The machine will restart automatically.
* At this point you will notice that the `post installation` section of the installation has not been completed automatically. Since all commands are bash, it can be executed dully by copy& paste or extracted and executed as a separate script.
* Afterwards the machine state should reflect the puppet manifests and can be fully monitored using the CERN Foreman infastruture.
