---
layout: page
title: Git tutorial
categories: main
permalink: /git-tutorial/
---

# ALICE GitHub basic workflow

## What are these instructions for?

These instructions are meant to be the simplest way to get you up and running developing one feature at the time for AliPhysics on GitHub. They can be reused for AliRoot by simply changing AliPhysics to AliRoot. If you are trying to do something more complicated or you want to have more detailed information on the inner workings of the system, please refer to the [advanced workflow](/git-advanced).

## Setup GitHub account

The first thing you need to do is to create an account there and to map your CERN account to a GitHub one. This is a one time operation. 

You can do this by going to:


> [https://alisw.cern.ch/alice-github/login](https://alisw.cern.ch/alice-github/login)

In case you don’t have a GitHub account it will ask you to create one. Notice there is no need for a paying account: a free GitHub account is sufficient to work on ALICE software.


> **Note:** we recommend you create a GitHub username which is the same as your CERN one, if available.
![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_DC2E677430BA2AA7FD207F2D16426909084AA99BBE80F321C6194E44ED5772CA_1481104790690_C55599B5-D37E-45AF-BE7B-0F8515669BC2.png)


In case you have an account, login and authorize “ALICE Continuous Integration” to read your user name so that it can be mapped to your CERN account.

![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_DC2E677430BA2AA7FD207F2D16426909084AA99BBE80F321C6194E44ED5772CA_1481104817047_7D8A528B-53AE-4855-8F54-9C7DDCD18E7D.png)

## Fork repository

“Forking” a repository in GitHub means that you create a remote copy of some official (*i.e.* *upstream*) repository in your own account. Such a copy is linked to the original one and allows you to propose changes in the form of “Pull Request”.

To fork AliPhysics you can point your browser to:


> https://github.com/alisw/AliPhysics/fork


## Setup user configuration on your local computer

Initial setup:


    git config --global user.name "<firstname> <lastname>"
    git config --global user.email <your-email-address>
    git config --global user.github <your-github-username>

Note that the `--global` flag sets those parameters for every Git repository on your laptop.


> The `user.name` will appear in every commit you make. Note that the old configuration suggested you to use your CERN account name, whereas we now suggest you configure it with your “Firstname Lastname”.

It is probably convenient to setup a [passwordless login for GitHub using SSH](https://help.github.com/articles/generating-an-ssh-key/). If you don’t want to use GitHub with SSH then you can check out how to setup the [credentials helper](https://help.github.com/articles/caching-your-github-password-in-git/) to temporarily or permanently store your password.

## Setup software repositories on your local computer

Please follow the [aliBuild tutorial](http://alisw.github.io/alibuild/tutorial.html). We will assume from now on that:


- your working directory is `~/alice` 
- you have aliBuild installed

If you have your old installation under `~/alice` then we suggest you use a different directory to avoid confusion.


> Note: until the migration to GitHub is effective the `aliBuild init` command from the tutorial should point to the `git-migration`  development branch of our recipes:
> 
> `aliBuild --dist git-migration init AliRoot,AliRootData,AliPhysics,AliPhysicsData` 

Now if you want to contribute to AliPhysics move to the clone directory and tell Git about the existence of your own fork, created in a previous step.


    cd ~/alice/AliPhysics
    git remote add <your-github-username> https://github.com/<your-github-username>/AliPhysics


## The workflow
![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_DC2E677430BA2AA7FD207F2D16426909084AA99BBE80F321C6194E44ED5772CA_1481104848816_38113614-B38C-4B7C-BA39-C6BE8D14B38B.png)


As depicted in the image, the general idea is:

- you always fetch the changes (“pull”) from the upstream repository
- you always write your changes (“push”) to your own fork
- you propose the changes to be added upstream by opening a “pull request”
- your code change proposal is checked by a series of automatic tests before being accepted (see the details on this point later on)
## Create your first pull request

Let’s say you now have a development to AliPhysics you want to propose for inclusion. This is done with a “pull request”. In order to create one you need to do the following. First push the changes you have to your repository:


    git push <your-github-username> master

At this point changes are on your own “fork” on GitHub, and **they are not yet upstream**. It is important to understand this point as this is the main difference to the old workflow. Create the pull request itself by going to [https://github.com/<your-github-username>/AliPhysics ](https://github.com/alisw/AliPhysics)and clicking on “New Pull Request”:

![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_DC2E677430BA2AA7FD207F2D16426909084AA99BBE80F321C6194E44ED5772CA_1481104883470_48EC02AC-0ACC-464E-A571-998A16C4F43B.png)


Review the proposed changes and if you are satisfied click on the green button saying “Create a pull request”:

![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_DC2E677430BA2AA7FD207F2D16426909084AA99BBE80F321C6194E44ED5772CA_1481104945640_E9ADB103-C71E-49EA-8008-54CA1ABDDF50.png)


Please make sure the title and the description of your pull request are written in a proper English and express the purpose of the changes in a simple and concise way.

A good description will help yourself, the other contributors to ALICE software, and it will guarantee a faster approval (if required).  Notice GitHub is a popular, public, hosting site, so there will be zero tolerance for offending statements or embarrassments to ALICE collaboration. In case of doubt, you can find more information in the [commit guidelines](/doc/7mesQeJRwfZyphJsJ85ER).

![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_98FF252C26DE06DC5A57C33513260F22ADF7DCDF39C8B349FAD53C27D51C830E_1484309825564_Schermata+2017-01-13+alle+13.16.39.png)


Once you are done, your code is **proposed** for inclusion in the official repository, but it’s not yet in, as it needs to pass the (likely automated) review process. Notice also that creating pull requests can be done from the command-line, see the advanced guide for more information.

## Review process

Pull requests are checked for permissions first: directory permissions are the same as before. If you weren’t previously supposed to modify files in a certain directory you still cannot do it now.

There might be three possible scenarios:


- The user is allowed to modify those files.
- The user is not allowed to modify those files and there is a list of “approvers” for the given directory who must approve it first.
- The user is not allowed and no approver is defined.

In the first case, the user is allowed to modify the files and the build system will tell you that by commenting. For example:

![Image](https://d2mxuefqeaa7sj.cloudfront.net/s_98FF252C26DE06DC5A57C33513260F22ADF7DCDF39C8B349FAD53C27D51C830E_1481549204743_Schermata+2016-12-12+alle+14.26.04.png)


Every single pull request is tested for a successful build on our production environment: this means that a pull request in AliPhysics should work with the reference AliRoot version using the same environment used on the Grid.

This test (plus optional additional tests as described in the [ALICE GitHub advanced guide](/git-advanced)) are run automatically: in case they succeed then the pull request is merged automatically without any further action.

If the user is not allowed to modify the code touched by the pull requests, the person(s) responsible will be contacted automatically and will be asked to comment on the pull request and wether or not the change is approved. Special comments are those that begin with:


- `**+1**`** ****: ****approve the change****.** If tests are successful the pull request will be automatically merged.
- `**+test**``**:**`** approve test****ing**** only.** Tests will be started but pull request won’t be automatically merged even if tests are all green.

No pull request is automatically rejected. In case a pull request wants to change the project structure it will be handled by the project administrators in cooperation with Computing or Physics Coordination.


## Amend an open pull request

In case the pull request is not approved or automatic tests do not pass, you should take the necessary measures to amend it. This is done by simply making changes to your local repository, commit them and push them: new commits will be automatically appended to the already opened pull request, and tests will be restarted.

In case you want to retire your pull request by throwing away your changes and resume your work from upstream you need to “reset” the state of your working copy to the upstream repository:


    cd ~alice/AliPhysics
    git fetch --all
    git reset --hard origin/master


> Note that doing so will imply a loss of your current work. You must really be sure that you want to discard it first. If you want to work on multiple features at the same time, and having different pull requests open at once, please refer to the [ALICE GitHub advanced guide](/git-advanced).
