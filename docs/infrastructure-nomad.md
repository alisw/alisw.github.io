---
title: Build infrastructure on Nomad
layout: main
categories: infrastructure
---

The CI system uses [Nomad][nomad] for job scheduling, plus [Consul][consul] for job discovery and DNS, and [Vault][vault] to store secrets.

These all have web interfaces. If you have access, you can log in at <https://alinomad.cern.ch>, <https://aliconsul.cern.ch> and <https://alivault.cern.ch> if on the CERN network.

Jobs are defined in [a dedicated git repository][job-decls-repo].

### Essential CI operations guide

For hints on how to adapt the instructions in this section to jobs other than the CI job, see the following sections.

#### Where to find logs

Logs are written out to disk by the Nomad agent under an allocation's `alloc/logs/` directory. They can also be fetched or streamed using the Nomad command-line client:

```bash
# Stream logs live, as they are written:
nomad alloc logs -stderr -tail -f '<alloc-uuid>'
# Fetch and print all stored logs for this allocation (warning: lots of text!):
nomad alloc logs -stderr '<alloc-uuid>'
# If a job only has one allocation, you can directly use `-job`:
nomad alloc logs -stderr -tail -f -job repo
```

If you use the last command (the one with `-job`) on a job that has multiple allocations (such as most CI jobs), Nomad will pick a random allocation within the job and give you its logs.

Run `nomad alloc logs -help` for more information on the command and its options.

#### Stopping and restarting CI jobs

If you want to fully bring down and redeploy a CI job, you must do this manually by stopping and rescheduling it.

Nomad has a "restart" facility for individual allocations, which may suffice, but this only reschedules individual allocs, and not necessarily for long enough for e.g. resource redistribution to happen properly. For instance, if you have one alloc waiting for a spot on a high-memory machine, and something else that doesn't need a lot of memory is taking that spot, then you may have to fully stop the latter alloc's job for the former to be properly scheduled, and afterwards, redeploy the stopped job.

In order to fully stop and redeploy a job, run the following commands (`ci-linux-x86` chosen as an example):

```bash
cd ci-jobs
# First, make sure the running job is identical to the local declaration:
nomad job plan ci-linux-x86.nomad   # no changes should be shown
# Stop all allocations of the running job:
nomad job stop ci-linux-x86
# Now redeploy the job from the current definitions:
nomad job plan ci-linux-x86.nomad   # make sure scheduling is fine
nomad job run ci-linux-x86.nomad    # actually redeploy the job
```

Build areas are preserved across allocation restarts, but only if the replacement alloc runs on the same host as its predecessor. If this is not the case, it's not worth transferring the entire build directory over the network, so we just start from scratch instead. This is implemented using the following settings in the CI job definition:

```hcl
group "ci" {
  ephemeral_disk {
    sticky = true
    migrate = false
  }
}
```

#### Scaling a CI job

If you want to permanently change the number of running builders for a specific type of builder (e.g. the `ci-linux-x86` one), change the `count` of its task group in the job declaration.

Deploy your scaling update to the cluster by running the usual commands:

```bash
cd ci-jobs
$EDITOR ci-linux-x86.nomad              # change the group's count
nomad job validate ci-linux-x86.nomad   # check syntax
nomad job plan ci-linux-x86.nomad       # make sure only your desired changes will be applied
nomad job run ci-linux-x86.nomad        # actually deploy your changes
git add ci-linux-x86.nomad && git commit # track your changes
```

If you want to scale a job without committing this change to the Git repository, this can be done like so:

```bash
nomad job scale ci-linux-x86 N    # N is the desired number of builders for this job
```

Builders take their work by claiming it, so they need no knowledge of each other: adding one simply
drains the queue faster, and removing one lets its claims lapse. No further change is needed after
scaling.

#### Deploying a change that affects several CI jobs

There is one CI job declaration per architecture (`ci-linux-x86.nomad`, `ci-linux-arm.nomad`,
`ci-macos-arm.nomad`, and so on), so a change that applies to all of them has to be deployed once
per file.

It's a good idea to validate them all first, but to do the actual deployment (i.e. `nomad job run`)
one file at a time, so that issues with the deployment can be caught early.

```bash
cd ci-jobs
for f in ci-*.nomad; do nomad job validate "$f"; done
# Now, for each file in turn:
nomad job plan ci-linux-x86.nomad       # make sure your desired changes will be applied
nomad job run ci-linux-x86.nomad        # actually deploy your changes
```

#### Troubleshooting placement failures

If the Nomad scheduler fails to place your job, you will see a message like this when you run `nomad job plan`:

```fundamental
Scheduler dry-run:
- WARNING: Failed to place all allocations.
  Task Group "ci" (failed to place 10 allocations):
    * Constraint "${attr.kernel.name} = linux": 3 nodes excluded by filter
    * Constraint "${meta.allow_compilation} = true": 3 nodes excluded by filter
    * Resources exhausted on 39 nodes
    * Dimension "cores" exhausted on 24 nodes
    * Dimension "disk" exhausted on 9 nodes
    * Dimension "cpu" exhausted on 6 nodes
```

In this case, check the following:

1. Check that the cluster actually has space for your job, and that your job declares the correct resource requirements.

   For example, not all Openstack machines have a separate build disk of 500 GiB, so a CI job legitimately cannot be scheduled on these.

2. If your job requires, for instance, lots of memory, but all the high-memory machines are taken by other jobs, you may have to stop and reschedule those other jobs while deploying yours. Assuming that the other jobs don't need a lot of memory, they should be scheduled on other free machines when you restart them after deploying your job.

3. If your job requires a lot of disk space, make sure that Nomad's internal tracking of available disk space is correct. You can do this using the [`nomad-diskfree`][nomad-diskfree] script and looking for any warnings on hosts that don't have any allocations running.

   If you see warnings from `nomad-diskfree`, `ssh` into the affected machine, manually clean up under `/build/nomad/alloc` if necessary, and run `systemctl restart nomad` to reset Nomad's idea of available disk space. If you then run `nomad job plan` again, you should see fewer unplaced allocations.

[nomad-diskfree]: https://github.com/alisw/ali-bot/tree/master/utils/nomad-diskfree

### Developing locally

#### Setting up your local environment

You will need to install a reasonably recent version of Nomad to parse existing job declarations.

Set the following environment values to connect to ALICE's scheduling cluster:

More info on how to get the certificates [here](https://alice-doc.github.io/alice-analysis-tutorial/start/cert.html)

```bash
export NOMAD_ADDR='https://alinomad.cern.ch:443'
export CONSUL_HTTP_ADDR='https://aliconsul.cern.ch:443'
export VAULT_ADDR='https://alivault.cern.ch:443'
# These .pem files must not be password-protected.
# For extra security, make sure they are mode 0600 (owner read-write only).
export {NOMAD,CONSUL,VAULT}_CACERT='<path/to/cern-ca-bundle.crt>'
export {NOMAD,CONSUL,VAULT}_CLIENT_KEY='<path/to/grid-personal-key.pem>'
export {NOMAD,CONSUL,VAULT}_CLIENT_CERT='<path/to/grid-personal-cert.pem>'
# Export your access tokens to the cluster, so that the command-line nomad,
# consul and vault clients can use them:
export NOMAD_TOKEN='<nomad access token UUID>'
export CONSUL_HTTP_TOKEN='<consul access token UUID>'
export VAULT_TOKEN='<vault access token UUID>'
# Alternatively, set these tokens only for commands that need them, e.g.
# by using aliases to retrieve the secrets and invoke the respective command:
alias nomad='NOMAD_TOKEN=$(pass cern/ci/nomad-token | head -1) \nomad'
alias consul='CONSUL_HTTP_TOKEN=$(pass cern/ci/consul-token | head -1) \consul'
alias vault='VAULT_TOKEN=$(pass cern/ci/vault-token | head -1) \vault'
```

However, this only works for machines on the CERN network. If dialling in from outside, you'll have to set up a proxy (e.g. using SSH forwarding) like so, in addition to setting the `*_TOKEN` and certificate-related environment variables as above:

```bash
ssh -L localhost:4646:alimesos01.cern.ch:4646 \
    -L localhost:8500:alimesos01.cern.ch:8500 \
    -L localhost:8200:alimesos01.cern.ch:8200 \
    -N alimesos01.cern.ch &
export NOMAD_ADDR='https://localhost:4646'
export CONSUL_HTTP_ADDR='https://localhost:8500'
export VAULT_ADDR='https://localhost:8200'
export {NOMAD,CONSUL,VAULT}_TLS_SERVER_NAME=alimesos01.cern.ch
```


#### Writing job declarations

Jobs are written as plain [Nomad job declarations][nomad-jobspec] in [HCL2 syntax][hcl-syntax].
We used to template some of them with Levant, in order to declare several jobs whose *identifiers*
differ (something plain Nomad templating cannot express); this is no longer the case, and HCL2
variables are used instead.

##### Single-instance jobs (e.g. `rsync` server)

Most jobs are a single `.nomad` file stored in the root directory of the
[ci-jobs repository][job-decls-repo], named `<job-name>.nomad`. The CI builders are declared this
way, one job per platform family (`ci-linux-x86.nomad`, `ci-linux-arm.nomad`, `ci-macos-arm.nomad`,
...), as is the AliEn publisher. The [RPM publisher](infrastructure-rpms.md) is a single job too,
but it keeps its declaration in a directory of its own.

An example of this is the declaration for the [`rsync` server that serves tarballs for `aliBuild`][rsync-job].

In order to deploy this job, simply run the following commands, checking the output of each:

```bash
nomad job validate repo.nomad   # check syntax
nomad job plan repo.nomad       # check if the job can be scheduled
nomad job run repo.nomad        # actually run the job
```

##### Jobs with several near-identical instances (e.g. Jenkins agents)

Some jobs exist in several near-identical copies that differ only by name and a handful of values.
These live in a directory of their own, with a single `.nomad` file holding the job declaration and
one `<instance>.vars.hcl` file per instance holding that instance's values, declared as
[HCL2 variables][hcl2-variables] in the jobspec.

The [Jenkins agents][jenkins-builder] are the main example.
A job's identifier *is* its HCL block label, which cannot be interpolated, so the jobspec is
rendered to JSON by a small script in the directory that fills in the name and reads the matching
variable file. The rendering is purely local: `nomad job run -output` parses the declaration without
contacting the cluster, so the result can be inspected before anything is deployed.

```bash
cd jenkins-builder
./render.sh slc10-builder-1 > /tmp/jenkins-slc10-builder-1.json
nomad job plan -json /tmp/jenkins-slc10-builder-1.json   # make sure job can be scheduled
nomad job run -json /tmp/jenkins-slc10-builder-1.json    # actually run job
```

[nomad]: https://www.nomadproject.io/
[nomad-jobspec]: https://www.nomadproject.io/docs/job-specification
[hcl-syntax]: https://www.nomadproject.io/docs/job-specification/hcl2
[consul]: https://www.consul.io/
[vault]: https://www.vaultproject.io/
[job-decls-repo]: https://github.com/alisw/ci-jobs
[hcl2-variables]: https://developer.hashicorp.com/nomad/docs/job-specification/hcl2/variables
[rsync-job]: https://github.com/alisw/ci-jobs/blob/master/repo.nomad
[jenkins-builder]: https://github.com/alisw/ci-jobs/tree/master/jenkins-builder


##### Tips and tricks for writing Nomad job declarations

###### Using Vault secrets

If you want to use Vault secrets in your job declaration, you can substitute them inside of templates.

For instance, to assign secrets to environment variables that will be set when the job runs, use a block like the following inside your `task` block:

<!-- We have to escape the opening curly braces in the below code as Jekyll seems to treat them specially. -->

```hcl
template {
  data = <<-EOD
    {{ "{{" }} with secret "kv/data/my-secret-name" }}
    MY_SECRET={{ "{{" }} .Data.data.my_secret | toJSON }}
    MY_OTHER_SECRET={{ "{{" }} .Data.data.my_other_secret | toJSON }}
    {{ "{{" }} end }}
    EOD
  env = true
  destination = "${NOMAD_SECRETS_DIR}/secrets.env"
  perms = "400"
  change_mode = "restart"
}

# Required in order to read vault secrets.
vault {
  policies = ["nomad"]
}
```

This example assumes that you have a secret called `my-secret-name` stored in Vault, whose contents are a JSON object of the form:

```json
{
  "my_secret": "my sup3r s3cr3t",
  "my_other_secret": "my 0th3r s3cr3t"
}
```


## Troubleshooting

### Stuck allocations/jobs

When a job is not able to be deleted and recreated, you can force the deletion using the following commands (requires nomad management token):

```bash
nomad job stop -purge <job-name>
nomad system gc
```


### Nomad error initializing client: tls: failed to parse private key

If you are on macOS you'll need an unencrypted key, you can export it from your certificate with a command like this (note the ``-nodes`` flag)

```bash
openssl pkcs12 -nocerts -in ~/Downloads/myCertificate.p12 -out ~/.globus/userkey.pem -nodes
```

Otherwise you'll probably get the following error:

```
Error initializing client: tls: failed to parse private key
```
