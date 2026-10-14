---
title: CI builders on macOS
layout: main
categories: infrastructure
---

The macOS build infrastructure works the same way as its Linux counterpart -- it uses Nomad.

This guide covers:

* [Installation and initial setup of the machine](#installation-and-initial-setup)
* [Adding a CI checker](#adding-a-ci-checker)

## Installation and initial setup

These are the instructions for macOS Ventura.

* During setup, create a user account for the `alibuild` user, do not sign in to iCloud.
* Sign into the App Store as `ali.bot@cern.ch`.

Some commands below should be run on your local machine.
These may refer to the new Mac's hostname, so set a variable for this now, or substitute `$newhost` manually in the commands below.
```bash
newhost='<short hostname of the new Mac, e.g. alibuildmac09>'
```

### Set up firewall exceptions

Add the new Mac's hostname to the list of non-Puppetized CI hosts in Puppet.
You have to do this in two places, once each in the header of:

* `it-puppet-hostgroup-alibuild/code/manifests/mesos/master.pp`
* `it-puppet-hostgroup-alibuild/code/manifests/mesos/slave.pp`

### Set up the Mac's network connection

On the Mac, go to System Settings -> Network and disable WiFi.

Then, in the network settings, go to Ethernet -> Details -> DNS, and click "+" under "DNS Servers" to add the IPv4 addresses of alimesos01, alimesos02 and alimesos03 as DNS servers.
Remove other DNS servers (including the automatic CERN central ones).

You will have to register them in LanDB too, the network will redirect you to the right page automatically.

### Set up the Mac's hostname

Make sure the Mac knows itself by it's real hostname, as it can have issues
with DNS resolution otherwise. You can do this by going to System Preferences
-> General -> About, and changing the computer name to the short hostname of
the Mac (e.g. `alibuildmac09`).

### Enable SSH and VNC

To start both the SSH and VNC servers, enable remote login in System Preferences -> General -> Sharing -> Remote Login, and Screen Sharing.

Additionally, you may want to add the proper SSH keys to the `alibuild` user's `authorized_keys` file.

### Prevent the Mac from going to sleep

By default, Macs will go into a low-power state after a while without interactive use, which interrupts the CI build process.
To prevent this, change the following settings in System Settings:

In Energy, enable "Start up automatically after a power failure", "Wake for network access", and "Prevent computer from sleeping automatically when the display is off".

In Lock Screen, set both "Start Screen Saver when inactive" and "Turn display off when inactive" to "Never".

### Disable crash reporting

Some jobs are intended to crash, and we don't want the Mac to get stuck waiting for user input.

```bash
launchctl unload -w /System/Library/LaunchAgents/com.apple.ReportCrash.plist
sudo launchctl unload -w /System/Library/LaunchDaemons/com.apple.ReportCrash.Root.plist
```

### Software prerequisites

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

### Create a build volume

Open "Disk Utility".
Select "Macintosh HD", then in the window title bar, press "+" above "Volume".
Call the new volume `build` with format "APFS".

### Create the work environment

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

### Copy Grid CA certificates from another host

O2 unit tests need this to connect to services like the test CCDB instance.

On your local computer, run:
```bash
ssh alibuildmac00.cern.ch tar -cz /etc/grid-security/certificates | ssh "$newhost" 'cat > certs.tar.gz'
```

Then, on the Mac being set up, run:
```bash
sudo mkdir -p /etc/grid-security/certificates
sudo tar -xzf ~/certs.tar.gz --strip-components=3 -C /etc/grid-security/certificates
```

### Get a Grid host certificate

O2 unit tests need this to connect to services like the test CCDB instance.

At <https://ca.cern.ch/ca/>, request a new Grid host certificate for the machine.
Make sure that reminders about expiring certs go to the responsible group, not just you.
In order to do this, you must be listed as the "responsible" or "main user" of the machine in LanDB.

Create the certificate request using the system OpenSSL. On your local machine, set up the environment:

```bash
TARGET_MACHINE=alibuildmacXX.cern.ch
REMOTE_WORK_DIR=/Users/alibuild/renew-certificate
```

Open the page where you can paste the certificate request: 

```open
open "https://ca.cern.ch/ca/host/Submit.aspx?template=ee2host&instructions=openssl&subject=$TARGET_MACHINE"
```

Get in your clipboard the certificate request:

```bash
rm -fr $HOME/Downloads/host.cert
ssh $TARGET_MACHINE mkdir -p $REMOTE_WORK_DIR
ssh $TARGET_MACHINE cp /System/Library/OpenSSL/openssl.cnf $REMOTE_WORK_DIR/openssl.cnf
cat <<EOF | ssh $TARGET_MACHINE "cat >>$REMOTE_WORK_DIR/openssl.cnf"
[req]
req_extensions = v3_req

[ v3_req ]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = $TARGET_MACHINE
EOF
ssh $TARGET_MACHINE openssl req -new -subj "/CN=$TARGET_MACHINE" -out $REMOTE_WORK_DIR/newcsr.csr -keyout $REMOTE_WORK_DIR/privkey.pem -nodes -sha512 -newkey rsa:2048 -config $REMOTE_WORK_DIR/openssl.cnf
ssh $TARGET_MACHINE cat $REMOTE_WORK_DIR/newcsr.csr | pbcopy
```

Then paste it to generate the certificates. Download the base 64 certificate and copy it to the target machine with:

```bash
NEW_CERT=$HOME/Downloads/host.cert
scp $NEW_CERT $TARGET_MACHINE:$REMOTE_WORK_DIR/host.cert
# FIXME: linux users feel free to provide a PR using "pass"
security find-generic-password -a 'alibuild' -s "$TARGET_MACHINE" -w | ssh $TARGET_MACHINE sudo -S install -m 0600 -o alibuild -g staff $REMOTE_WORK_DIR/host.cert /etc/grid-security/hostcert.pem
security find-generic-password -a 'alibuild' -s "$TARGET_MACHINE" -w | ssh $TARGET_MACHINE sudo -S install -m 0600 -o alibuild -g staff $REMOTE_WORK_DIR/privkey.pem /etc/grid-security/hostkey.pem
```

Test that everything works correctly with (might need some adjustments to the path). On the `$TARGET_MACHINE`:

```bash
ssh $TARGET_MACHINE
# Adapt to find a valid xjalienfs
export WORK_DIR=/Volumes/build/alice-ci-workdir/o2/sw
source "$WORK_DIR/osx*/xjalienfs/latest/etc/profile.d/init.sh"
X509_USER_CERT=/etc/grid-security/hostcert.pem X509_USER_KEY=/etc/grid-security/hostkey.pem alien-token-init
```

Nomad does not apparently handle the change of certificate gracefully as of 1.6.2, so you need to restart it:

```
security find-generic-password -a 'alibuild' -s "$TARGET_MACHINE" -w | ssh $TARGET_MACHINE sudo -S brew service stop nomad
security find-generic-password -a 'alibuild' -s "$TARGET_MACHINE" -w | ssh $TARGET_MACHINE sudo -S sudo brew services start nomad --file=$HOME/homebrew.mxcl.nomad.plist
```

Finally clean-up the temporary area with:

```bash
# We keep it explicit to avoid having REMOTE_WORK_DIR empty or pointing to some wrong place.
# Check this is actually what you want to do!
ssh $TARGET_MACHINE rm -rf /Users/alibuild/renew-certificate
rm -fr $HOME/Downloads/host.cert
```

### Ask for AliEn access

Ask Costin or Max to add the DN of the certificate you just installed to the list of allowed certs for the `alienci` user.
You can get the certificate's DN using:
```bash
openssl x509 -in /etc/grid-security/hostcert.pem -noout -text | grep Subject:
```

### Set up Nomad and Consul

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
scp -3 alibuildmac00.cern.ch:{homebrew.mxcl.{nomad,consul}.plist,restart-services.sh} "$newhost:."
```

Start the services by running the following command on the Mac:
```bash
sudo ~/restart-services.sh
```

Finally, make sure that Nomad's work directory is readable by all users.
This is necessary to run CI jobs as the `alibuild` user, since it cannot find the script otherwise:
```bash
sudo chmod go+rx /opt/build/nomad
```

The host should appear in [the list of Nomad clients](https://alinomad.cern.ch/ui/clients).
If it does not, check the log, e.g. using `less -RSM "$(brew --prefix)/var/log/nomad.log"`.
If Nomad complains about not being able to connect to the master nodes at `alimesosNN.cern.ch`, update their firewall rules using:
```bash
pdsh -w 'alimesos[01-03].cern.ch' puppet agent -tv
```

## Adding a CI checker

### Adding a Nomad job

The macOS checkers are declared by a single Nomad job, `ci-jobs/ci-macos-arm.nomad`, which runs one
builder per eligible Mac. Unlike the Linux checkers, which can be placed anywhere in the cluster,
this job names the machines it may run on explicitly, so that we can tightly control what runs
where: many Macs have much less disk space than the Linux machines.

Two things in the jobspec therefore have to be updated when a Mac joins or leaves the fleet:

- the `constraint` on `${node.unique.name}`, which lists the hosts the job may be placed on. Being
  a macOS machine with compilation enabled is not sufficient, since other jobs own the remaining
  Macs;
- the task group's `count`, which must match the number of *eligible* hosts, as a
  `distinct_hosts` constraint allows only one builder per machine. A `count` higher than the number
  of eligible hosts leaves the deployment unable to reach its desired count, and it then never
  starts the rolling replacement either; make sure the new node has been marked eligible before
  raising it.

Once that's done, deploy as for any other plain jobspec:

```bash
cd ci-jobs
nomad job plan ci-macos-arm.nomad
# If the above succeeds, then:
nomad job run ci-macos-arm.nomad
```

### Configuring individual checks

Mac CI checkers are configured like their Linux equivalents, using `.env` files under `ali-bot/ci/repo-config/`.
The Macs specifically are listed under [`ali-bot/ci/repo-config/macos/`](https://github.com/alisw/ali-bot/tree/master/ci/repo-config/macos).

The directories there are named after the *platform*, not after a host: a builder serves the `.env`
files of every directory listed in the job's `CUR_CONTAINERS`, so all the Macs of a fleet run the
same set of checks and take their work from the same queue.

## Notable macOS post-mortes

* [Issues after a powercut](https://its.cern.ch/jira/browse/O2-4950)

## Troubleshooting

### Nomad service doesn't start after reboot / Job allocation stuck in pending

Some machines aren't able to remount the disks properly after a reboot. You can usually fix this issue with the following commands:


```
sudo mkdir /Volumes/build
diskutil list # find the right disk for the next command
sudo mount -t hfs /dev/disk2s1 /Volumes/build
sudo /Users/alibuild/restart-services.sh
```

### Macs continuously running out of memory

Make sure the Crash Reporter is disabled, as it can cause jobs like `o2-framework-crashing-workflow` to hang indefinitely.

### Nomad cannot stream logs (Path escapes the alloc directory)

Nomad has issues following symlinks in the data_dir, you can solve it by replacing it for the real path. 

e.g in `nomad.hcl`:

```diff
- data_dir = "/opt/build/nomad"
+ data_dir = "/Volumes/build/nomad"
```
