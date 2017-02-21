---
layout: page
title: GitHub advanced workflow
categories: main
permalink: /git-advanced
---

This guide is meant to be a set of advanced topics regarding the ALICE Github
workflow. If you are looking for a more step-by-step guide on how to get
started and propose code changes to AliPhysics, please start from the [basic
GitHub workflow](/git-tutorial).


## I want to use the command line!

If you find the web based workflow not matching your taste, GitHub offers the
possibility to use a command-line based workflow as well via a utility called
`hub` .


### Install hub

On macOS you can install it with brew:

```bash
brew install hub
```

On Linux you can [use the precompiled
releases](https://github.com/github/hub/releases\). After you have installed
hub, the `hub` command will be available. Note that the [hub
documentation](https://hub.github.com/) suggests you configure `git` as an
alias to `hub` in your shell configuration:

```bash
alias git=hub
```

This way you can “extend” the `git` command functionalities with the ones
offered by `hub` seamlessly. We will not assume the alias is configured in this
document.

### Forking and cloning the repository

In order to fork a repository you can use `hub fork` and `hub clone`. The
following example clones the official AliPhysics repository and configures your
own fork (the `user.github` Git configuration parameter must be set as
suggested in the [basic workflow documentation](/git-tutorial)). Note that your
fork is created on your GitHub account in case it does not exist:

```bash
cd ~/alice
hub clone alisw/AliPhysics
hub fork
```

### Creating a pull request

The second step to request your changes to be tested and reviewed in order to
be included upstream. This is done by typing:

```bash
hub pull-request
```

The command above allows you to do so without leaving the terminal. The output
of the command above will be a URL like:

```bash
https://github.com/alisw/AliRoot/pull/4
```

If you copy-paste it to your browser you will see the GitHub interface for the
opened pull request, which is the same as if you had opened it via the web
interface.


## How to write a pull request

Try to be short (stay within 50 characters) and concise in the title, and save
any elaboration for the description.

A good example is:

- **Title:** Fix missing particles in the decay table
- **Description:** Warning messages about missing particle entries during the
  ppbench are fixed.

A bad example is:

- **Title:** A fix
- **Description:** <empty>

You should elaborate what the fix is for at least.

Another bad example is:

- **Title:** Fix missing particles in the decay table that caused ppbench to
  complain a lot with warning messages
- **Description:** <empty>

The title is too long, and most of what it says can be moved to the description
field.


## How to use large data files for analysis

Some large OADB files were removed from AliPhysics and they are no longer
available in the Git repository. Those files are not lost; they have been moved
to the following EOS location, which is accessible from lxplus:

```
/eos/experiment/alice/analysis-data
```

We have preserved the same directory structure found on AliPhysics, and the
same permissions: for instance, `PWGLF` is writable by all members of
`alice-svn-pwglf` (“svn” is a remain from the past and not a reference to the
legacy version control system).

Every day, in concomitance with the AliPhysics daily tag (at 4pm Geneva time),
this folder is snapshotted on CVMFS under the following path:

```
/cvmfs/alice.cern.ch/data/analysis/YYYY/vAN-YYYYMMDD
```

carrying the same name as the corresponding AliPhysics tag. CVMFS has the
advantage to make data access from Grid jobs reliable and faster due to caches
(files unchanged in two different snapshots are not downloaded twice).

In order to profit from the separate storage for large files we have created an
interface in AliRoot to allow transparent access to OADB files using a relative
path. For instance, if you want to access the following large OADB data file:

```
PWGLF/FORWARD/CORRECTIONS/data/fmd_corrections.root
```

you can do:

```cpp
TFile::Open(AliDataFile::GetFileNameOADB("PWGLF/FORWARD/CORRECTIONS/data/fmd_corrections.root"))
```

The static function `AliDataFile::GetFileNameOADB` returns the first accessible
full URL of the OADB file by finding the first match from the following ordered
paths:

1. `$OADB_PATH/<file>`
2. `$ALICE_DATA/OADB/<file>`
3. `$ALICE_PHYSICS/OADB/<file>`
4. `/cvmfs/alice.cern.ch/data/analysis/YYYY/vAN-YYYYMMDD/OADB/<file>` (for Grid jobs)
5. `root://eospublic.cern.ch//eos/experiment/alice/analysis-data/OADB/<file>`

This means that for laptop analysis it will always be possible to access data
files, somehow, and in a transparent fashion. If you want to have your OADB
data locally, you can download it from lxplus:

```bash
export OADB_PATH=/path/to/my/local/oadb
rsync -av --delete cern_user@lxplus.cern.ch:/eos/experiment/alice/analysis-data/ $OADB_PATH/
```

> Trailing slashes are important to rsync! Do not forget them!

Note that the variable `$OADB_PATH` must be exported to the environment where
you run your local analysis in order to make it visible to the job.


### Non-OADB data files

The same EOS path has also PWG-specific directories, outside the OADB one, for
other analysis-specific data. The following interface can be used to access
files from there:

```cpp
TFile::Open(AliDataFile::GetFileName("PWGMM/my_large_data.root"))
```

> Note the difference between `GetFileName()` and `GetFileNameOADB()`.

In this case, the file will be searched in the following locations in order:

1. `$ALICE_DATA/<file>`
2. `$ALICE_PHYSICS/<file>`
3. `/cvmfs/alice.cern.ch/data/analysis/YYYY/vAN-YYYYMMDD/<file>` (for Grid jobs)
4. `root://eospublic.cern.ch//eos/experiment/alice/analysis-data/<file>`
