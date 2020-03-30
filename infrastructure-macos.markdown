The macOS build infrastructure works the same way as it's Linux counterpart. However, instead of being deployed via Apache Aurora, the scripts are run directly as scripts on macOS.


## Initial Setup 
These are the instructions for macOS 10.15 (Mojave)

* during setup create a user account for the `aliBuild` user, do not sound in to iCloud.
* Sign into the AppStore as `ali.bot@cern.ch`.

#### Software Prerequisites
* Install XCode 
* Install the command line tools 
```bash
sudo xcode-select --install
```
* approve the license conditions by
```bash
sudo xcodebuild -license
```
* Install Homebrew using the [instructions on their webpage](https://brew.sh/).
* Check if everything is OK via 
```bash
brew doctor
```
* Install the software prerequisites: 
```bash
brew install alisw/system-deps/o2-full-deps 
```
* For `alidist` PR builders install additional steps are required:
```bash
brew install mysql openjdk
sudo ln -sfn /usr/local/opt/openjdk/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk.jdk
```
Then append the `openjdk` to `PATH` via the `.zshrc`
```bash
export PATH="/usr/local/opt/openjdk/bin:$PATH"
```
* don't forget to reload your session


#### Python 
macOS developer tools (for now) come with both `python2.7` and `python3`. We will use `python3` from `brew` to not mess with Apples defaults. Both `python3` and `pip` are already installed as parts of the prerequisites and we need to create entries for `pip` and `python` in `/usr/local/bin`.

* Reinstalling pip will create the right symbolic links in `/usr/local/bin`:
```bash
pip3 install --upgrade --force-reinstall pip
```
* Manually create a symbolic link from `python3`to `python`:
```bash
ln -s  /usr/local/bin/python3 /usr/local/bin/python
```

#### Disable System Integrity Protection (SIP)
* Reboot the machine into recovery mode by holding `Command-R` at startup until the Apple logo appears
* In recovery mode run ```csrutil disable``` in a terminal and reboot - it can be re-enabled in the recovery mode via ```csrutil enable```
* Check the


#### Create work environment
With macOS 10.15 `/`is no longer writable. Create a _synthetic link_ by adding  
```bash
/build   System/Volumes/Data/build
```
to `/etc/synthetic.conf` (tab separated, not by whitespaces) and reboot.
Afterwards exclude the `/build` directory from Spotlight in the system preferences. Go to `(Apple) menu>System preferences>Spotlight`. In the `Privacy` tab, hit the `+` button. Now select the `/build` directory and confirm.

#### Prepare Builder Scripts
* In `/build` clone the `ali-Bot` repository which contains the CI scripts. 
```bash
git clone https://github.com/alisw/ali-bot.git
```
* Obtain the necessary credentials and install them in the `/Users/alibuild/.continuous-builder` file. Then restrict access: 
```bash
chmod 600  /Users/alibuild/.continuous-builder
```

## Running the PR Checker

There is a dedicated process to run validation of pullrequests for macos.The routines therein are implemented within a [runscript](https://github.com/alisw/ali-bot/blob/master/ci/run-continuous-builder.sh) inside [ali-bot](https://github.com/alisw/ali-bot). This script needs to be started manually using a `screen` session to ensure it stays open.

The builder script is launched via
```bash
screen -dmSL <session_name> /build/ali-bot/ci/run-continuous-builder.sh <config_file_argument>
```
inside the `/build` directory where will produce a logfile `/build/screenlog.0`


There are currently three dedicated worker nodes:
* `alibuildmac00`: process pull requests for `ALIROOT` and is launched by providing config file [`aliroot-guntest`](https://github.com/alisw/ali-bot/blob/master/ci/conf/aliroot-guntest.sh)
* `alibuildmac01`: process pull requests for O2 and is launched by providing argument [`o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/o2-0.sh)
* `alibuildmac03`: process pull requests for alidist and are launched by providing argument [`alidist_o2-0`](https://github.com/alisw/ali-bot/blob/master/ci/conf/alidist_o2-0.sh)
* the actual builds are taking place in `/build/<config_file_argument>` 
