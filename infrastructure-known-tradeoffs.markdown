---
title: Known tradeoffs / build issues of the build infrastructure
layout: main
categories: infrastructure
---

This is a list of known issues or tradeoffs in our build infrastructure. We document them and try very hard to find a viable solution to all of them, however so far the solution seems to be unaffordable or has even worse drawbacks so we decided to simply live with them when they happen. Any contribution to improve the situation is welcome.

# PR checking

## PR checking dies due to external services (e.g. CCDB) being down

Sometimes checks fail because external services are down. Dealing with them in a proper way would imply mocking the service, but:

* developers often do not know / have time to do proper mocking which is a very tedious process and can often only catch a very limited subset of the various failure modes.
* proper mocking is extremely time consuming because it needs to track the change in behaviour of the services very closely in order not to have the test validate a wrong (compared to production) behaviour.

As a mitigation we run our test continuously, rebuilding broken tests when there is no pending ones.

## PR checks can affect each other, even if unrelated

In order to save time, we check our tests in the same build area, so that we rebuild only changes between one build and another. Due to limitations in CMake or undetected missing dependencies, we can however end up in a state where a given test interferes with another, in particular:

* When libraries / dictionaries are moved around
* When a missing / implicit dependency is present and the order in which PRs are build in the PR checker is by chance a working one.

## PR checks introduce relocation issues a few days after merging

In order to save time, PR checkers do their best to reuse pre-built tarballs which are downloaded from a central server. However by design this requires have packages fully relocatable in particular:

* They should not contain any absolute path
* If they do, the path must contain the package hash, so that the build tool / deployment script can relocate the tarball correctly.
* They must not cache any absolute path associated to their dependencies.

Failing that the net result will be that a relocation issue will be present and will manifest itself 
Rebuilding a PR twice in two different locations is deemed to expensive.
Doing proper sandboxing requires changing the tools we have to something like Bazel.

## Errors appear in the PR checker which are not there local builds

Some of the recipes use environment variables (in particular `ALIBUILD_O2_TESTS`) to trigger different behaviors, e.g. increase the amount of testing being done and enable / disable special features. We should try to minimize their usage, however unfortunately they are still widely used.

## PRs take long to complete all tests

By construction you are limited by the longest path, and even if we try to minimize the amount of work done, one has to ultimately chose between minimizing false negatives and performance. Work is currently being done to reduce the unneeded tests in particular for the analysis. A proper solution for this would be to use a tool which imposes specifying all the hidden dependencies and takes advantage of that. However, this most likely means to move away from CMake and so far it was not considered a viable solution.

# RPM generation

## Updatable RPM packages have conflicting files

Updatable RPMs are generate from the tarballs of the various packages which are also deployed in CVMFS. Those tarballs are built and installed in a separate per-package location, in order to allow multiple, coexisting installations. This means that conflicting files can be introduced without any previous warning at RPM generation time. The alternative, i.e. installing everything in a single location, would either move the problem to it's conjugate for CVMFS installation, or it would mean that what is installed in CVMFS is different from what is packaged in the updateable RPMs, duplicating CI and debugging issues.

## Externals

## Old / own version of externals

Sometimes the externals provided in alidist are either old, or provide a rebuild of a commonly available tool. In general this happens because we need to still support Run 2 Production requirements (including ROOT5 and XRootD3) and we prefer maintain a single set of tools, rather than split our configuration management.

Mitigations for this are the defaults mechanism in aliBuild (including the new "per-architecture defaults") and the requirement selection mechanisms (`--disable`, `prefer_system`, architecture specific requirements) which aliBuild provides.
