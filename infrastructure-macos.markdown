---
title: CI builders on macOS
layout: main
categories: infrastructure
---

The macOS build infrastructure works the same way as its Linux counterpart -- it uses Nomad.

This guide covers:
* [Installation and initial setup of the machine](#installation_and_initial_setup)
* [Adding a CI checker](#adding_ci_checker)

# Installation and initial setup
{: #installation_and_initial_setup}

These are the instructions for macOS Ventura.

* During setup, create a user account for the `alibuild` user, do not sign in to iCloud.
* Sign into the App Store as `ali.bot@cern.ch`.

## Set up firewall exceptions

Add the new Mac's hostname to the list of non-Puppetized CI hosts in Puppet.
You have to do this in two places, once each in the header of:

* `it-puppet-hostgroup-alibuild/code/manifests/mesos/master.pp`
* `it-puppet-hostgroup-alibuild/code/manifests/mesos/slave.pp`

## Software prerequisites

* Install the right version of XCode, either from the App Store or from [xcodereleases.com](https://xcodereleases.com/).
  Usually this should be the latest one from the App Store, but occasionally we don't support the latest one yet.
  In that case, use the latest supported one.
* Open XCode in order to install the command line tools and accept the license, or run:
  ```bash
  sudo xcode-select --install
  sudo xcodebuild -license
  ```
* Install Homebrew using the [instructions on their webpage](https://brew.sh/):
  ```bash
  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
  echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
  ```
  ...and reload the session or `source ~/.zprofile`.
* Check if everything is OK via:
  ```bash
  brew doctor
  ```
* Install the software prerequisites:
  ```bash
  brew install alisw/system-deps/alice-build-machine
  ```

## Create a build volume

Open "Disk Utility".
Select "Macintosh HD", then in the window title bar, press "+" above "Volume".
Call the new volume `build` with format "APFS".

## Create the work environment

With recent MacOS versions, `/` is not writable.
We install under `/opt/build` instead:
```bash
sudo ln -s ../Volumes/build /opt/build
sudo mkdir /opt/build/alice-ci-workdir
sudo chown alibuild:staff /opt/build/alice-ci-workdir
```

Afterwards exclude the `/opt/build` directory from Spotlight in the system preferences.
In System Preferences, go to Siri & Spotlight -> Spotlight Privacy (at the bottom of the page).
In the "Privacy" tab, hit the "+" button.
Now select the `build` volume and confirm.

## Copy Grid CA certificates from another host

O2 unit tests need this to connect to services like the test CCDB instance.

On your local computer, run:
```bash
ssh alibuildmac00.cern.ch tar -cz /etc/grid-security/certificates | ssh <new hostname> 'cat > certs.tar.gz'
```

Then, on the Mac being set up, run:
```bash
sudo mkdir -p /etc/grid-security/certificates
sudo tar -xzf ~/certs.tar.gz --strip-components=3 -C /etc/grid-security/certificates
```

## Get a Grid host certificate

O2 unit tests need this to connect to services like the test CCDB instance.

At <https://ca.cern.ch/ca/>, request a new Grid host certificate for the machine.
Make sure that reminders about expiring certs go to the responsible group, not just you.
In order to do this, you must be listed as the "responsible" or "main user" of the machine in LanDB.

Set up a temporary passphrase for the generated certificate before downloading it.
After downloading it, copy it to the Mac being set up.

Assuming the certificate you downloaded was called `<short-hostname>.p12` (which is the default), run the following on the Mac:
<!-- https://ca.cern.ch/ca/Help/?kbid=024100 -->
```bash
sudo openssl pkcs12 -in "$(hostname -s).p12" -clcerts -nokeys -out /etc/grid-security/hostcert.pem
sudo openssl pkcs12 -in "$(hostname -s).p12" -nocerts -nodes -out /etc/grid-security/hostkey.pem
sudo chmod 0600 /etc/grid-security/hostkey.pem
sudo chown alibuild:staff /etc/grid-security/*.pem
```

## Ask for AliEn access

Ask Costin or Max to add the DN of the certificate you just installed to the list of allowed certs for the `alienci` user.
You can get the certificate's DN using:
```bash
openssl x509 -in /etc/grid-security/hostcert.pem -noout -text | grep Subject:
```

## Set up Nomad and Consul

```bash
sudo mkdir -p "$(brew --prefix)/etc"/{nomad,consul}.d
```

Then, copy `nomad.hcl` and `consul.hcl` into these directories from another CI Mac.
Fix the permissions, since the files contain secrets:
```bash
sudo chown root:admin "$(brew --prefix)/etc"/{nomad,consul}.d/*.hcl
sudo chmod 600 "$(brew --prefix)/etc"/{nomad,consul}.d/*.hcl
```

Now adapt the files to the local host:

* In `consul.hcl`, change `advertise_addr` to the host's external IP address.
* In `nomad.hcl`, change `bind_addr` to the host's external IP address.
* In `nomad.hcl`, change `cpu_total_compute`, since Nomad doesn't know how to compute this itself for M1/M2 machines.
  To compute the right number, run `sudo powermetrics -s cpu_power -n 1` and add up the maximum frequency (in MHz) for each listed CPU.

You also need to set up custom launchd services for Nomad and Consul, since the ones packaged by Homebrew are not suitable for production use.
On your local machine, run the following to copy the files into place:
```bash
scp -3 alibuildmac00.cern.ch:{homebrew.mxcl.{nomad,consul}.plist,restart-services.sh} <new hostname>:.
```

Finally, start the services by running the following command on the Mac:
```bash
sudo ~/restart-services.sh
```

The host should appear in [the list of Nomad clients](https://alinomad.cern.ch/ui/clients).
If it does not, check the log, e.g. using `less -RSM "$(brew --prefix)/var/log/nomad.log"`.
If Nomad complains about not being able to connect to the master nodes at `alimesosNN.cern.ch`, update their firewall rules using:
```bash
pdsh -w 'alimesos[01-03].cern.ch' puppet agent -tv
```

# Adding a CI checker
{: #adding_ci_checker}

## Adding a Nomad job

The Macs are configured on a host-by-host basis, unlike the Linux checkers, so that we can more tightly control what checks run where.
This saves precious disks space, since many Macs lack this resource compared to the Linux machines.

In the private `ci-jobs` repository, configure a new CI job variation by creating a file called `ci-jobs/ci/<new hostname>.yaml` with the following content (substituting the parts in `<>`):
```yaml
---
role: macos
arch: <new hostname>
config_suffix: ''
num_builders: 1
# Specify the total resources available on the host, so that we can reserve the
# entire thing. This makes nomad's statistics more accurate.
resources:
  cpu: <the number you configured earlier as cpu_total_compute>
  memory: <total MB of memory available on the host>
```

Once that's done, actually run the CI job on the new host:

```bash
cd ci-jobs/ci
levant render -var-file <new hostname>.yaml | nomad job plan -
# If the above succeeds, then:
levant render -var-file <new hostname>.yaml | nomad job run -
```

## Configuring individual checks

Mac CI checkers are configured like their Linux equivalents, using `.env` files under `ali-bot/ci/repo-config/`.
The Macs specifically are listed under [`ali-bot/ci/repo-config/macos/`](https://github.com/alisw/ali-bot/tree/master/ci/repo-config/macos).

Each host has a directory there named after its short hostname; it will run checks for the `.env` files inside its directory.

If checks are not picked up, make sure the hostname matches what the Mac thinks it is.
If in doubt, run `hostname -s` to check.
