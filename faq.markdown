---
layout: page
title: FAQ
categories: main
permalink: /faq/
---

## How do I install the builds done by the automatic system?

If you only need to run your root macro on top of some prebuild
AliPhysics and you do not have access to CVMFS, which remains the
preferred, supported and recommended way of getting ALICE software, you
can try using the prebuild tarballs which you can find at:

<https://ali-ci.cern.ch/repo/TARS/&lt;architecture&gt;/dist-runtime/>

e.g., for CERN Centos 7:

<https://ali-ci.cern.ch/repo/TARS/slc7_x86-64/dist-runtime/>

(other architectures can be found in https://ali-ci.cern.ch/repo/TARS/).

If you have a RPM based distribution and you wish to try it out, please
have a look at the instructions provided here:

<https://dberzano.github.io/alice/install-aliroot/rpms/>

## How do I build AliRoot / O2 using alibuild?

In order to build ALICE packages, you can use the following recipe:

    # Clone our build tool
    git clone https://github.com/alisw/alibuild
    # Clone our build recipes
    git clone https://github.com/alisw/alidist
    # Optional: clone O2 or any other package you want to develop.
    #           aliBuild will prefer your copy to the one specified in the
    #           recipe
    git clone -b dev https://github.com/AliceO2Group/AliceO2 O2
    # Start the build
    alibuild/aliBuild --debug --jobs NJOBS build O2

the software will be compiled, by default, in the `sw` directory. Use

    WORK_DIR=$PWD source sw/<architecture>/O2/latest/etc/profile.d/init.sh

to have the environment setup. For example on OSX:

    WORK_DIR=$PWD source sw/osx_x86-64/O2/latest/etc/profile.d/init.sh

If you want to develop, you can change the sources in the O2 folder and then

    cd sw/<architecture>/BUILD/O2/latest && make -j 20 install

For more details about alibuild you can look at:

    http://alisw.github.io/alibuild
