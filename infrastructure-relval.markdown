---
title: Release validation operations
layout: main
categories: infrastructure
---

# Managing data on EOS

EOS and its XRootD interface are used by the release validation:

 - for storing reference data
 - for storing validation results

In order to manipulate EOS data you need to have a valid X.509 proxy. If you do
not have EOS installed you can use it from the `alisw/slc6-cvmfs` container
which has it preinstalled.

Assuming your proxy to be used for EOS is available under `~/.globus/eos-proxy`
you can start the container like the following:

    docker run -it --rm                                                   \
      -privileged                                                         \
      -v ~/.globus:/root/.globus:ro                                       \
      -e X509_USER_PROXY=/root/.globus/eos-proxy                          \
      -e X509_CERT_DIR=/cvmfs/grid.cern.ch/etc/grid-security/certificates \
      alisw/slc6-cvmfs                                                    \
      parrot_run eos root://eospublic.cern.ch/

Where `parrot_run` is used to enable CVMFS (only used for reading the CA
certificates). This command sends you straight to the EOS prompt. Note that the
`-privileged` option is required by Parrot.

If you have EOS installed on your machine you can simply export a couple of
environment variables and start EOS afterwards:

    export X509_USER_PROXY=<path_to_eos_proxy>
    export X509_CERT_DIR=/cvmfs/grid.cern.ch/etc/grid-security/certificates
    export EOS_MGM_URL=root://eospublic.cern.ch
    eos


## Remove old release validations

From the EOS prompt on `eospublic.cern.ch` move to the results output folder:

    cd /eos/opstest/pbuncic/output/

You can then list the directories with `ls` and remove them one by one with:

    rm -r


## Used quota on EOS

From the EOS prompt on `eospublic.cern.ch` query the quota for our directory:

    quota /eos/opstest/pbuncic/


## Stage new data on EOS from AliEn

This operation requires a working AliEn installation, we will be assuming that
you are using AliEn from aliBuild. Enter the AliEn environment:

    alienv enter AliEn-Runtime/latest

Now get a valid token and proxy:

    alien-token-init <your_alien_username>

Now get and use the staging data script to stage new data:

    git clone https://github.com/alisw/release-validation
    cd release-validation/
    ./stage-data.sh /alice/data/2016/LHC16e/000252858/raw/ 300

where:

 - the first parameter is the AliEn path (without the `alien://` prefix) where
   raw data can be found
 - the second parameter is the number of files to stage, which will be picked
   randomly among the full raw list

The script will exit on the first error. You can re-run it many times until it
converges to the full list being staged on EOS. Data already copied is skipped.

The script will echo the path of a `.done` file containing the EOS paths to be
used as a dataset for a release validation. In order to make this file usable:

 - add it as a `.txt` file in the `datasets/` directory of the
   `release-validation` repository, and push it
 - edit the Jenkins job in order to add the dataset name (file name without the
   `.txt` extension) to the drop-down list associated to the `DATASETS` Jenkins
   variable


# Proxy certificate

The release validation uses a Grid proxy certificate mapped to the **alibot**
EOS/CERN service account on eospublic.cern.ch.

As CERN does not allow service accounts to own certificates, we are using a host
certificate for this purpose. The corresponding node is managed by the CERN
Puppet infrastructure therefore its certificate will be automatically renewed.
Certificate and key will be found under `/etc/grid-security` on the very host.

When the proxy expires or is about to, it has to be renewed using the
`grid-proxy-init` utility found on lxplus, which is always up-to-date. To do so,

    grid-proxy-init -cert hostcert.pem \
                    -key hostkey.pem   \
                    -out eos-proxy     \
                    -valid 9000:0

Adjust the validity to make it _shorter_ than the host certificate (the utility
will warn you in case it is longer). The validity is specified in
`hours:minutes`.

To verify whether this proxy can be correctly parsed, use the following command:

    voms-proxy-info -file eos-proxy

The new proxy can now be deployed to production.
