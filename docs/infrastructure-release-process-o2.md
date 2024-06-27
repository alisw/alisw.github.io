---
title: Release process O2PDPSuite
layout: main
categories: infrastructure
---

# O2PDPSuite release process for EPNs


1. O2PDPSuite is a meta package which contains O2, QualityControl, DataDistribution and all their dependencies.

2. Multiple O2PDPSuite versions can be installed on the EPN in parallel without interfering with each other. The default O2PDPSuite version is taken from CONSUL. The operators have the possibility to overwrite the default version for each partition via the AliECS GUI. This means that multiple partitions can run simultaneously using different software versions.

3. We categorize the O2PDPSuite versions we deploy in the following 3 categories:
    - *Official releases*: fully supported stable versions, to be used as default CONSUL setting, usually updated once per week on Monday in sync with the FLPSuite upgrade.
    - *Test versions* for detectors: installed upon request of detectors / other groups (e.g. EPN) with a newer version of O2. These versions are tested with the same Full System Test as official releases, but they do not undergo the Monday software workout. They are partially supported, particularly it is important for us to get feedback in case of issues so it can be fixed for the next release. But if a problem with such a version cannot be solved easily, the user should go back to the fully supported official release.
    - *Internal test versions*: We do not provide support for other users, these versions are mostly for internal tests. We do not prevent anyone from testing, and please let us know in case there are errors, but support is on a best effort basis only.

4. We will set up a web site where we list all versions currently installed, and what support we provide for each of them.

5. We will have the following software builds:
    - *Nightly test builds*: These are basically the builds we have had so far. They are based on the currently installed FLPSuite on all days except Monday, and on the newest FLPSuite on Monday. In this way, they should be compatible with the software running on the FLP if installed in the morning (on Mondays after the FLPSuite update). The builds use the nightly O2 tag, and several packages (DD, QC, Monitoring, ODC, DDS, O2DPG, and some others on demand) from alidist/master, to have the latest software for tests.
    - *Weekly release build*: Created on Thursday morning, based on the latest available FLPSuite, using the O2 tag of that night, and Monitoring, DDS, ODC, O2DPG from alidist master of that morning.
    - *Manual builds on demand*, based either on a nightly test build, or on the weekly release build, or a specified alidist branch, with specific packages manually overridden to a specific version.

6. Release coordination process:
    - The weekly build we create on Thursday is the candidate we want to run as default from the next Monday on.
    - We will install it on Thursday on the EPNs, and announce the changes to the previous build at the release coordination meeting on Thursday.
    - The build is for the next FLPSuite, so it may be that it is not compatible with the currently running FLPSuite.
    - It can be tested using the FLP Staging running the new FLPSuite.
    - In case the new FLPSuite is not yet available during build time, we build against the old FLP Suite. Then, a manual build against the new FLPSuite must be done before Monday morning.
    - If the software workout on Monday succeeds, this will become the new default version in CONSUL.
    - If RC wants multiple release builds for Monday (e.g. with 2 different DataDistribution versions), they can request these and we will create the corresponding manual builds in time.
    - In addition, we will always deploy the latest nightly software build on Monday morning as alternative version.

7. Build names:
    - The weekly release and manual release builds will follow the following naming scheme, to make it clear which DD version they contain, and what FLPSuite they are based on: `O2PDPSuite-epn-%Y%m%d-DD[VERSION]-flp-suite-v[VERSION]`.
    - Daily test builds will have just the date, and in case of 2 builds in addition the time, as before.

8. Updating default software version in CONSUL:
    - Regularly, on Monday morning the default in CONSUL will be changed to the new release version.
    - Any test version installed by PDP (regardless whether requested by a detector or for an internal test) will not be set in CONSUL, but must be enabled per partition in the AliECS GUI.
    - If RC wants to update to a new version, i.e. make a test version the default version under the week, they can call the PDP oncall for the change, or for the revert if needed, or just change the CONSUL values by themselves.

9. Software tests
    - All PRs to O2 are tested in a small full system test with a TF consisting of 5 collisions.
    - When deployed to the EPN farm, all O2 versions are tested with a 15 minute full system test replaying full PbPb TFs at 50kHz interaction rate on a single node, running the full synchronous processing workflow.
    - QC and calibration tasks are gradually added to the full system test.
