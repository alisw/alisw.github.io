---
title: Build infrastructure on Nomad
layout: main
categories: infrastructure
---

The CI system uses [Nomad][nomad] for job scheduling, plus [Consul][consul] for job discovery and DNS, and [Vault][vault] to store secrets.

These all have web interfaces. If you have access, you can log in at <https://alinomad.cern.ch>, <https://aliconsul.cern.ch> and <https://alivault.cern.ch> if on the CERN network.

Jobs are defined in [a dedicated git repository][job-decls-repo].

## Table of contents
{:.no_toc}

1. This is a placeholder for the table of contents. Do not delete this line.
{:toc}

# Essential CI operations guide

For hints on how to adapt the instructions in this section to jobs other than the CI job, see the following sections.

## Where to find logs

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

## Stopping and restarting CI jobs

If you want to fully bring down and redeploy a CI job, you must do this manually by stopping and rescheduling it.

Nomad has a "restart" facility for individual allocations, which may suffice, but this only reschedules individual allocs, and not necessarily for long enough for e.g. resource redistribution to happen properly. For instance, if you have one alloc waiting for a spot on a high-memory machine, and something else that doesn't need a lot of memory is taking that spot, then you may have to fully stop the latter alloc's job for the former to be properly scheduled, and afterwards, redeploy the stopped job.

In order to fully stop and redeploy a job, run the following commands (`ci-mesosci-cs8` chosen as an example):

```bash
cd ci-jobs/ci
# First, make sure the running job is identical to the local declaration:
levant render -var-file mesosci-cs8.yaml | nomad job plan -  # no changes should be shown
# Stop all allocations of the running job:
nomad job stop ci-mesosci-cs8
# Now redeploy the job from the current definitions:
levant render -var-file mesosci-cs8.yaml | nomad job plan -  # make sure scheduling is fine
levant render -var-file mesosci-cs8.yaml | nomad job run -   # actually redeploy the job
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

## Scaling a CI job

If you want to permanently change the number of running CI jobs for a specific type of builder (e.g. the `mesosci-cs8` one), change the desired number of builders by setting the value of `num_builders` in `mesosci-cs8.yaml`.

Deploy your scaling update to the cluster by running the usual commands:

```bash
cd ci
$EDITOR mesosci-cs8.yaml                                         # change num_builders value
levant render -var-file mesosci-cs8.yaml | nomad job validate -  # check syntax
levant render -var-file mesosci-cs8.yaml | nomad job plan -      # make sure only your desired changes will be applied
levant render -var-file mesosci-cs8.yaml | nomad job run -       # actually deploy your changes
git add mesosci-cs8.yaml && git commit                           # track your changes
```

If you want to scale a job without committing this change to the Git repository, this can be done like so:

```bash
nomad job scale ci-mesosci-cs8 N    # N is the desired number of builders for this job
```

The above command will automatically install a new `config/workers-pool-size` file into the working area of the running builders without restarting them.

## Deploying changes to the CI job template

When changing the template itself (i.e. the `ci.nomad` file), the changes must be deployed for each YAML file, as each declares a separate instance of the templated job.

```bash
cd ci
$EDITOR ci.nomad
for f in *.yaml; do levant render -var-file "$f.yaml" | nomad job validate -; done
# Now, for each .yaml file in the directory:
levant render -var-file vars.yaml | nomad job plan -      # make sure your desired changes will be applied
levant render -var-file vars.yaml | nomad job run -       # actually deploy your changes
git add ci.nomad && git commit                            # track your changes
```

While there's no harm in running the syntax validation step in a loop, it's probably better to do the actual deployment (i.e. `nomad job run`) manually for each YAML file, so that issues with the deployment can be caught early.

## Troubleshooting placement failures

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

# Developing locally

## Setting up your local environment

You will need to install a reasonably recent versions of Nomad to parse existing job declarations. Additionally, you should install the latest version of Levant; ideally [version 0.3.1 or later][levant-release].

Set the following environment values to connect to ALICE's scheduling cluster:

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
alias levant='NOMAD_TOKEN=$(pass cern/ci/nomad-token | head -1) \levant'
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

[levant-release]: https://releases.hashicorp.com/levant

## Writing job declarations

Jobs are defined using [Levant][levant] templates. While plain nomad templating is powerful, it does not allow variable job identifiers (which are crucial for declaring e.g. multiple similar Jenkins builders or CI workers). Levant allows templating on top of the HCL job specification read by nomad.

As Levant bundles a version of the Nomad client, Levant [`0.3.1` or later][levant-releases] is required in order to parse the HCL job declarations we use.

### Simple job declarations (e.g. `rsync` server)

Simple job declarations, i.e. those that declare only a single job, don't use Levant for templating at all. They are simple HCL files stored in the root directory of the [ci-jobs repository][jobs-decls-repo], named `<job-name>.nomad`.

An example of this is the declaration for the [`rsync` server that serves tarballs for `aliBuild`][rsync-job].

In order to deploy this job, simply run the following commands, checking the output of each:

```bash
nomad job validate repo.nomad   # check syntax
nomad job plan repo.nomad       # check if the job can be scheduled
nomad job run repo.nomad        # actually run the job
```

### Complex, templated job declarations (e.g. CI)

Complicated job declarations are broken up into a common, templated declaration, and multiple YAML "variable files" to declare the variations of the base job to be deployed. These should be collected into a single directory, with multiple `.yaml` files, but only one `.nomad` file per directory.

An example of this is the [CI job][ci]. It contains a `ci.nomad` file, which is the general job declaration. This file is processed first by Levant (in the `levant render` step above). Levant [only processes directives][levant-template] surrounded by double square brackets (`[[ ... ]]`). The result of this first processing step is a [Nomad job declaration][nomad-jobspec] in [HCL syntax][hcl-syntax], which can be passed directly to Nomad.

What values Levant inserts into the file is determined by the YAML file it is given. The structure of these files depends entirely on what values the template expects, but for CI jobs, an example is the [`mesosci-slc7-o2physics.yaml` variable file][o2physics-ci].

In order to run or update this job, run the following commands, replacing `vars.yaml` with the variable file for the instance you want to deploy, and checking the output of each command before running the next one:

```bash
cd ci    # so that levant picks up the correct .nomad file
levant render -var-file vars.yaml | nomad job validate -  # check syntax
levant render -var-file vars.yaml | nomad job plan -      # make sure job can be scheduled
levant render -var-file vars.yaml | nomad job run -       # actually run job
```

[nomad]: https://www.nomadproject.io/
[nomad-jobspec]: https://www.nomadproject.io/docs/job-specification
[hcl-syntax]: https://www.nomadproject.io/docs/job-specification/hcl2
[consul]: https://www.consul.io/
[vault]: https://www.vaultproject.io/
[job-decls-repo]: https://github.com/alisw/ci-jobs
[levant]: https://github.com/hashicorp/levant
[levant-releases]: https://releases.hashicorp.com/levant
[levant-template]: https://github.com/hashicorp/levant/blob/main/docs/templates.md
[rsync-job]: https://github.com/alisw/ci-jobs/blob/master/repo.nomad
[ci]: https://github.com/alisw/ci-jobs/tree/master/ci
[o2physics-ci]: https://github.com/alisw/ci-jobs/blob/master/ci/mesosci-slc7-o2physics.yaml


# Tips and tricks for writing Nomad job declarations

## Using Vault secrets

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
