---
title: CVMFS releases
layout: main
categories: infrastructure
---

ALICE software is compiled and published on CVMFS, for use on the Grid, Hyperloop and LEGO trains.

## Adding a new architecture

If you want to start publishing software for a new architecture (e.g. `el10-x86_64`), you need to do the following.

Note that the architecture is named differently on CVMFS than in `aliBuild`: an `aliBuild`
architecture `slc<N>_x86-64` is published as `el<N>-x86_64`.
The mapping is declared per architecture in the publisher configuration (see below).

You can edit the contents of CVMFS by SSH'ing into cvmfs-alice.cern.ch, then running `sudo -iu cvalice` and starting a CVMFS transaction using `cvmfs_server transaction alice.cern.ch`.

1. Make `/cvmfs/alice.cern.ch/bin/alienv` handle the new architecture.
   The usual procedure is to send a pull request to change the [copy in ali-bot](https://github.com/alisw/ali-bot/blob/master/cvmfs/alienv).
   This will run some tests on the changed script.
   Once merged into ali-bot, then deploy manually by copying the file to its path in CVMFS.
2. Add the new architecture to [aliPublish.conf](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish.conf),
   under `architectures:`, with its `CVMFS:` name (`el<N>-x86_64`).
   You will at least need to publish the `GCC-Toolchain` and `grid-base-packages` packages, in addition to any physics packages you want to make available (like `O2PDPSuite`).

   Note that `grid-base-packages` must be published explicitly: nothing depends on it at runtime, so
   it is never pulled in automatically, yet the `GCC-Toolchain` modulefile requires it, so nothing
   is loadable on the new architecture until it is there.

   The publisher is invoked once per configuration file, so while bringing an architecture up it is
   often more convenient to give it a configuration file of its own, publishing a single package to
   a single architecture (see
   [`aliPublish-cvmfs-slc10.conf`](https://github.com/alisw/ali-bot/blob/master/publish/aliPublish-cvmfs-slc10.conf)
   for an example). That way the resulting directory layout, relocation and modulefiles can be
   inspected, and publishing can be stopped again, without touching the production configuration;
   the architecture is folded into `aliPublish.conf` once it works.
3. Create a `BASE/1.0` modulefile for the new architecture at `/cvmfs/alice.cern.ch/<arch>/Modules/modulefiles/BASE/1.0`.
   The easiest way to do this is to copy the modulefile from an existing architecture and adjust it as necessary.
4. Create symlinks for GCC builds under `/cvmfs/alice.cern.ch/etc/toolchain/modulefiles/<arch>/Toolchain/`.
   For this, you should run the publisher manually once (or wait for it to run via cron), so that GCC builds are available under `/cvmfs/alice.cern.ch/<arch>/Modules/modulefiles/GCC-Toolchain/`.
   For each desired GCC version, add a symlink `/cvmfs/alice.cern.ch/etc/toolchain/modulefiles/<arch>/Toolchain/GCC-vX.Y.Z` pointing to `../../../../../<arch>/Modules/modulefiles/GCC-Toolchain/vX.Y.Z-N`.
5. Create a `default` symlink for the `grid-base-packages` modulefile under `/cvmfs/alice.cern.ch/<arch>/Modules/modulefiles/grid-base-packages`.
   If you've just built `grid-base-packages` for the first time for the new architecture, it'll be called `v1-1`, so run `ln -s v1-1 /cvmfs/alice.cern.ch/<arch>/Modules/modulefiles/grid-base-packages/default`.
   
When you're done, commit your changes by running `cd` (so that your shell isn't keeping a CVMFS directory open), then running `cvmfs_server publish alice.cern.ch`.

If you want to abort your changes, run `cvmfs_server abort` and confirm by typing `y` at the prompt instead.

Remember to re-enable the publisher in the crontab, if you disabled it earlier.
