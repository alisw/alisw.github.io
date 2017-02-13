---
layout: page
title: Git advanced
categories: main
permalink: /git-advanced/
---
# ALICE GitHub advanced guide

This guide is meant to be a set of advanced topics regarding the ALICE Github workflow.
If you are looking for a more step by step guide on how to get started and propose code changes to AliPhysics,
please have a look at the [Basic Workflow Guide](/git-tutorial).

# I want to use the command line!

If you find the web based workflow not to your taste, Github offers the possibility to use a command line based workflow via a utility called `hub` .

## Install “hub”

On macOS you can install it with brew:


    brew install hub

On Linux you can [use the precompiled releases](https://github.com/github/hub/releases).

## Forking and cloning the repository

In order to fork a repository you can use `hub fork` and `hub clone` 


    cd ~/alice
    hub fork alisw/AliRoot
    hub clone <your-github-username>/AliRoot

## Creating a pull request
[ ] Fix this…

The second step is asking your changes to be tested and reviewed in order to be included upstream:


    hub pull-request

Note that it is also possible to open a pull request from the GitHub web interface if you want, as you will see later on. The command above allows you to do so without leaving the terminal. The output of the command above will be a URL like:


    https://github.com/alisw/AliRoot/pull/4

If you copy-paste it to your browser you will see a nice interface.

# What happens if my changes need to be approved by more than one person?

Here is an example that involves multiple approvers:

![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_98FF252C26DE06DC5A57C33513260F22ADF7DCDF39C8B349FAD53C27D51C830E_1481552414512_Schermata+2016-12-12+alle+15.19.51.png)

# How to write a pull request

**Please be concise in the title (<50 characters) and explain the details in the description!** A good example is:

- **Title:** Fix missing particles in the decay table
- **Description:** Warning messages about missing particle entries during the ppbench are fixed.

A bad example (too concise!) is:

- **Title:** A fix
- **Description:** <empty>

Another bad example is:

- **Title:** Fix missing particles in the decay table that caused ppbench to complain a lot with warning messages
- **Description:** <empty>


# Work with the AliPhysics OADB (proposal #1)
> Note that this is one possible workflow proposal which assumes having separate AliRootData and AliPhysicsData repositories. It is by no means the final workflow, it’s just an exercise to understand what would be the complexity for the user if we adopted it.


## Use the most recent OADB

If you want to pull changes from upstream you will have to do it both for AliPhysics and AliPhysicsData:


    cd ~alice/AliPhysics/
    git pull
    cd ~alice/AliPhysicsData/
    git pull

You should then `make` both repositories in their respective build directories (assuming you have already performed the first time installation using aliBuild):


    cd ~alice/sw/AliPhysicsData-latest/AliPhysicsData/
    make install
    cd ~alice/sw/AliPhysics-latest/AliPhysics/
    make install


## Push changes to the OADB

If you are one of the users authorized to push changes to the AliPhysics OADB (or any other data file that does not fit in the AliPhysics code repository) you should move to the `AliPhysicsData` directory and work from there:


    cd ~alice/AliPhysicsData/
    # perform your changes
    git commit -a -m 'Updated OADB'
    git push

If you don’t have proper authorization to push changes you should ask the ALICE administrators, who, in accordance with the Physics Coordination, will give you proper access.

The above workflow applies to `AliRootData` too.


# Work with the AliPhysics OADB (proposal #2)
> Note that this is one possible workflow proposal which assumes having AliRoot and AliPhysics using their respective and separated “data” repositories as submodules. It is by no means the final workflow, it’s just an exercise to understand what would be the complexity for the user if we adopted it.


## Use the most recent OADB

If you want to pull changes from upstream do:


    cd ~alice/AliPhysics/
    git pull
    git submodule update --remote

The second command will update the `extras` directory under AliPhysics containing all binary files and/or data files.

You should then `make` AliPhysics in the build directory as usual (assuming you have already performed the first time installation using aliBuild):


    cd ~alice/sw/AliPhysicsData-latest/AliPhysicsData/
    make install


## Push changes to the OADB

If you are one of the users authorized to push changes to the AliPhysics OADB (or any other data file that does not fit in the AliPhysics code repository) you should modify those files under the `extras` directory of the AliPhysics repository.


    cd ~alice/AliPhysics/extra/
    # perform your changes
    git commit -a -m 'Updated OADB'
    git push

If you don’t have proper authorization to push changes you should ask the ALICE administrators that, in accordance with the Physics Coordination, will give you proper access.

This `git push` pushes the changes to the OADB repository but does not tell AliPhysics to use them. For that you need to open a pull request as explained in the +ALICE GitHub basic workflow:


    cd ~alice/AliPhysics
    git commit -a -m 'Updated OADB in AliPhysics'
    git push

The commands above push to your fork on GitHub from which you will open a pull request as usual.

The above workflow applies to `AliRootData` too.


# Work with the AliPhysics OADB (other proposals)
## Do not separate AliPhysics/AliPhysicsData

Unfortunately in order to keep the main repository lean we really need to keep data and binary files separated from the code repository. This is also in preparation of the workflow we adopt in O2. The technical limitation is that we have no guarantee from CERN IT that having many large forks is handled nicely (in fact they recommend *not* to have large repositories at all).


## Use CVMFS
- We don’t have a good way of allowing people to publish (no access control, etc.) - the CVMFS team is working on that but it will not be ready in time.
- We would require having CVMFS installed everywhere. This is currently not the case. This requirement could be harsh for macOS users.
- Currently CVMFS supports versioning, however there is no way to cleanly select the version of the OADB snapshot you want. CVMFS team is working on that too (but it’s not ready now).


## Use FTP/HTTP/etc.
- We *could* use one of those services to host files (and IT can provide us with hosting).
- We don’t however have versioning on that, which is required.


## Git LFS/Git Annex

Those are Git extensions requiring external programs that handle “large” files in the same main repository.

- Require an external tool.
- Complicated to use (annex in particular).
- LFS requires hosting (or paying for it).
- Not always clear what ends in the code repository and what ends elsewhere (we want users to be conscious that data ends in the data place, and code in the code place).
- Breaks the pull requests workflow: you can make a pull request for a *pointer* to a large file, but the large file itself has to be pushed even if it would not be accepted. Pull requests involving large files are not practicable. [See here for more.](https://help.github.com/articles/collaboration-with-git-large-file-storage/)


## Git subtree

It seems to solve most of our problems, and be a better solution with respect to `git submodule`, however the Git version available on lxplus and SLC6 in general (1.7.1) does not support them. We might consider to use them if we can mitigate 

# Backlog
- As a user I want to know how to migrate from my current setup to the new one.
- As a user I would like to have instructions on how to create a pull request for one commit out of two on a branch. Value 90.

