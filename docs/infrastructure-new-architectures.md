---
title: New architectures
layout: main
categories: infrastructure
---
## Adding new Architectures to CVMFS

## Adding new compilers to CVMFS

In order to add a new compiler in CVMFS, one needs to first fix the GCC-Toolchain recipe.
In particular `$ModulesCurrentModulefile` should change to `\$ModulesCurrentModulefile` (FIXME for next time).

Finally, one needs to create a symlink in `/cvmfs/alice.cern.ch/etc/toolchain/modulefiles/<architecture>/GCC-<version>` 
which points to `../../../../../el7-x86_64/Modules/modulefiles/GCC-Toolchain/<version>-<revision>`.

## Enabling a new compiler on Hyperloop

Besides adding the compiler to CVMFS as specified above, one needs to release again jq using the new compiler.
