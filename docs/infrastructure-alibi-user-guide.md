---
title: AliBI Quickstart Guide 
layout: main
categories: infrastructure
---


The **Ali**ce **B**enchmark **I**nfrastructure is a service that was established to provide a platform for systematic, reproducible and consistent benchmarks for ALICE O2.

Its primary task is to provide automated performance regression testing for simulation and reconstruction nightlies, however it is open to developers for their own **benchmarking** tasks within extended CERN working hours.

## Hardware
For the sake of reproducible results, a physical server (compute node) has been acquired:
* 2x Xeon Gold 6132 (2.6GHz, 14Core each)
* 384GB ECC DDR4-2666 RAM
* Nvidia Geforce RTX 2080 (8GB GDDR5)
* AMD Radeon VII (16GB HBM II)

## Software
* latest version of Cern CentOS 7
* Full integration into CERN environment
* Users can exchange data via:
    * AFS
    * EOS
    * CVMFS
* Large SSD scratch space for I/O critical tasks
* Docker 

## Getting Access
Become member of e-group [alibi-users](https://e-groups.cern.ch/)

## Working with the AliBI service
To avoid the interference of multiple users accessing the machine in parallel, users have to request exclusive access to the compute node by asking for an allocation via the [SLURM workload manager](https://slurm.schedmd.com/).

Allocations can be requested in two different work queues:
* Short term interactive shell login directly on the compute node for at most 2h
* Batch scripts (non-interactive) with a runtime of at most 10h.

### Query Work Queues
* Login to `alibi.cern.ch`
* The `sinfo` command will display the available work queues, their resources, time limits and the state 
```bash
-bash-4.2$ sinfo
PARTITION    AVAIL  TIMELIMIT  NODES  STATE NODELIST
interactive*    up    2:00:00      1  alloc alibicompute01.cern.ch
batch           up   10:00:00      1  alloc alibicompute01.cern.ch
```
Note that both queues are processed by a single node: `alibicompute01.cern.ch`. As a consequence an active job from either queue will set the state of both queues to `alloc` and will block until the job is finished.
Find out more about `sinfo` [here](https://slurm.schedmd.com/sinfo.html)

### Query for Jobs
* Login to `alibi.cern.ch`
* The `squeue` command will display all of the request that have been submitted to one of the queues and are either currently running or pending
```bash
-bash-4.2$ squeue 
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
               188     batch submit_s  swenzel  R    1:58:26      1 alibicompute01.cern.ch

```
Find out more about `squeue` [here](https://slurm.schedmd.com/squeue.html) here. There also is the graphical `sview` client which is documented [here](https://slurm.schedmd.com/sview.html).

### Requesting an Interactive Shell Session
Users can request interactive shell sessions. This allows direct login to the compute node via `ssh` as long as the allocation is active. 
* Login to `alibi.cern.ch`
* Ask for interactive session
```bash
salloc --time HH:MM:SS
```
This will request a shell session in the `interactive` queue lasting as long as specified in `--time`. Requests of more than the queue limit of 2h will be rejected. Without the `--time` parameter session length defaults to 15min.

```
-bash-4.2$ salloc 
salloc: Granted job allocation 192
salloc: Waiting for resource configuration
salloc: Nodes alibicompute01.cern.ch are ready for job
allocation on partition interactive granted. Go and login into alibicompute01.cern.ch
JobId=192 JobName=sh
   UserId=milettri(101957) GroupId=def-cg(2766) MCS_label=N/A
   Priority=4294901732 Nice=0 Account=(null) QOS=(null)
   JobState=RUNNING Reason=None Dependency=(null)
   Requeue=0 Restarts=0 BatchFlag=0 Reboot=0 ExitCode=0:0
   RunTime=00:00:03 TimeLimit=00:15:00 TimeMin=N/A
   SubmitTime=2020-02-10T15:04:34 EligibleTime=2020-02-10T15:04:34
   AccrueTime=Unknown
   StartTime=2020-02-10T15:04:34 EndTime=2020-02-10T15:19:34 Deadline=N/A
   SuspendTime=None SecsPreSuspend=0 LastSchedEval=2020-02-10T15:04:34
   Partition=interactive AllocNode:Sid=alibilogin01.cern.ch:27480
   ReqNodeList=(null) ExcNodeList=(null)
   NodeList=alibicompute01.cern.ch
   BatchHost=alibicompute01.cern.ch
   NumNodes=1 NumCPUs=56 NumTasks=1 CPUs/Task=1 ReqB:S:C:T=0:0:*:*
   TRES=cpu=56,node=1,billing=56
   Socks/Node=* NtasksPerN:B:S:C=0:0:*:* CoreSpec=*
   MinCPUsNode=1 MinMemoryNode=0 MinTmpDiskNode=0
   Features=(null) DelayBoot=00:00:00
   OverSubscribe=NO Contiguous=0 Licenses=(null) Network=(null)
   Command=(null)
   WorkDir=/afs/cern.ch/user/m/milettri
   Power=
```

Once your interactive allocation has been granted, you can `ssh` into the compute node and start using it. The session will be automatically closed if time runs out.
```
bash-4.2$ salloc: Job 192 has exceeded its time limit and its allocation has been revoked.
/bin/sh: line 1:  4309 Hangup                  $SHELL
```

Find out more about `salloc` [here](https://slurm.schedmd.com/salloc.html).

### Submitting a Batch Job
TBD

## FAQ

### Which Local Directories Can I Use on The Compute Node?
Fast, local directories are available under
```
$L_HOME=/home/$USER
```
This is your local home directory. Data is kept and users will be asked to clean up this directory if we are about to run out of capacity
```
$SCRATCH=/tmp/$USER
```
This is a temporary directory which will be cleared out regularly without prior announcement

### Can I use SSH keys?
Yes but just as with `lxplus` you will get any Kerberos, AFS and eos token which will cause access issues including no access (neither read nor write) to your home directory

### I Cannot Read/Write to AFS or EOS
You probably logged in with a `ssh` key or your ssh is misconfigured. 
To check the state of your Kerberos cache use `klist`. If it complains about an empty credentials cache:
```bash
-bash-4.2$ klist
klist: No credentials cache found (filename: /tmp/krb5cc_101957)
```

issue the following commands:
* `kinit`: request/renew kerberos token
* `aklog`: request/renew afs token
* `eosfusebind`: sets up eos directories

After this operation your kerberos cache should look like this 
```
-bash-4.2$ klist
Ticket cache: FILE:/tmp/krb5cc_101957
Default principal: milettri@CERN.CH

Valid starting       Expires              Service principal
06.02.2020 10:30:43  07.02.2020 11:30:38  krbtgt/CERN.CH@CERN.CH
	renew until 11.02.2020 10:30:38
06.02.2020 10:30:43  07.02.2020 11:30:38  afs/cern.ch@CERN.CH
	renew until 11.02.2020 10:30:38
```
and you should be able to access eos and afs directories.

